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

- AdGuard config, arr stack configs and app backups
- Small size (~few MB), quick to backup
- **Includes only:** Config files (`.xml`, `.yaml`, `.conf`) and app-created Backups folders
- Regenerable data (SSL certs, metadata, databases) excluded - apps restore from their Backups

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

The Restic client is integrated into the main homelab Docker Compose setup using [Resticker](https://github.com/djmaze/resticker) (`mazzolino/restic`), which includes built-in scheduling via go-cron.

**Configuration:** See [restic/compose.yaml](restic/compose.yaml) for the Docker Compose configuration.

### 1. Configure Environment Variables

Add the following to your `.env` file (see [.env.example](.env.example)):

| Variable               | Description                        |
|------------------------|------------------------------------|
| `RESTIC_REST_HOST`     | REST server IP/hostname            |
| `RESTIC_REST_PORT`     | REST server port (default: `8000`) |
| `RESTIC_PASSWORD`      | Repository encryption password     |
| `RESTIC_REST_USERNAME` | REST server username               |
| `RESTIC_REST_PASSWORD` | REST server password               |

**Critical:** Back up your `RESTIC_PASSWORD` securely. It is required to decrypt your backups. Store it in a password manager like 1Password.

### 2. Start Backup Containers

Repositories are automatically initialized on first run if they don't exist.

```bash
cd ~/docker
docker compose up -d restic-configs restic-immich
```

### 3. Test Backups Manually

Trigger a manual backup:

```bash
# Trigger configs backup
docker exec restic-configs /usr/local/bin/backup

# Trigger Immich backup
docker exec restic-immich /usr/local/bin/backup
```

### 4. View Snapshots

```bash
# List config snapshots
docker exec restic-configs restic snapshots

# List Immich snapshots
docker exec restic-immich restic snapshots
```

### 5. View Backup Logs

```bash
# View configs backup logs
docker logs restic-configs

# View Immich backup logs
docker logs restic-immich

# Follow logs in real-time
docker logs -f restic-configs
```

## Restore Procedures

### List Available Snapshots

```bash
# List config snapshots
docker exec restic-configs restic snapshots

# List Immich snapshots
docker exec restic-immich restic snapshots

# List files in a snapshot
docker exec restic-configs restic ls <snapshot-id>
```

### Restore Service Configurations

```bash
cd ~/docker

# Stop affected services
docker compose stop <service>

# Restore entire snapshot to a staging directory
mkdir -p ~/restore-staging
docker run --rm --env-file .env \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_HOST}:${RESTIC_REST_PORT}/homelab-configs" \
  -v ~/restore-staging:/restore \
  restic/restic restore <snapshot-id> --target /restore

# Or restore specific files/directories
docker run --rm --env-file .env \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_HOST}:${RESTIC_REST_PORT}/homelab-configs" \
  -v ~/restore-staging:/restore \
  restic/restic restore <snapshot-id> \
    --target /restore \
    --include "/data/traefik"

# Move restored files to production
sudo cp -r ~/restore-staging/data/<service>/* ~/docker/<service>/

# Start services
docker compose start <service>

# Cleanup
rm -rf ~/restore-staging
```

### Restore Immich

1. **Stop Immich services:**

   ```bash
   cd ~/docker
   docker compose down immich-server immich-machine-learning database
   ```

2. **Restore library from Restic:**

   ```bash
   cd ~/docker

   # Restore to staging directory
   mkdir -p ~/restore-staging
   docker run --rm --env-file .env \
     -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_HOST}:${RESTIC_REST_PORT}/homelab-immich" \
     -v ~/restore-staging:/restore \
     restic/restic restore latest --target /restore

   # Move restored files
   sudo mv ~/restore-staging/data/library/* ~/docker/immich/library/
   ```

3. **Restore database from SQL dump:**

   ```bash
   cd ~/docker

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
cd ~/docker

# Find the file in snapshots
docker exec restic-configs restic find "*AdGuardHome.yaml*"

# Restore specific file
mkdir -p ~/restore-staging
docker run --rm --env-file .env \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_HOST}:${RESTIC_REST_PORT}/homelab-configs" \
  -v ~/restore-staging:/restore \
  restic/restic restore <snapshot-id> \
    --target /restore \
    --include "**/AdGuardHome.yaml"
```

## Maintenance

### Check Repository Health

```bash
# Quick check (verify structure)
docker exec restic-configs restic check

# Full check (verify all data, slower)
docker exec restic-configs restic check --read-data
```

### View Repository Statistics

```bash
# Repository stats
docker exec restic-configs restic stats

# Detailed stats by snapshot
docker exec restic-configs restic stats --mode raw-data
```

### Manual Snapshot Pruning

Pruning is done automatically after each backup. To run manually:

```bash
# Dry-run to see what would be deleted
docker exec restic-configs restic forget \
  --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --dry-run

# Execute pruning
docker exec restic-configs restic forget \
  --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune
```

### Update Restic

```bash
cd ~/docker

# Pull latest images
docker pull mazzolino/restic:latest
docker pull restic/restic:latest

# Restart backup containers to use new image
docker compose up -d restic-configs restic-immich
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

4. Check backup container logs:

   ```bash
   docker logs restic-configs
   ```

### Authentication Failed

1. Verify credentials in `~/docker/.env`

2. Test with curl:

   ```bash
   curl -u homelab http://<rest-server-ip>:8000/
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
# List locks
docker exec restic-configs restic list locks

# Remove stale locks (use with caution)
docker exec restic-configs restic unlock
```

### Slow Backups

1. **Check network speed** between homelab and NAS

2. **Increase parallelism** by adding to `RESTIC_BACKUP_ARGS` in `compose.yaml`:

   ```yaml
   RESTIC_BACKUP_ARGS: >-
     --verbose
     -o rest.connections=10
   ```

3. **Exclude unnecessary files** in `RESTIC_BACKUP_ARGS`

### Backup Verification

Periodically verify backups can be restored:

```bash
cd ~/docker

# Mount repository and browse (requires FUSE on host)
mkdir -p /tmp/restic-mount
docker run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --env-file .env \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_HOST}:${RESTIC_REST_PORT}/homelab-configs" \
  -v /tmp/restic-mount:/mnt/restic:shared \
  restic/restic mount /mnt/restic

# Browse snapshots in another terminal
ls /tmp/restic-mount/snapshots/

# Unmount when done
fusermount -u /tmp/restic-mount
```

**Alternative:** Use `docker exec restic-configs restic ls latest` or restore to a staging directory instead of mounting.

## Security Considerations

1. **Encryption:** All data is encrypted with your `RESTIC_PASSWORD` before leaving the homelab server

2. **Authentication:** rest-server requires HTTP Basic Auth

3. **Append-only mode:** Enable `--append-only` in rest-server for ransomware protection (clients cannot delete backups)

4. **Credential storage:** Keep `~/docker/.env` secure (600 permissions) and backed up separately

5. **Read-only mounts:** Backup containers mount source directories as read-only (`:ro`) to prevent accidental modifications

6. **Network:** Consider using HTTPS with a reverse proxy for encrypted transport (optional if on trusted LAN)
