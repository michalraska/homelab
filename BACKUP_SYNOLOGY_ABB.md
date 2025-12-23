# Backup Setup with Synology Active Backup for Business

This guide covers setting up automated backups of Docker service configurations and Immich photo library using Synology Active Backup for Business (ABB) with rsync over SSH.

## Backup Strategy

### Task 1: Service Configurations

- Traefik, arr stack, AdGuard, Cloudflare Tunnel
- Small/medium size, quick to backup
- **Excludes:** Media files (`arr/data/`) - too large, typically sourced from torrents

### Task 2: Immich Library

- Photos, videos, thumbnails, and automatic database dumps
- Large size, may take longer to backup
- **Excludes:** Immich PostgreSQL directory - Immich dumps database to library automatically

## Prerequisites

1. **Synology NAS** with Active Backup for Business installed
2. **Network connectivity** between Synology NAS and homelab server

## Homelab Server Configuration

### 1. Create Backup User

Create a dedicated user for backups:

```bash
# Create system user for backups
# Note: /bin/bash is required for rsync over SSH to work
sudo useradd -r -s /bin/bash -m homelab-backup

# Create .ssh directory for the backup user
sudo mkdir -p /home/homelab-backup/.ssh
sudo chmod 700 /home/homelab-backup/.ssh
sudo chown homelab-backup:homelab-backup /home/homelab-backup/.ssh

# Create restore staging directory (for ABB restore operations)
sudo mkdir -p /home/homelab-backup/restoring
sudo chown homelab-backup:homelab-backup /home/homelab-backup/restoring
```

### 2. Configure SSH Keep-Alive for Long Transfers

Large backup/restore operations can take a long time. Default SSH settings may timeout and disconnect during transfers. Configure extended keep-alive for the backup user:

```bash
# Copy the SSH config file
sudo cp sshd/99-backup-user.conf /etc/ssh/sshd_config.d/

# Restart SSH to apply changes
sudo systemctl restart ssh

# Verify the configuration is loaded
sudo sshd -T | grep -i clientalive
```

This allows the backup user up to 60 minutes without response before disconnecting, while keeping stricter timeouts for other users.

### 3. SSH Key Authentication (Ubuntu 24.04)

#### Generate SSH Key Pair

Generate an SSH key pair using one of these methods:

**Option A: Generate in 1Password**
1. In 1Password, create a new **SSH Key** item
2. Select **Ed25519** as the key type
3. Copy the public key for the homelab server
4. Export the private key for Synology ABB

**Option B: Generate with ssh-keygen**
```bash
ssh-keygen -t ed25519 -C "synology-abb-backup" -f homelab-backup-key
```

Store the private key securely (e.g., in 1Password).

#### On Ubuntu 24.04 Homelab Server

**Add the public key to authorized_keys:**

```bash
# Create authorized_keys file
sudo touch /home/homelab-backup/.ssh/authorized_keys
sudo chmod 600 /home/homelab-backup/.ssh/authorized_keys
sudo chown homelab-backup:homelab-backup /home/homelab-backup/.ssh/authorized_keys

# Add the public key
# Replace <PUBLIC_KEY_CONTENT> with the public key
echo "<PUBLIC_KEY_CONTENT>" | sudo tee -a /home/homelab-backup/.ssh/authorized_keys

# Verify the key was added
sudo cat /home/homelab-backup/.ssh/authorized_keys
```

#### On Synology NAS (via DSM WebUI)

1. Open **Active Backup for Business** in DSM
2. Go to **File Server** → **Add Server**
3. Select **rsync** as server type
4. Enter connection details:
   - Server address: <your homelab hostname or IP>
   - Connection Mode: rsync shell mode via SSH
   - Port: 22
   - Account: `homelab-backup`
   - Auth Policy: **By SSH key**
5. **Upload the private key** you generated earlier
6. Click **OK** to verify SSH connectivity
7. If successful, proceed to configure backup directories

## Synology Active Backup for Business Configuration

### 1. Create Backup Task: Immich Library

1. Create a backup task for Immich
2. Add the following path (replace `<user>` with your username):

| Path | Description |
|------|-------------|
| `/home/<user>/docker/immich/library/` | Immich photos, videos, and DB dumps |

### 2. Create Backup Task: Service Configurations

1. Go to **File Server** → **Add Server** (use the server configured in SSH section)
2. Create a new backup task for service configurations
3. Add the following paths (replace `<user>` with your username, e.g., `michal`):

| Path | Description |
|------|-------------|
| `/home/<user>/docker/traefik/` | Traefik config and Let's Encrypt certs |
| `/home/<user>/docker/arr/` | Arr stack configs (excludes `data/` - see below) |
| `/home/<user>/docker/adguard/` | AdGuard Home configuration |
| `/home/<user>/docker/cloudflare-tunnel/` | Cloudflare Tunnel credentials |

**Configure exclusions** for the arr task to skip media files:

- Exclude: `data/media/` (movies and TV shows - too large, sourced from torrents)
- Exclude: `data/torrents/` (torrent downloads)

#### Configure ACL Permissions for AdGuard

Most Docker services (Immich, Jellyfin, arr stack) create files with world-readable permissions (755/644), so no special configuration is needed for backup access.

**AdGuard Home is the exception** - it creates files with restrictive permissions for security reasons:

- `AdGuardHome.yaml` (600) - contains user credentials
- `work/data/` (700) - contains query logs

Grant read access to the backup user for AdGuard only:

```bash
# Install ACL tools if not present
sudo apt install acl

# Grant read and traverse permissions to backup user for AdGuard directory only
# Note: X (capital) grants execute only on directories (for traversal), not on regular files
sudo setfacl -R -m u:homelab-backup:rX $HOME/docker/adguard

# Set default ACL for new files (inheritable permissions)
sudo setfacl -R -d -m u:homelab-backup:rX $HOME/docker/adguard
```

**Verify ACL permissions:**

```bash
getfacl ~/docker/adguard
getfacl ~/docker/adguard/conf/AdGuardHome.yaml
```

## Restore Procedures (via ABB)

Since most Docker service directories are owned by root, ABB cannot restore directly to original locations. Instead, restore to a staging directory and then move files manually.

### Restore via ABB Interface

1. Open **Active Backup for Business**
2. Go to **File Server** → Select your backup task
3. Click **Restore**
4. Browse to the backup version you want to restore
5. Select files/directories to restore
6. Choose restore destination: `/home/homelab-backup/restoring/`

### Move Restored Files to Production

After ABB completes the restore to the staging directory (ABB preserves the full path structure):

```bash
# Stop the affected service first
docker compose stop <service-name>

# Move restored files to production location
sudo mv /home/homelab-backup/restoring/home/<user>/docker/<service>/* ~/docker/<service>/

# Start the service
docker compose start <service-name>
```

### Restore Immich (from ABB)

1. Use ABB to restore `/home/<user>/docker/immich/library/` to `/home/homelab-backup/restoring/`

2. Stop all Immich services:

   ```bash
   docker compose -f ~/docker/immich/compose.yaml down
   ```

3. Move restored library files to production (ABB preserves the full path structure):

   ```bash
   sudo mv /home/homelab-backup/restoring/home/<user>/docker/immich/library/* ~/docker/immich/library/
   ```

4. Restore the database from the SQL dump:

   ```bash
   # Start only PostgreSQL
   docker compose -f ~/docker/immich/compose.yaml up -d database

   # Wait for Postgres to be ready, then restore the database dump
   gunzip --stdout ~/docker/immich/library/backups/<latest-backup>.sql.gz \
   | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
   | docker exec -i immich_postgres psql --dbname=immich --username=postgres
   ```

5. Start all Immich containers:

   ```bash
   docker compose -f ~/docker/immich/compose.yaml up -d
   ```

**Note:** If you encounter database migration conflicts, set `DB_SKIP_MIGRATIONS=true` in `immich/.env` before starting services, then remove it and restart after the restore completes.

### Restore AdGuard Home

1. Use ABB to restore `/home/<user>/docker/adguard/` to `/home/homelab-backup/restoring/`

2. Stop AdGuard Home:

   ```bash
   docker compose -f ~/docker/adguard/compose.yaml stop
   ```

3. Move restored files (ABB preserves the full path structure):

   ```bash
   sudo mv /home/homelab-backup/restoring/home/<user>/docker/adguard/* ~/docker/adguard/
   ```

4. Start AdGuard Home:

   ```bash
   docker compose -f ~/docker/adguard/compose.yaml start
   ```

5. Verify DNS resolution is working and check the AdGuard Home web UI

### Restore Other Services

1. Use ABB to restore `/home/<user>/docker/<service>/` to `/home/homelab-backup/restoring/`

2. Stop the service:

   ```bash
   docker compose stop <service-name>
   ```

3. Move restored files (ABB preserves the full path structure):

   ```bash
   sudo mv /home/homelab-backup/restoring/home/<user>/docker/<service>/* ~/docker/<service>/
   ```

4. Start the service:

   ```bash
   docker compose start <service-name>
   ```

## Troubleshooting

### Timeout Errors / Operation Timed Out

If ABB reports "The operation timed out" or backups/restores are interrupted:

1. **Verify SSH keep-alive configuration is applied:**

   ```bash
   sudo sshd -T -C user=homelab-backup | grep -i clientalive
   ```

   Expected output:

   ```text
   clientaliveinterval 60
   clientalivecountmax 60
   ```

2. **Check SSH logs for timeout events:**

   ```bash
   sudo journalctl -u ssh --since "1 hour ago" | grep -i "timeout\|disconnect"
   ```

3. **Re-apply the SSH configuration if needed:**

   ```bash
   sudo cp sshd/99-backup-user.conf /etc/ssh/sshd_config.d/
   sudo systemctl restart ssh
   ```

### Permission Denied Errors

If ABB reports permission errors for **AdGuard**:

```bash
# Verify ACL permissions
getfacl ~/docker/adguard

# Re-apply ACL if needed
sudo setfacl -R -m u:homelab-backup:rX $HOME/docker/adguard
sudo setfacl -R -d -m u:homelab-backup:rX $HOME/docker/adguard
```

For other services (Immich, Jellyfin, arr stack), permission errors are unlikely since they use world-readable permissions (755/644). If you encounter issues, verify the file permissions:

```bash
ls -la ~/docker/<service>/
```
