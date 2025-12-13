# Homelab Setup with Docker Compose

Docker Compose setup for media management, photo storage, DNS filtering, and external access via Cloudflare Tunnel. Includes Traefik reverse proxy with network segmentation.

## Services Included

| Service | Purpose | Access | Network |
|---------|---------|--------|---------|
| **Traefik** | Reverse proxy | `http://traefik.local` (intranet) | ingress, services |
| **Jellyfin** | Media server | `https://jellyfin.yourdomain` (external + intranet) | services |
| **Immich** | Photo storage | `https://immich.yourdomain` (external + intranet) | services, immich |
| **Sonarr** | TV automation | `http://sonarr.yourdomain` (intranet) | services |
| **Radarr** | Movie automation | `http://radarr.yourdomain` (intranet) | services |
| **Prowlarr** | Indexer manager | `http://prowlarr.yourdomain` (intranet) | services |
| **Bazarr** | Subtitle manager | `http://bazarr.yourdomain` (intranet) | services |
| **Homarr** | Dashboard | `http://homarr.yourdomain` (intranet) | services |
| **Dashdot** | System monitor | `http://dashdot.yourdomain` (intranet) | services |
| **AdGuard Home** | DNS filtering | `http://adguard.local` (intranet), DNS:53 | services |
| **qBittorrent** | Download client | `http://localhost:8080` (VPN) | vpn-isolated (via Gluetun) |
| **Gluetun** | VPN gateway | Internal only | vpn-isolated, services |
| **Cloudflare Tunnel** | External routing | Routes to Traefik | ingress |

## Architecture

```
External (Internet)
   ↓
Cloudflare Tunnel (HTTPS encryption)
   ↓
Traefik on port 80 (HTTP from tunnel)
   ├─→ Immich (2283) with HTTPS/TLS
   └─→ Jellyfin (8096) with HTTPS/TLS

Intranet (Local Network)
   ↓
AdGuard Home (DNS filtering on 53/TCP+UDP + DNS rewrites for *.yourdomain and *.local)
   ↓
Traefik on port 80 (HTTP)
   ├─→ Dashboard via traefik.local (API routing only)
   ├─→ AdGuard Home Web UI via adguard.local (3000)
   ├─→ Jellyfin via jellyfin.yourdomain (8096)
   ├─→ Immich via immich.yourdomain (2283)
   ├─→ Sonarr via sonarr.yourdomain (8989)
   ├─→ Radarr via radarr.yourdomain (7878)
   ├─→ Prowlarr via prowlarr.yourdomain (9696)
   ├─→ Bazarr via bazarr.yourdomain (6767)
   ├─→ Homarr via homarr.yourdomain (7575)
   └─→ Dashdot via dashdot.yourdomain (3001)

VPN Network (Isolated)
   ├─→ Gluetun (VPN Gateway) → Also on services network for arr access
   └─→ qBittorrent (8080) → All traffic via VPN
```

## Directory Structure

```
docker/
├── arr/
│   ├── data/
│   │   ├── cache/           # Jellyfin cache
│   │   ├── media/
│   │   │   ├── movies/      # Radarr movies library
│   │   │   └── tv/          # Sonarr TV shows library
│   │   └── torrents/        # qBittorrent downloads (hardlinks enabled)
│   │       ├── incomplete/  # In-progress downloads
│   │       ├── movies/      # Completed movie torrents
│   │       ├── tv/          # Completed TV torrents
│   │       └── torrents/    # .torrent files
│   ├── sonarr/             # Sonarr config
│   ├── radarr/             # Radarr config
│   ├── prowlarr/           # Prowlarr config
│   ├── bazarr/             # Bazarr config
│   ├── jellyfin/           # Jellyfin config
│   ├── homarr/             # Homarr config
│   ├── qbittorrent/        # qBittorrent config
│   └── gluetun/            # Gluetun VPN config
├── immich/
│   ├── library/            # Photos/videos
│   ├── upload/             # Temporary uploads
│   ├── postgres/           # PostgreSQL database
│   └── redis/              # Redis cache
├── adguard/
│   ├── work/               # AdGuard working directory
│   └── conf/               # AdGuard configuration
├── traefik/
│   ├── acme/               # Let's Encrypt certificates
│   └── config/             # Traefik configuration
├── cloudflare-tunnel/      # Cloudflare Tunnel credentials
└── dashdot/                # Dashdot configuration
```

**Note:** This setup uses local repository directories for portability. All paths are relative to the repository root.

## Prerequisites

1. **Operating System**: Ubuntu 24.04 LTS (tested and recommended)
2. **Docker** (29.1+) and **Docker Compose** (5.0+)
3. **Storage**: A filesystem supporting hardlinks (ext4, btrfs, etc.) for optimal *arr stack performance
4. **Cloudflare Account** with a domain
5. **NordVPN Account** with OpenVPN service credentials
6. **User/Group IDs**: Know your PUID and PGID (typically 1000:1000)

## Setup Instructions

### 1. Secure SSH Access

Before proceeding, secure SSH access to your server using key-based authentication:

**On your local machine (generate SSH key if you don't have one):**
```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
ssh-copy-id user@<homelab-ip>
```

Alternatively, use [1Password SSH Agent](https://developer.1password.com/docs/ssh/get-started/) to manage SSH keys securely.

**On the server (apply hardened SSH configuration):**
```bash
# Copy the provided SSH hardening configuration
sudo cp ssh/sshd_config /etc/ssh/sshd_config.d/99-hardening.conf

# Test configuration before applying
sudo sshd -t

# Restart SSH service
sudo systemctl restart ssh
```

**Important:** Keep your current SSH session open and test login in a new terminal before closing. This prevents lockout if configuration is incorrect.

The configuration disables password authentication, root login, and enforces modern cryptographic algorithms. See [ssh/sshd_config](ssh/sshd_config) for details.

### 2. Install Docker Engine (Ubuntu)

Install Docker using the official APT repository (recommended for production):

```bash
# Install prerequisites
sudo apt update
sudo apt install ca-certificates curl

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository to APT sources
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

# Install Docker Engine and Docker Compose plugin
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to the docker group (to run without sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

### 3. Clone the Repository

```bash
git clone git@github.com:michalraska/homelab.git ~/docker
cd ~/docker
```

### 4. Initial Configuration

```bash
# Copy environment template
cp .env.example .env

# Edit .env with your specific values
vi .env
```

### 5. Required Environment Variables

Environment variables are distributed across service-specific directories for better organization.
The `TZ` (timezone) variable is defined once in the root `.env` and inherited by all services.

**Root `.env`** (shared settings):
```bash
TZ=Europe/Prague                             # System timezone (inherited by all services)
```

**Traefik** (`traefik/.env`):
```bash
ACME_EMAIL=your-email@example.com           # Let's Encrypt notifications
CLOUDFLARE_API_KEY=<scoped API token>       # Cloudflare DNS challenge token
```

**Cloudflare Tunnel** (`cloudflare-tunnel/.env`):
```bash
CLOUDFLARE_TUNNEL_TOKEN=<generated token>   # From cloudflared tunnel token command
```

**Immich** (`immich/.env`):
```bash
DOMAIN=yourdomain.com                       # Your domain
DB_USERNAME=immich
DB_PASSWORD=<secure password>               # Generate with: openssl rand -base64 32
DB_DATABASE_NAME=immich
IMMICH_VERSION=release
```

**ARR Stack** (`arr/.env`):
```bash
DOMAIN=yourdomain.com                       # Your domain
PUID=1000                                   # Your user ID
PGID=1000                                   # Your group ID
OPENVPN_USER=<nordvpn service username>     # NordVPN service credentials
OPENVPN_PASSWORD=<nordvpn service password>
SERVER_COUNTRIES=Czech Republic             # VPN server location
```

**Dashdot** (`dashdot/.env`):
```bash
DOMAIN=yourdomain.com
```

**AdGuard** (`adguard/.env`):
```bash
# No service-specific variables required (TZ inherited from root)
```

### 6. Get NordVPN OpenVPN Credentials

1. Go to [NordVPN Dashboard](https://my.nordvpn.com/dashboard/)
2. Navigate to **Manual Setup** or **Service Credentials**
3. Generate service credentials for OpenVPN
4. Copy username and password to `arr/.env` as `OPENVPN_USER` and `OPENVPN_PASSWORD`
5. Set `SERVER_COUNTRIES` to your preferred location (e.g., `Czech Republic`)

### 7. Set Up Cloudflare for HTTPS (Let's Encrypt DNS Challenge)

Before setting up the tunnel, create a scoped API token for Let's Encrypt:

1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click **"Create Token"**
3. Select the **"Edit zone DNS"** template
4. Configure:
   - **Permissions**: Zone > DNS > Edit, Zone > Zone > Read
   - **Zone Resources**: Include all zones
5. Copy the token to `traefik/.env` as `CLOUDFLARE_API_KEY`
6. Also set `ACME_EMAIL` to your email for Let's Encrypt notifications

```bash
# Example traefik/.env configuration
ACME_EMAIL=your-email@example.com
CLOUDFLARE_API_KEY=your_scoped_api_token_here
```

### 8. Set Up Cloudflare Tunnel

```bash
# Install cloudflared locally (if not already)
# macOS: brew install cloudflare/cloudflare/cloudflared
# Linux: curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared

# Create tunnel
cloudflared tunnel create homelab

# Generate token
cloudflared tunnel token homelab

# Copy the token to cloudflare-tunnel/.env as CLOUDFLARE_TUNNEL_TOKEN
```

### 9. Start Services

```bash
# Validate compose file
docker compose config

# Start all services (detached)
docker compose up -d

# Check logs
docker compose logs -f

# Check service health
docker compose ps
```

## Post-Setup Configuration

### AdGuard Home

**Initial Setup:**
1. Access directly via: `http://<homelab-ip>:3000`
2. Complete initial setup wizard
3. Create admin account with strong password
4. **Configure web interface port** (if not already set to 3000):
   - Edit `docker/adguard/conf/AdGuardHome.yaml`
   - Under `http:` section, set `address: 0.0.0.0:3000`
   - This prevents collision with other services (Traefik uses port 80)
   - Restart AdGuard: `docker compose restart adguard-home`
5. After setup, you can also access via Traefik: `http://adguard.local` (requires DNS rewrite configuration below)

**Configure DNS Filtering:**
- Set router/devices to use `<homelab-ip>:53` as DNS server
- All DNS queries will be filtered through AdGuard

**Configure UniFi Network to use AdGuard Home:**
1. Open UniFi Network Console
2. Go to **Settings** → **Internet**
3. Select your WAN connection
4. Under **DNS Server**, set **Primary Server** to `<homelab-ip>`
5. Set **Secondary Server** to `86.54.11.13` (DNS4EU fallback)
6. Click **Apply Changes**

This ensures all devices on your network use AdGuard Home for DNS resolution without manual configuration on each device.

**DNS Rewrites for Internal Services (Important for Traefik Routing):**

Use AdGuard's **DNS rewrites** feature to resolve hostnames to your homelab IP:

1. Go to **Filters** → **DNS rewrites**
2. Add entries for `.local` domains:
   ```
   traefik.local → <homelab-ip>
   adguard.local → <homelab-ip>
   ```
3. Add wildcard entry for your domain (replace `yourdomain` with your actual domain):
   ```
   *.yourdomain → <homelab-ip>
   ```
   This will resolve all service URLs like:
   - jellyfin.yourdomain
   - sonarr.yourdomain
   - radarr.yourdomain
   - prowlarr.yourdomain
   - bazarr.yourdomain
   - immich.yourdomain
   - homarr.yourdomain
   - dashdot.yourdomain

**Why This Matters:**
- Without DNS rewrites, `.local` hostnames won't resolve on your network
- Traefik uses Host header matching (`Host: adguard.local`) for routing
- AdGuard provides the DNS resolution that makes `.local` hostnames work
- All traffic still goes through Traefik on port 80
- AdGuard's web UI is now accessible via Traefik instead of direct port mapping

**Network Architecture:**
- AdGuard runs on `services` network (internal only)
- Traefik bridges traffic from `ingress` network to `services` network
- DNS ports (53) remain exposed on host for network-wide DNS filtering
- Web UI (3000) is only accessible through Traefik routing

### Configure systemd-resolved for AdGuard Home (Ubuntu Server)

To use AdGuard Home as the primary DNS resolver on Ubuntu Server, configure systemd-resolved:

**1. Create systemd-resolved configuration file:**

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf > /dev/null <<EOF
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
EOF
```

**2. Backup and replace /etc/resolv.conf:**

```bash
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

**3. Restart systemd-resolved:**

```bash
sudo systemctl restart systemd-resolved
```

**4. Verify DNS configuration:**

```bash
# Check that 127.0.0.1 is configured as DNS
resolvectl status

# Test DNS resolution through AdGuard
nslookup example.com 127.0.0.1
```

**What this does:**
- `DNS=127.0.0.1`: Routes all DNS queries to AdGuard Home on localhost
- `DNSStubListener=no`: Disables the local stub DNS listener (127.0.0.53) to avoid conflicts
- Symlinks `/etc/resolv.conf` to systemd-resolved's config for persistence
- Ensures AdGuard Home becomes the system-wide DNS resolver

**Important Notes:**
- This configuration requires AdGuard Home to be running (`docker compose up -d adguard-home`)
- To revert: `sudo rm /etc/resolv.conf && sudo mv /etc/resolv.conf.backup /etc/resolv.conf && sudo systemctl restart systemd-resolved`

### Traefik Dashboard (Internal Only)

1. Access: `http://traefik.local` (from intranet)
2. Verify all services are discovered
3. Check router/service status

### Jellyfin
1. **Intranet**: `https://jellyfin.yourdomain`
2. **External**: `https://jellyfin.yourdomain` (HTTPS/TLS with Let's Encrypt)
3. Complete initial setup
4. Add libraries:
   - TV Shows: `/data/media/tv`
   - Movies: `/data/media/movies`

### Sonarr
1. Access: `http://sonarr.yourdomain`
2. Configure root folder: `/data/media/tv`
3. Set download client to qBittorrent:
   - Host: `gluetun`
   - Port: `8080`
   - Category: `sonarr`

### Radarr
1. Access: `http://radarr.yourdomain`
2. Configure root folder: `/data/media/movies`
3. Set download client to qBittorrent:
   - Host: `gluetun`
   - Port: `8080`
   - Category: `radarr`

### Prowlarr
1. Access: `http://prowlarr.yourdomain`
2. Add indexers
3. Sync to Sonarr and Radarr

### Bazarr
1. Access: `http://bazarr.yourdomain`
2. Configure subtitle providers
3. Link to Sonarr and Radarr

### Homarr (Dashboard)
1. Access: `http://homarr.yourdomain`
2. Complete initial setup
3. Add service integrations for all your homelab services
4. Configure widgets and customize layout

### Dashdot (System Monitor)
1. Access: `http://dashdot.yourdomain`
2. View real-time system resources
3. Monitor CPU, RAM, disk I/O, and network usage
4. No configuration needed - auto-discovers system info

### qBittorrent (Behind VPN)
1. Access via: `http://localhost:8080` (exposed through Gluetun)
2. Verify VPN connection is active (check IP in WebUI)
3. Configure categories in qBittorrent WebUI:
   - **Settings** → **Downloads** → **Default Save Path**: `/data/torrents/`
   - Create category "radarr" with save path: `/data/torrents/movies/`
   - Create category "sonarr" with save path: `/data/torrents/tv/`
4. **Important**: All qBittorrent traffic routes through VPN - IP leaks are prevented at network level

### Immich
1. **Intranet**: `https://immich.yourdomain`
2. **External**: `https://immich.yourdomain` (HTTPS/TLS with Let's Encrypt)
3. Create initial admin account
4. Configure backup directory

### Cloudflare Tunnel
1. Access [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Navigate to Access → Tunnels
3. Verify tunnel is connected and showing status
4. Cloudflare automatically routes to Traefik (no manual route config needed)

## Security Considerations

### Network Isolation
- **ingress**: Traefik and Cloudflare Tunnel only (entry point for all external traffic)
- **services**: Internal services (AdGuard, Sonarr, Radarr, Jellyfin, Immich, etc.) and Traefik - isolated from direct external access at the Docker network level
- **immich**: Immich database (PostgreSQL) and cache (Redis) isolated from other services
- **vpn-isolated**: qBittorrent isolated, all traffic enforced through VPN (Gluetun bridges to services network for arr stack access)

### Reverse Proxy
- Traefik is single entry point for both external and internal traffic
- Jellyfin and Immich exposed to Cloudflare Tunnel (via Traefik)
  - Immich: HTTPS/TLS with Let's Encrypt
  - Jellyfin: HTTPS/TLS with Let's Encrypt
- Arr services (Sonarr, Radarr, etc.) only accessible from services network via Traefik
- No direct port exposure of internal services to host (except AdGuard DNS on port 53)

### Container Hardening
- All services use `security_opt: no-new-privileges:true`
- Traefik and Cloudflared have all capabilities dropped
- Immich services drop NET_RAW capability
- Gluetun only keeps NET_ADMIN (required for VPN)

### Secrets Management
- Use `.env` file for sensitive data (never commit to git)
- Generate strong passwords: `openssl rand -base64 32`
- Cloudflare tunnel token stored in `.env`
- Database passwords should be 32+ characters

## Hardlinks & Storage Optimization

The setup follows TRaSH Guides best practices for hardlinks between qBittorrent downloads and *arr destinations:

- Filesystem must support hardlinks (ext4, btrfs, zfs, etc.)
- All arr services use unified `/data` mount point (mapped to `./arr/data/` on host)
- qBittorrent downloads to `/data/torrents/` (categories: `/data/torrents/movies/`, `/data/torrents/tv/`)
- Sonarr/Radarr move files via hardlinks to `/data/media/tv/` and `/data/media/movies/`
- **Result**: No disk space duplication, instant moves, optimal performance

## Maintenance

### Health Checks
All services include health checks. Monitor with:
```bash
docker compose ps
```

### Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f jellyfin

# Last 100 lines
docker compose logs --tail=100 jellyfin
```

### Updates
```bash
# Pull latest images
docker compose pull

# Restart services
docker compose up -d --no-deps --build
```

### Cleanup
```bash
# Remove stopped containers
docker compose down

# Remove dangling volumes
docker volume prune

# Full cleanup (⚠️ removes all containers and networks)
docker compose down -v
```

## License

This setup is provided as-is for personal use.

## Contributing

Feel free to customize and improve this setup for your needs.
