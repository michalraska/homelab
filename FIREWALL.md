# Firewall Setup (UFW)

UFW (Uncomplicated Firewall) provides defense in depth for your homelab server. Since the server is behind a NAT router, external access is already blocked by default. UFW adds an extra security layer and documents which ports are expected to be open.

**Note:** External access to Jellyfin and Immich works via Cloudflare Tunnel, which bypasses the firewall entirely.

## Exposed Ports

| Port | Service | Purpose |
|------|---------|---------|
| 22/tcp | SSH | Server management |
| 53/tcp, 53/udp | AdGuard Home | Network-wide DNS filtering |
| 80/tcp | Traefik | HTTP entry point |
| 443/tcp | Traefik | HTTPS entry point (Let's Encrypt) |
| 3000/tcp | AdGuard Web UI | Direct access (alternative to Traefik) |
| 8080/tcp | qBittorrent | Web UI via Gluetun VPN |

## Installation

```bash
sudo apt install ufw
```

## Configuration

### Default Policies

Deny all incoming connections, allow all outgoing:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### Firewall Rules

```bash
# SSH with brute-force protection (rate limiting)
sudo ufw limit 22/tcp comment 'SSH'

# AdGuard DNS
sudo ufw allow 53 comment 'DNS'

# Traefik HTTP/HTTPS
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# AdGuard direct access
sudo ufw allow 3000/tcp comment 'AdGuard UI'

# qBittorrent Web UI
sudo ufw allow 8080/tcp comment 'qBittorrent'
```

### Enable Firewall

```bash
sudo ufw enable
sudo ufw status verbose
```

## Docker and UFW

Docker manipulates iptables directly and bypasses UFW for container networking. This is expected behavior and doesn't compromise security for this setup:

- Internal Docker networks are isolated from the host
- Services only expose ports through Traefik reverse proxy
- Traefik handles access control via Host header matching
- Container-to-container traffic stays within Docker networks

No additional configuration is needed. The UFW rules protect the host-exposed ports, while Docker manages container networking separately.

## Testing the Firewall

### Verify UFW Status (on the server)

```bash
# Check firewall is active and rules are loaded
sudo ufw status verbose

# List rules with numbers
sudo ufw status numbered

# Show all listening ports
sudo ss -tlnup
```

### Port Scanning with Nmap (from another device)

Install nmap: `sudo apt install nmap` (Linux) or `brew install nmap` (macOS)

```bash
# Quick TCP scan of expected open ports
nmap -p 22,53,80,443,3000,8080 <homelab-ip>

# Expected output for open ports:
# PORT     STATE SERVICE
# 22/tcp   open  ssh
# 53/tcp   open  domain
# 80/tcp   open  http
# 443/tcp  open  https
# 3000/tcp open  ppp
# 8080/tcp open  http-proxy

# Full TCP port scan (verify other ports are closed/filtered)
nmap -p- <homelab-ip>

# UDP scan for DNS (requires sudo)
sudo nmap -sU -p 53 <homelab-ip>

# Service version detection
nmap -sV -p 22,53,80,443,3000,8080 <homelab-ip>

# Comprehensive scan (TCP + common UDP + service detection)
sudo nmap -sS -sU -sV -p T:22,53,80,443,3000,8080,U:53 <homelab-ip>
```

**Interpreting nmap results:**
- `open` - Port is accepting connections (expected for allowed ports)
- `closed` - Port responds but no service listening
- `filtered` - Firewall is blocking (no response) - expected for blocked ports

### Test Specific Services

```bash
# SSH - verify connection and rate limiting
ssh -v user@<homelab-ip>

# DNS - query AdGuard
dig @<homelab-ip> google.com
nslookup google.com <homelab-ip>

# HTTP/HTTPS - test Traefik
nc -zv <homelab-ip> 80
nc -zv <homelab-ip> 443

# Test with openssl for HTTPS
openssl s_client -connect <homelab-ip>:443 -servername jellyfin.yourdomain </dev/null
```

### Verify Blocked Ports

```bash
# Scan a range that should be closed/filtered
nmap -p 9000-9100 <homelab-ip>

# All ports should show as 'closed' or 'filtered'
# If any show 'open', investigate

# Test specific blocked port with netcat
nc -zv -w 3 <homelab-ip> 9999
# Expected: Connection refused or timeout
```

### Service Verification Checklist

After enabling UFW, verify all services work:

- [ ] SSH: `ssh user@<homelab-ip>`
- [ ] DNS: `dig @<homelab-ip> example.com`
- [ ] Traefik HTTP: `curl -I http://<homelab-ip>`
- [ ] Traefik HTTPS: `curl -Ik https://<homelab-ip>`
- [ ] AdGuard UI: `http://adguard.local` or `http://<homelab-ip>:3000`
- [ ] Jellyfin: `https://jellyfin.yourdomain`
- [ ] Immich: `https://immich.yourdomain`
- [ ] qBittorrent: `http://<homelab-ip>:8080`
- [ ] All *.local domains resolve via AdGuard

### Monitor Firewall Logs

```bash
# View recent blocked connections
sudo tail -50 /var/log/ufw.log

# Watch logs in real-time during testing
sudo tail -f /var/log/ufw.log

# Filter for blocked entries
sudo grep -i "block" /var/log/ufw.log | tail -20
```

## Troubleshooting

### Service not accessible after enabling UFW

```bash
# Check if port is in UFW rules
sudo ufw status | grep <port>

# Check if service is listening
sudo ss -tlnp | grep <port>

# Temporarily disable UFW to test
sudo ufw disable
# Test service
sudo ufw enable
```

### Docker containers can't reach internet

```bash
# Check Docker's iptables rules
sudo iptables -L DOCKER-USER -n -v

# UFW should not interfere with Docker's outbound traffic
# If issues persist, check /etc/ufw/after.rules
```

### Need to see what's being blocked

```bash
# Enable UFW logging (medium verbosity)
sudo ufw logging medium

# Check logs
sudo tail -f /var/log/ufw.log
```

**Logs location:** `/var/log/ufw.log`

## Managing Rules

```bash
# List rules with numbers
sudo ufw status numbered

# Delete a rule by number
sudo ufw delete <number>

# Delete a rule by specification
sudo ufw delete allow 8080/tcp

# Insert a rule at specific position
sudo ufw insert 1 allow from 192.168.1.0/24

# Reset all rules (careful!)
sudo ufw reset
```

## Disable Firewall

If you need to temporarily disable the firewall:

```bash
# Disable
sudo ufw disable

# Re-enable
sudo ufw enable
```
