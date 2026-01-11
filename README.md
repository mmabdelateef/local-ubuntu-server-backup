# Traefik Setup - Backup Configuration

This folder contains the complete Traefik reverse proxy configuration backed up from 192.168.1.106.

## Files Included

- `docker-compose.yml` - Main Docker Compose configuration for Traefik and all services
- `traefik.config.yml` - Traefik static configuration (entry points, providers, ACME)

## Services Configured

1. **Traefik** - Reverse proxy with automatic HTTPS
2. **NextCloud** - `cloud.mabdellateef.fyi`
3. **Portainer** - `portainer.mabdellateef.fyi` (container management)
4. **Pi-hole** - `pihole.mabdellateef.fyi` (DNS/ad blocker)
5. **n8n** - `n8n.mabdellateef.fyi` (workflow automation)
6. **MariaDB** - Database for NextCloud

## Prerequisites for Fresh Linux Install

1. **Install Docker**
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   sudo usermod -aG docker $USER
   ```

2. **Install Docker Compose**
   ```bash
   sudo apt update
   sudo apt install docker-compose-plugin
   ```

3. **Cloudflare API Token**
   - You need the Cloudflare DNS API token: `BDJJd2HKJOFZeISUcusxhNyZUXNDLz6heE_jUYrx`
   - This is used for Let's Encrypt DNS challenge

4. **DNS Configuration**
   - Ensure these domains point to your server's public IP:
     - cloud.mabdellateef.fyi
     - portainer.mabdellateef.fyi
     - pihole.mabdellateef.fyi
     - n8n.mabdellateef.fyi

## Installation Steps

1. **Create the traefik directory**
   ```bash
   mkdir -p ~/traefik
   cd ~/traefik
   ```

2. **Copy configuration files**
   - Copy `docker-compose.yml` to `~/traefik/`
   - Copy `traefik.config.yml` to `~/traefik/`

3. **Create letsencrypt directory**
   ```bash
   mkdir -p ~/traefik/letsencrypt
   chmod 600 ~/traefik/letsencrypt
   ```

4. **Update network interface** (for Pi-hole macvlan)
   - Get your network interface name:
     ```bash
     ip addr
     # or
     ip route | grep default
     ```
   - Update the `parent` value in `docker-compose.yml` under `networks.pihole_net.driver_opts`
   - Current value: `enxf0a731b0d1fc`

5. **Verify subnet and gateway** (for Pi-hole)
   - Check your network configuration:
     ```bash
     ip route
     ```
   - Update `subnet` and `gateway` in `docker-compose.yml` under `networks.pihole_net.ipam.config`
   - Current values: `192.168.1.0/24` and `192.168.1.1`

6. **Launch the stack**
   ```bash
   cd ~/traefik
   docker-compose up -d
   ```

7. **Monitor logs**
   ```bash
   docker-compose logs -f traefik
   ```

## Auto-Start on System Boot

The entire stack is configured to start automatically when your server boots up. This is accomplished through a two-level mechanism:

### How Auto-Start Works:

1. **Docker Service Auto-Start**
   - The Docker daemon is enabled as a systemd service
   - It starts automatically when the system boots
   - Verify with: `systemctl is-enabled docker` (should show "enabled")

2. **Container Auto-Restart Policy**
   - All services in `docker-compose.yml` have `restart: always`
   - When Docker daemon starts, it automatically starts containers with this policy
   - Containers also restart automatically if they crash

### Configure Auto-Start on New System:

```bash
# Enable Docker service to start on boot (usually already enabled)
sudo systemctl enable docker

# Start Docker service now
sudo systemctl start docker

# Verify Docker is enabled
systemctl is-enabled docker  # Should output: enabled

# Start your containers (they'll auto-start on next boot)
cd ~/traefik
docker-compose up -d
```

### Verify Auto-Start:

```bash
# Check Docker service status
systemctl status docker

# Check which containers are running
docker ps

# Check container restart policy
docker inspect traefik --format='{{.HostConfig.RestartPolicy.Name}}'
# Should output: always
```

### Boot Sequence:

```
System Boots
    ↓
Systemd starts services
    ↓
Docker daemon starts (enabled via systemd)
    ↓
Docker reads container configurations
    ↓
Containers with restart=always are started
    ↓
Your entire stack is running!
```

**Note:** The `restart: always` policy in docker-compose.yml ensures that:
- Containers start on system boot
- Containers restart if they crash
- Containers restart if Docker daemon restarts
- Containers restart even if manually stopped (unless explicitly stopped with `docker stop`)

If you DON'T want auto-start, change `restart: always` to `restart: unless-stopped` in docker-compose.yml.

## Important Notes

### Passwords & Secrets
- Cloudflare API Token: `BDJJd2HKJOFZeISUcusxhNyZUXNDLz6heE_jUYrx`
- Pi-hole Web Password: `M@123`
- n8n Basic Auth User: `admin`
- n8n Basic Auth Password: `M@123`
- MariaDB Root Password: `example`
- NextCloud DB Password: `example`

### Ports Used
- 80 - HTTP (Traefik)
- 443 - HTTPS (Traefik)
- 8080 - Traefik Dashboard (insecure, accessible locally)
- 9000 - Portainer UI
- 9999 - NextCloud (direct access)
- 5678 - n8n (direct access)
- 53 - Pi-hole DNS (TCP/UDP)
- 67 - Pi-hole DHCP (UDP)

### Traefik Dashboard
Access at: `http://<server-ip>:8080`

### Data Persistence
Docker volumes will be created automatically:
- `nextcloud_data`
- `mariadb_data`
- `portainer_data`
- `./letsencrypt` - SSL certificates

### SSL Certificates
- Automatically generated via Let's Encrypt
- Uses DNS-01 challenge with Cloudflare
- Stored in `./letsencrypt/acme.json`
- Auto-renewed before expiry

## Troubleshooting

1. **Check container status**
   ```bash
   docker-compose ps
   ```

2. **View logs for specific service**
   ```bash
   docker-compose logs <service-name>
   ```

3. **Restart a service**
   ```bash
   docker-compose restart <service-name>
   ```

4. **Full restart**
   ```bash
   docker-compose down
   docker-compose up -d
   ```

5. **Certificate issues**
   - Check Cloudflare token is valid
   - Verify DNS records point to server
   - Check logs: `docker-compose logs traefik | grep acme`

6. **Pi-hole 502 Bad Gateway**

   If Pi-hole returns 502 Bad Gateway when accessing from outside, this is a network isolation issue. Pi-hole uses a macvlan network for direct LAN access, but Traefik cannot reach containers on macvlan networks from its bridge network.

   **Solution:** Pi-hole must be on BOTH networks:
   - `default` network - for Traefik connectivity
   - `pihole_net` (macvlan) - for direct LAN access at 192.168.1.190

   Required labels in docker-compose.yml:
   ```yaml
   networks:
     default:        # Required for Traefik connectivity
     pihole_net:     # Macvlan for direct LAN access
       ipv4_address: 192.168.1.190
   labels:
     - "traefik.docker.network=traefik_default"  # CRITICAL: Tell Traefik which network to use
   ```

   Also ensure ports 80/443 are NOT mapped for Pi-hole (Traefik handles external HTTPS traffic).

   **Verify fix:**
   ```bash
   # Check Traefik is using the correct IP (should be 172.x.x.x, not 192.168.x.x)
   curl -s http://localhost:8080/api/http/services | grep -A5 pihole

   # Test connectivity from Traefik container
   docker exec traefik ping -c 2 <pihole-internal-ip>
   ```

## Backup Strategy

To backup this setup again:
```bash
# From your Mac
scp mostafa@192.168.1.106:/home/mostafa/traefik/docker-compose.yml ~/Desktop/traefik-backup/
scp mostafa@192.168.1.106:/home/mostafa/traefik/traefik.config.yml ~/Desktop/traefik-backup/

# Optionally backup certificates and data
scp -r mostafa@192.168.1.106:/home/mostafa/traefik/letsencrypt ~/Desktop/traefik-backup/
```

## Security Recommendations

1. Change default passwords in production
2. Consider enabling Traefik dashboard authentication
3. Restrict Traefik dashboard to specific IPs
4. Use strong database passwords
5. Regularly update Docker images
6. Keep Cloudflare API token secure

---
Backup created: 2026-01-11
Last updated: 2026-01-11 (Pi-hole network fix + redirect middleware)
Server: 192.168.1.106
