# Dynamic DNS with Cloudflare - ddclient Configuration

This document describes the Dynamic DNS (DDNS) workflow on the home server that automatically updates Cloudflare DNS records with the current public IP address.

## Overview

The server uses **ddclient**, a Perl-based dynamic DNS client, running as a systemd daemon to keep Cloudflare DNS records synchronized with the home network's dynamic public IP address.

### How It Works

1. ddclient runs as a background daemon, checking for IP changes every 5 minutes
2. It fetches the current public IP using an external web service
3. When the IP changes, ddclient updates the DNS records via Cloudflare's API
4. Both the root domain and wildcard subdomain records are updated

## Configuration Details

### Service Status

- **Service Name**: `ddclient.service`
- **Status**: Enabled and running
- **Running Since**: Sat 2025-06-21 14:05:44 EEST

### Configuration File

**Location**: `/etc/ddclient.conf`

```conf
# General configuration
daemon=300                       # Check every 5 minutes (300 seconds)
syslog=yes                       # Log to syslog
mail-failure=root                # Notify on failure
ssl=yes                          # Use HTTPS for secure communication

# Cloudflare-specific configuration
use=web, web=dynamicdns.park-your-domain.com/getip  # Fetch external IP dynamically
protocol=cloudflare
server=api.cloudflare.com/client/v4
login=token                      # Authentication method
password=<CLOUDFLARE_API_TOKEN>  # Your Cloudflare API Token
zone=mabdellateef.fyi            # The domain zone

# DNS Records to Update
mabdellateef.fyi                 # Root domain
*.mabdellateef.fyi               # Wildcard subdomains
```

### Systemd Service File

**Location**: `/usr/lib/systemd/system/ddclient.service`

```ini
[Unit]
Documentation=man:ddclient(8)
Description=Update dynamic domain name service entries
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/ddclient.pid
Environment=daemon_interval=5m
EnvironmentFile=-/etc/default/ddclient
ExecStart=/usr/bin/ddclient -daemon $daemon_interval -syslog -pid /run/ddclient.pid
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Cache File

**Location**: `/var/cache/ddclient/ddclient.cache`

This file stores the last known IP address and update timestamps to avoid unnecessary API calls.

## DNS Records Updated

| Record | Type | Description |
|--------|------|-------------|
| `mabdellateef.fyi` | A | Root domain pointing to public IP |
| `*.mabdellateef.fyi` | A | Wildcard for all subdomains |

## Common Management Commands

### Check Service Status
```bash
systemctl status ddclient
```

### View Recent Logs
```bash
journalctl -u ddclient -f
# Or for last 50 lines:
journalctl -u ddclient -n 50
```

### Restart Service
```bash
sudo systemctl restart ddclient
```

### Force IP Update
```bash
sudo ddclient -force
```

### Test Configuration (Dry Run)
```bash
sudo ddclient -debug -verbose -noquiet
```

### Check Current Cached IP
```bash
sudo cat /var/cache/ddclient/ddclient.cache
```

## Cloudflare API Token Requirements

The Cloudflare API token used must have the following permissions:

- **Zone:DNS:Edit** - To update DNS records
- **Zone:Zone:Read** - To read zone information

### Creating a Cloudflare API Token

1. Go to Cloudflare Dashboard > My Profile > API Tokens
2. Click "Create Token"
3. Use the "Edit zone DNS" template
4. Configure:
   - **Permissions**: Zone - DNS - Edit
   - **Zone Resources**: Include - Specific zone - mabdellateef.fyi
5. Create and copy the token

## Troubleshooting

### Service Not Starting
```bash
# Check for configuration errors
sudo ddclient -debug -verbose -noquiet

# Check system logs
journalctl -u ddclient --since "1 hour ago"
```

### IP Not Updating
1. Verify the API token is valid in Cloudflare dashboard
2. Check if the IP detection URL is accessible:
   ```bash
   curl https://dynamicdns.park-your-domain.com/getip
   ```
3. Clear the cache and force update:
   ```bash
   sudo rm /var/cache/ddclient/ddclient.cache
   sudo ddclient -force
   ```

### Permission Issues
Ensure the configuration file has correct permissions:
```bash
sudo chmod 600 /etc/ddclient.conf
sudo chown root:root /etc/ddclient.conf
```

## Integration with Traefik

This DDNS setup works in conjunction with Traefik reverse proxy:

1. **ddclient** keeps the DNS records updated with the current public IP
2. **Traefik** handles incoming requests and routes them to appropriate services
3. **Let's Encrypt** (via Traefik) issues SSL certificates for the domains

The wildcard DNS record (`*.mabdellateef.fyi`) allows Traefik to handle any subdomain without requiring individual DNS record updates.

## Backup Considerations

When backing up this configuration, ensure you save:

1. `/etc/ddclient.conf` - Main configuration (contains sensitive API token)
2. `/var/cache/ddclient/ddclient.cache` - Optional, regenerates automatically

**Security Note**: The configuration file contains the Cloudflare API token. Store backups securely and never commit to version control.

## Alternative IP Detection Methods

If the default IP detection URL becomes unreliable, alternatives include:

```conf
# Using ipify
use=web, web=api.ipify.org

# Using ifconfig.me
use=web, web=ifconfig.me/ip

# Using icanhazip
use=web, web=icanhazip.com
```

## Historical Note

The configuration shows commented-out DuckDNS settings, indicating a previous migration from DuckDNS to Cloudflare for dynamic DNS management. Cloudflare provides more features including:
- Wildcard DNS support
- Proxy/CDN capabilities
- Better DNS propagation times
- Additional security features
