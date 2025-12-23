# Backup Setup with Restic and Rest-Server on Synology NAS

This guide covers setting up automated backups using [Restic](https://restic.net/) with a self-hosted [rest-server](https://github.com/restic/rest-server) running in Docker on a Synology NAS (DSM 7.1+, tested on DS218+).

## Overview

**Why REST backend instead of CIFS/SMB?**

Using rest-server over REST API is preferred over mounting a CIFS/SMB share because:

- **Repository corruption risk:** Restic has [known unresolved issues](https://github.com/restic/restic/issues/2659) with CIFS/SMB mounts that can cause repository corruption. This is related to Go's asynchronous preemption and how SMB handles certain filesystem operations. Users have [reported 100% corruption rates](https://forum.restic.net/t/using-a-samba-protocol-storage-repository-on-windows-will-inevitably-cause-corruption/10406) in some scenarios.
- **Append-only mode:** rest-server supports append-only mode for ransomware protection, which is not possible with SMB
- **No mount management:** No need to handle mount/unmount, fstab entries, or credential files on the client

## Architecture

```
Homelab Server (Ubuntu)                    Synology NAS (DSM 7.1)
┌─────────────────────────────────────┐    ┌─────────────────────────────────┐
│                                     │    │                                 │
│  Restic Client                      │    │  Docker                         │
│  ├── Service configs backup    ─────────►│  └── rest-server (:8000)        │
│  └── Immich library backup     ─────────►│      └── /data/restic-repos/    │
│                                     │    │          ├── homelab-configs/   │
│  Cron / Systemd Timer               │    │          └── homelab-immich/    │
│  └── Scheduled backups              │    │                                 │
│                                     │    │  Volume: /volume1/docker/restic │
└─────────────────────────────────────┘    └─────────────────────────────────┘
```

## Backup Strategy

### Repository 1: Service Configurations

- Traefik, arr stack, AdGuard, Cloudflare Tunnel configs
- Small/medium size, quick to backup
- **Excludes:** Media files (`arr/data/media/`, `arr/data/torrents/`)

### Repository 2: Immich Library

- Photos, videos, thumbnails, and database dumps
- Large size, incremental backups essential
- **Excludes:** PostgreSQL data directory (Immich dumps DB to library)

## Prerequisites

1. **Synology NAS** with DSM 7.1+ and Docker package installed
2. **Network connectivity** between homelab server and Synology NAS

## Part 1: Synology NAS Setup (rest-server)

### 1. Create Directory Structure

1. Open **File Station**
2. Navigate to `docker` folder (create it if it doesn't exist)
3. Create folder `restic`
4. Inside `restic`, create two subfolders: `repos` and `auth`

Final structure: `/volume1/docker/restic/repos` and `/volume1/docker/restic/auth`

### 2. Create Docker Compose File

1. Open **File Station** → navigate to `/volume1/docker/restic`
2. Right-click → **Create** → **Create file** → name it `compose.yaml`
3. Right-click `compose.yaml` → **Open with Text Editor**
4. Paste the following content and save:

```yaml
services:
  rest-server:
    image: restic/rest-server:latest
    container_name: restic-rest-server
    restart: unless-stopped
    ports:
      - "8000:8000"
    volumes:
      - ./repos:/data
      - ./auth/.htpasswd:/data/.htpasswd:ro
    environment:
      # Enable append-only mode for ransomware protection (optional)
      # With append-only, clients cannot delete or modify existing backups
      # - OPTIONS=--append-only
      # Enable Prometheus metrics (optional)
      # - OPTIONS=--prometheus
    security_opt:
      - no-new-privileges:true
```

### 3. Start rest-server

1. Open **Docker** package
2. Go to **Registry** → search for `restic` → double-click `restic/rest-server` → **Choose Tag**: `latest`
3. Go to **Image** → double-click `restic/rest-server:latest`
4. **Network**: Select `bridge`
5. **General Settings**:
   - Container Name: `restic-rest-server`
   - Enable **Enable auto-restart**
   - Port: `8000` (Local Port) → `8000` (Container Port)
6. Click **Advanced Settings** → **Environment** tab → **Add**:
   - `DATA_DIRECTORY`: `/data`
   - `PASSWORD_FILE`: `/data/.htpasswd`
7. Click **Save** → **Next**
8. **Volume Settings** → **Add Folder**:
   - Select `docker/restic/repos` → Mount path: `/data` (Read-Only: unchecked)
9. Click **Next** → **Done**

### 4. Configure Web Station Portal

Web Station is required to expose the Docker container's port externally.

**Enable Firewall Rule for Port 8000:**

1. Open **Control Panel** → **Security** → **Firewall**
2. Select your firewall profile (e.g., `default`) → click **Edit Rules**
3. Click **Create** to add a new rule
4. Configure the rule:
   - **Ports**: Select **Custom** → enter `8000`
   - **Source IP**: `All` (or restrict to your LAN subnet for security)
   - **Action**: `Allow`
5. Click **OK** → **Apply**

**Create Web Station Portal:**

1. Open **Web Station** package
2. Go to **Web Service Portal** → click **Create**
3. Select **Package Server Portal**
4. Select package: **Docker** → click **Next**
5. Configure the portal:
   - **Service**: Select `Docker restic-rest-server1 8000`
   - **Portal type**: `Port-based`
   - **Port**: Check `HTTP` → enter `8000`
6. Click **Create**

### 5. Create Authentication

1. In **Docker** → **Container** → select `restic-rest-server`
2. Click **Details** → **Terminal** tab
3. Click **Create** → select `sh` (not bash - the container uses Alpine)
4. Run:

   ```bash
   create_user homelab
   ```

5. Enter and confirm password when prompted
6. Store this password securely (e.g., in 1Password)

**Important:** Store the username and password securely - you'll need them for the homelab server configuration.

### 6. Verify rest-server is Running

1. In **Docker** → **Container**
2. Verify `restic-rest-server` shows status **Running**
3. Click on the container → **Details** → **Log** tab to check for any errors

## Part 2: Homelab Server Setup (Restic Client)

### 1. Install Restic

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install restic

# Verify installation
restic version
```

**Or install latest version directly:**

```bash
# Download latest release
RESTIC_VERSION=$(curl -s https://api.github.com/repos/restic/restic/releases/latest | grep tag_name | cut -d '"' -f 4 | tr -d 'v')
wget https://github.com/restic/restic/releases/download/v${RESTIC_VERSION}/restic_${RESTIC_VERSION}_linux_amd64.bz2

# Extract and install
bunzip2 restic_${RESTIC_VERSION}_linux_amd64.bz2
chmod +x restic_${RESTIC_VERSION}_linux_amd64
sudo mv restic_${RESTIC_VERSION}_linux_amd64 /usr/local/bin/restic

# Verify
restic version
```

### 2. Create Environment File

Create `/home/<user>/.config/restic/env` with your credentials:

```bash
mkdir -p ~/.config/restic

cat > ~/.config/restic/env << 'EOF'
# Restic repository password (encryption key)
# Generate with: openssl rand -base64 32
export RESTIC_PASSWORD="<your-repository-password>"

# Rest-server credentials
export RESTIC_REST_USERNAME="homelab"
export RESTIC_REST_PASSWORD="<your-rest-server-password>"

# Repository URLs
export RESTIC_REPOSITORY_CONFIGS="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@<synology-ip>:8000/homelab-configs"
export RESTIC_REPOSITORY_IMMICH="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@<synology-ip>:8000/homelab-immich"
EOF

# Secure the file
chmod 600 ~/.config/restic/env
```

**Critical:** Back up this file securely. The `RESTIC_PASSWORD` is required to decrypt your backups. Store it in a password manager like 1Password.

### 3. Initialize Repositories

```bash
# Load environment
source ~/.config/restic/env

# Initialize service configs repository
restic -r "$RESTIC_REPOSITORY_CONFIGS" init

# Initialize Immich repository
restic -r "$RESTIC_REPOSITORY_IMMICH" init
```

### 4. Create Backup Scripts

Create `/home/<user>/scripts/restic-backup-configs.sh`:

```bash
#!/bin/bash
set -euo pipefail

# Load environment
source ~/.config/restic/env

# Backup service configurations
restic -r "$RESTIC_REPOSITORY_CONFIGS" backup \
    --verbose \
    --exclude="arr/data/media" \
    --exclude="arr/data/torrents" \
    --exclude="arr/data/cache" \
    --exclude="immich" \
    ~/docker/traefik \
    ~/docker/arr \
    ~/docker/adguard \
    ~/docker/cloudflare-tunnel \
    ~/docker/dashdot

# Prune old snapshots (keep last 7 daily, 4 weekly, 12 monthly)
restic -r "$RESTIC_REPOSITORY_CONFIGS" forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --prune

# Check repository integrity (weekly recommended)
if [ "$(date +%u)" -eq 7 ]; then
    restic -r "$RESTIC_REPOSITORY_CONFIGS" check
fi

echo "Configs backup completed: $(date)"
```

Create `/home/<user>/scripts/restic-backup-immich.sh`:

```bash
#!/bin/bash
set -euo pipefail

# Load environment
source ~/.config/restic/env

# Backup Immich library (photos, videos, and DB dumps)
restic -r "$RESTIC_REPOSITORY_IMMICH" backup \
    --verbose \
    --exclude="postgres" \
    ~/docker/immich/library

# Prune old snapshots (keep more for photos)
restic -r "$RESTIC_REPOSITORY_IMMICH" forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --keep-yearly 5 \
    --prune

# Check repository integrity (weekly recommended)
if [ "$(date +%u)" -eq 7 ]; then
    restic -r "$RESTIC_REPOSITORY_IMMICH" check
fi

echo "Immich backup completed: $(date)"
```

Make scripts executable:

```bash
chmod +x ~/scripts/restic-backup-*.sh
```

### 5. Test Backups Manually

```bash
# Run config backup
~/scripts/restic-backup-configs.sh

# Run Immich backup
~/scripts/restic-backup-immich.sh

# List snapshots
source ~/.config/restic/env
restic -r "$RESTIC_REPOSITORY_CONFIGS" snapshots
restic -r "$RESTIC_REPOSITORY_IMMICH" snapshots
```

### 6. Schedule Automated Backups

**Option A: Cron (simple)**

```bash
crontab -e
```

Add:

```cron
# Service configs backup - daily at 2:00 AM
0 2 * * * /home/<user>/scripts/restic-backup-configs.sh >> /home/<user>/logs/restic-configs.log 2>&1

# Immich backup - daily at 3:00 AM
0 3 * * * /home/<user>/scripts/restic-backup-immich.sh >> /home/<user>/logs/restic-immich.log 2>&1
```

Create log directory:

```bash
mkdir -p ~/logs
```

**Option B: Systemd Timer (recommended)**

Create `/etc/systemd/system/restic-backup-configs.service`:

```ini
[Unit]
Description=Restic backup of service configurations
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=<user>
ExecStart=/home/<user>/scripts/restic-backup-configs.sh
StandardOutput=journal
StandardError=journal
```

Create `/etc/systemd/system/restic-backup-configs.timer`:

```ini
[Unit]
Description=Daily restic backup of service configurations

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

Enable the timer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now restic-backup-configs.timer

# Check timer status
systemctl list-timers restic-backup-configs.timer
```

Repeat for Immich backup with different time (e.g., 03:00).

## Restore Procedures

### List Available Snapshots

```bash
source ~/.config/restic/env

# List config snapshots
restic -r "$RESTIC_REPOSITORY_CONFIGS" snapshots

# List Immich snapshots
restic -r "$RESTIC_REPOSITORY_IMMICH" snapshots

# List files in a snapshot
restic -r "$RESTIC_REPOSITORY_CONFIGS" ls <snapshot-id>
```

### Restore Service Configurations

```bash
source ~/.config/restic/env

# Stop affected services
docker compose stop <service-name>

# Restore entire snapshot to a staging directory
restic -r "$RESTIC_REPOSITORY_CONFIGS" restore <snapshot-id> --target ~/restore-staging

# Or restore specific files/directories
restic -r "$RESTIC_REPOSITORY_CONFIGS" restore <snapshot-id> \
    --target ~/restore-staging \
    --include "/home/<user>/docker/traefik"

# Move restored files to production
sudo cp -r ~/restore-staging/home/<user>/docker/<service>/* ~/docker/<service>/

# Start services
docker compose start <service-name>

# Cleanup
rm -rf ~/restore-staging
```

### Restore Immich

1. **Stop Immich services:**

   ```bash
   docker compose down immich-server database immich-machine-learning
   ```

2. **Restore library from Restic:**

   ```bash
   source ~/.config/restic/env

   # Restore to staging directory
   restic -r "$RESTIC_REPOSITORY_IMMICH" restore latest --target ~/restore-staging

   # Move restored files
   sudo mv ~/restore-staging/home/<user>/docker/immich/library/* ~/docker/immich/library/
   ```

3. **Restore database from SQL dump:**

   ```bash
   # Start only PostgreSQL
   docker compose up -d database

   # Wait for Postgres to be ready, then restore
   gunzip --stdout ~/docker/immich/library/backups/<latest-backup>.sql.gz \
   | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
   | docker exec -i immich_postgres psql --dbname=immich --username=postgres
   ```

4. **Start all Immich containers:**

   ```bash
   docker compose up -d
   ```

5. **Cleanup:**

   ```bash
   rm -rf ~/restore-staging
   ```

### Restore Single File

```bash
source ~/.config/restic/env

# Find the file in snapshots
restic -r "$RESTIC_REPOSITORY_CONFIGS" find "*AdGuardHome.yaml*"

# Restore specific file
restic -r "$RESTIC_REPOSITORY_CONFIGS" restore <snapshot-id> \
    --target ~/restore-staging \
    --include "**/AdGuardHome.yaml"
```

## Maintenance

### Check Repository Health

```bash
source ~/.config/restic/env

# Quick check (verify structure)
restic -r "$RESTIC_REPOSITORY_CONFIGS" check

# Full check (verify all data, slower)
restic -r "$RESTIC_REPOSITORY_CONFIGS" check --read-data
```

### View Repository Statistics

```bash
source ~/.config/restic/env

# Repository stats
restic -r "$RESTIC_REPOSITORY_CONFIGS" stats

# Detailed stats by snapshot
restic -r "$RESTIC_REPOSITORY_CONFIGS" stats --mode raw-data
```

### Manual Snapshot Pruning

```bash
source ~/.config/restic/env

# Dry-run to see what would be deleted
restic -r "$RESTIC_REPOSITORY_CONFIGS" forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --dry-run

# Execute pruning
restic -r "$RESTIC_REPOSITORY_CONFIGS" forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --prune
```

### Update Restic

```bash
# If installed via apt
sudo apt update && sudo apt upgrade restic

# If installed manually, re-run installation steps
```

## Troubleshooting

### Connection Refused

If you can't connect to rest-server:

1. Verify rest-server is running on Synology:

   ```bash
   docker ps | grep restic
   ```

2. Check Synology firewall allows port 8000

3. Test from Synology locally:

   ```bash
   curl http://localhost:8000/
   ```

### Authentication Failed

1. Verify credentials in `~/.config/restic/env`

2. Test with curl:

   ```bash
   curl -u homelab http://<synology-ip>:8000/
   ```

3. Recreate user if needed:

   ```bash
   # On Synology
   docker exec -it restic-rest-server delete_user homelab
   docker exec -it restic-rest-server create_user homelab
   ```

### Repository Locked

If a backup was interrupted:

```bash
source ~/.config/restic/env

# List locks
restic -r "$RESTIC_REPOSITORY_CONFIGS" list locks

# Remove stale locks (use with caution)
restic -r "$RESTIC_REPOSITORY_CONFIGS" unlock
```

### Slow Backups

1. **Check network speed** between homelab and NAS

2. **Increase parallelism** (if NAS has resources):

   ```bash
   restic -r "$RESTIC_REPOSITORY_CONFIGS" backup \
       --verbose \
       -o rest.connections=10 \
       ~/docker/traefik
   ```

3. **Exclude unnecessary files** in backup scripts

### Backup Verification

Periodically verify backups can be restored:

```bash
source ~/.config/restic/env

# Mount repository and browse (requires FUSE)
mkdir -p /tmp/restic-mount
restic -r "$RESTIC_REPOSITORY_CONFIGS" mount /tmp/restic-mount

# Browse snapshots in another terminal
ls /tmp/restic-mount/snapshots/

# Unmount when done
fusermount -u /tmp/restic-mount
```

## Security Considerations

1. **Encryption:** All data is encrypted with your `RESTIC_PASSWORD` before leaving the homelab server

2. **Authentication:** rest-server requires HTTP Basic Auth

3. **Append-only mode:** Enable `--append-only` in rest-server for ransomware protection (clients cannot delete backups)

4. **Credential storage:** Keep `~/.config/restic/env` secure (600 permissions) and backed up separately

5. **Network:** Consider using HTTPS with a reverse proxy for encrypted transport (optional if on trusted LAN)
