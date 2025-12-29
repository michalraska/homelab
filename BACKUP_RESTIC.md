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

This setup uses Docker Compose with the [Resticker](https://github.com/djmaze/resticker) image (`mazzolino/restic`), which includes built-in scheduling via go-cron.

### 1. Create Directory Structure

```bash
mkdir -p ~/docker/restic
cd ~/docker/restic
```

### 2. Create Environment File

Create `~/docker/restic/.env` with your credentials:

```bash
cat > ~/docker/restic/.env << 'EOF'
# Restic repository password (encryption key)
# Generate with: openssl rand -base64 32
RESTIC_PASSWORD=<your-repository-password>

# Rest-server credentials
RESTIC_REST_USERNAME=homelab
RESTIC_REST_PASSWORD=<your-rest-server-password>

# Synology NAS IP
SYNOLOGY_IP=<synology-ip>
EOF

# Secure the file
chmod 600 ~/docker/restic/.env
```

**Critical:** Back up this file securely. The `RESTIC_PASSWORD` is required to decrypt your backups. Store it in a password manager like 1Password.

### 3. Create Docker Compose File

Create `~/docker/restic/compose.yaml`:

```yaml
services:
  # Backup service configurations
  restic-configs:
    image: mazzolino/restic:latest
    container_name: restic-configs
    restart: unless-stopped
    hostname: homelab
    environment:
      # Using separate env vars for REST credentials (avoids credentials in logs)
      RESTIC_REPOSITORY: rest:http://${SYNOLOGY_IP}:8000/homelab-configs
      RESTIC_PASSWORD: ${RESTIC_PASSWORD}
      RESTIC_REST_USERNAME: ${RESTIC_REST_USERNAME}
      RESTIC_REST_PASSWORD: ${RESTIC_REST_PASSWORD}
      # Backup daily at 2:00 AM (go-cron format: seconds minutes hours day month weekday)
      BACKUP_CRON: "0 0 2 * * *"
      RESTIC_BACKUP_SOURCES: /data/traefik /data/arr /data/adguard /data/cloudflare-tunnel /data/dashdot
      RESTIC_BACKUP_ARGS: >-
        --exclude="arr/data/media"
        --exclude="arr/data/torrents"
        --exclude="arr/data/cache"
        --verbose
      # Prune weekly on Sundays at 2:30 AM
      PRUNE_CRON: "0 30 2 * * 0"
      RESTIC_FORGET_ARGS: >-
        --group-by host,paths
        --keep-daily 7
        --keep-weekly 4
        --keep-monthly 12
      # Check repository monthly on 1st at 4:00 AM
      CHECK_CRON: "0 0 4 1 * *"
      TZ: Europe/Prague
    volumes:
      - ~/docker/traefik:/data/traefik:ro
      - ~/docker/arr:/data/arr:ro
      - ~/docker/adguard:/data/adguard:ro
      - ~/docker/cloudflare-tunnel:/data/cloudflare-tunnel:ro
      - ~/docker/dashdot:/data/dashdot:ro
    security_opt:
      - no-new-privileges:true

  # Backup Immich library
  restic-immich:
    image: mazzolino/restic:latest
    container_name: restic-immich
    restart: unless-stopped
    hostname: homelab
    environment:
      # Using separate env vars for REST credentials (avoids credentials in logs)
      RESTIC_REPOSITORY: rest:http://${SYNOLOGY_IP}:8000/homelab-immich
      RESTIC_PASSWORD: ${RESTIC_PASSWORD}
      RESTIC_REST_USERNAME: ${RESTIC_REST_USERNAME}
      RESTIC_REST_PASSWORD: ${RESTIC_REST_PASSWORD}
      # Backup daily at 3:00 AM
      BACKUP_CRON: "0 0 3 * * *"
      RESTIC_BACKUP_SOURCES: /data/library
      RESTIC_BACKUP_ARGS: >-
        --exclude="postgres"
        --verbose
      # Prune weekly on Sundays at 3:30 AM
      PRUNE_CRON: "0 30 3 * * 0"
      RESTIC_FORGET_ARGS: >-
        --group-by host,paths
        --keep-daily 7
        --keep-weekly 4
        --keep-monthly 12
        --keep-yearly 5
      # Check repository monthly on 1st at 5:00 AM
      CHECK_CRON: "0 0 5 1 * *"
      TZ: Europe/Prague
    volumes:
      - ~/docker/immich/library:/data/library:ro
    security_opt:
      - no-new-privileges:true
```

### 4. Initialize Repositories

Before starting the containers, initialize the repositories:

```bash
cd ~/docker/restic

# Load environment variables
set -a && source .env && set +a

# Initialize service configs repository
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  -e RESTIC_REST_USERNAME="${RESTIC_REST_USERNAME}" \
  -e RESTIC_REST_PASSWORD="${RESTIC_REST_PASSWORD}" \
  restic/restic init \
  && echo "✓ homelab-configs repository initialized successfully" \
  || echo "✗ Failed to initialize homelab-configs repository"

# Initialize Immich repository
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${SYNOLOGY_IP}:8000/homelab-immich" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  -e RESTIC_REST_USERNAME="${RESTIC_REST_USERNAME}" \
  -e RESTIC_REST_PASSWORD="${RESTIC_REST_PASSWORD}" \
  restic/restic init \
  && echo "✓ homelab-immich repository initialized successfully" \
  || echo "✗ Failed to initialize homelab-immich repository"
```

### 5. Start Backup Containers

```bash
cd ~/docker/restic
docker compose up -d
```

### 6. Test Backups Manually

Trigger a manual backup:

```bash
# Trigger configs backup
docker exec restic-configs /usr/local/bin/backup

# Trigger Immich backup
docker exec restic-immich /usr/local/bin/backup
```

### 7. View Snapshots

```bash
cd ~/docker/restic
set -a && source .env && set +a

# List config snapshots
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic snapshots

# List Immich snapshots
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-immich" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic snapshots
```

### 8. View Backup Logs

```bash
# View configs backup logs
docker logs restic-configs

# View Immich backup logs
docker logs restic-immich

# Follow logs in real-time
docker logs -f restic-configs
```

## Restore Procedures

All restore commands use the official `restic/restic` Docker image. First, load the environment variables:

```bash
cd ~/docker/restic
set -a && source .env && set +a
```

### List Available Snapshots

```bash
# List config snapshots
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic snapshots

# List Immich snapshots
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-immich" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic snapshots

# List files in a snapshot
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic ls <snapshot-id>
```

### Restore Service Configurations

```bash
# Stop affected services
cd ~/docker/<service>
docker compose stop

# Restore entire snapshot to a staging directory
mkdir -p ~/restore-staging
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  -v ~/restore-staging:/restore \
  restic/restic restore <snapshot-id> --target /restore

# Or restore specific files/directories
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  -v ~/restore-staging:/restore \
  restic/restic restore <snapshot-id> \
    --target /restore \
    --include "/data/traefik"

# Move restored files to production
sudo cp -r ~/restore-staging/data/<service>/* ~/docker/<service>/

# Start services
docker compose start

# Cleanup
rm -rf ~/restore-staging
```

### Restore Immich

1. **Stop Immich services:**

   ```bash
   cd ~/docker/immich
   docker compose down
   ```

2. **Restore library from Restic:**

   ```bash
   cd ~/docker/restic
   set -a && source .env && set +a

   # Restore to staging directory
   mkdir -p ~/restore-staging
   docker run --rm \
     -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-immich" \
     -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
     -v ~/restore-staging:/restore \
     restic/restic restore latest --target /restore

   # Move restored files
   sudo mv ~/restore-staging/data/library/* ~/docker/immich/library/
   ```

3. **Restore database from SQL dump:**

   ```bash
   cd ~/docker/immich

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
cd ~/docker/restic
set -a && source .env && set +a

# Find the file in snapshots
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic find "*AdGuardHome.yaml*"

# Restore specific file
mkdir -p ~/restore-staging
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  -v ~/restore-staging:/restore \
  restic/restic restore <snapshot-id> \
    --target /restore \
    --include "**/AdGuardHome.yaml"
```

## Maintenance

First, load the environment variables:

```bash
cd ~/docker/restic
set -a && source .env && set +a
```

### Check Repository Health

```bash
# Quick check (verify structure)
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic check

# Full check (verify all data, slower)
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic check --read-data
```

### View Repository Statistics

```bash
# Repository stats
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic stats

# Detailed stats by snapshot
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic stats --mode raw-data
```

### Manual Snapshot Pruning

```bash
# Dry-run to see what would be deleted
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --dry-run

# Execute pruning
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --prune
```

### Update Restic

```bash
# Pull latest images
docker pull mazzolino/restic:latest
docker pull restic/restic:latest

# Restart backup containers to use new image
cd ~/docker/restic
docker compose up -d
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

1. Verify credentials in `~/docker/restic/.env`

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
cd ~/docker/restic
set -a && source .env && set +a

# List locks
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic list locks

# Remove stale locks (use with caution)
docker run --rm \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  restic/restic unlock
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
cd ~/docker/restic
set -a && source .env && set +a

# Mount repository and browse (requires FUSE on host)
mkdir -p /tmp/restic-mount
docker run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  -e RESTIC_REPOSITORY="rest:http://${RESTIC_REST_USERNAME}:${RESTIC_REST_PASSWORD}@${SYNOLOGY_IP}:8000/homelab-configs" \
  -e RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  -v /tmp/restic-mount:/mnt/restic:shared \
  restic/restic mount /mnt/restic

# Browse snapshots in another terminal
ls /tmp/restic-mount/snapshots/

# Unmount when done
fusermount -u /tmp/restic-mount
```

**Alternative:** Use `restic ls` or restore to a staging directory instead of mounting.

## Security Considerations

1. **Encryption:** All data is encrypted with your `RESTIC_PASSWORD` before leaving the homelab server

2. **Authentication:** rest-server requires HTTP Basic Auth

3. **Append-only mode:** Enable `--append-only` in rest-server for ransomware protection (clients cannot delete backups)

4. **Credential storage:** Keep `~/docker/restic/.env` secure (600 permissions) and backed up separately

5. **Read-only mounts:** Backup containers mount source directories as read-only (`:ro`) to prevent accidental modifications

6. **Network:** Consider using HTTPS with a reverse proxy for encrypted transport (optional if on trusted LAN)
