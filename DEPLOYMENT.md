# Packetpipe.com Deployment Guide

## Prerequisites
- Ubuntu/Debian server with sudo access
- Domain DNS pointing to your server IP
- NGINX installed

## Installation Steps

### 1. Install Required Packages
```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

### 2. Create Website Directory
```bash
sudo mkdir -p /var/www/packetpipe.com
sudo chown -R $USER:$USER /var/www/packetpipe.com
```

### 3. Upload Website Files
```bash
# Copy your files to the server
scp index.html *.svg *.png favicon.ico user@your-server:/var/www/packetpipe.com/
```

**Favicon and social sharing assets:** `favicon.svg`/`.ico`/`-32.png`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` and `og-image.png` must be uploaded alongside `index.html`. The OG card is rendered from `og-card.html`; to regenerate it after a change:

```bash
python3 -m http.server 8765 --bind 127.0.0.1 &
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu --hide-scrollbars \
  --window-size=1200,630 --virtual-time-budget=6000 --screenshot=og-image.png http://127.0.0.1:8765/og-card.html
```

After deploying, clear the cached preview with the [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/) or [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/).

### 4. Install Initial NGINX Configuration (HTTP Only)
```bash
# Copy the initial config file (no HTTPS yet)
sudo cp packetpipe.com-initial.conf /etc/nginx/sites-available/packetpipe.com

# Create symlink to enable the site
sudo ln -s /etc/nginx/sites-available/packetpipe.com /etc/nginx/sites-enabled/

# Test NGINX configuration
sudo nginx -t

# Reload NGINX
sudo systemctl reload nginx
```

Your site should now be accessible at http://packetpipe.com

### 5. Obtain SSL Certificate with Certbot

**If you have other sites with HTTP→HTTPS redirects**, use standalone mode:

```bash
# Stop NGINX temporarily
sudo systemctl stop nginx

# Get certificate using standalone mode
sudo certbot certonly --standalone -d packetpipe.com -d www.packetpipe.com

# Start NGINX again
sudo systemctl start nginx
```

**Or if you can temporarily disable other sites:**

```bash
# Use webroot method with your site directory
sudo certbot certonly --webroot -w /var/www/packetpipe.com -d packetpipe.com -d www.packetpipe.com
```

### 6. Upgrade to HTTPS Configuration
Once you have certificates, replace with the full HTTPS config:

```bash
# Backup the initial config
sudo mv /etc/nginx/sites-available/packetpipe.com /etc/nginx/sites-available/packetpipe.com.backup

# Copy the full HTTPS config
sudo cp packetpipe.com.conf /etc/nginx/sites-available/packetpipe.com

# Test configuration
sudo nginx -t

# Reload NGINX
sudo systemctl reload nginx
```

Your site should now redirect HTTP to HTTPS automatically.

### 7. Automatic SSL Certificate Renewal

Certbot automatically installs a cron job or systemd timer for renewals. Verify it's working:

#### Check Renewal Timer Status
```bash
sudo systemctl status certbot.timer
```

#### Test Renewal (Dry Run)
```bash
sudo certbot renew --dry-run
```

#### Manual Renewal (if needed)
```bash
sudo certbot renew
sudo systemctl reload nginx
```

#### Verify Automatic Renewal Setup
Certbot creates a systemd timer that runs twice daily. Check it:
```bash
# View timer schedule
sudo systemctl list-timers | grep certbot

# View timer configuration
sudo systemctl cat certbot.timer
```

The timer should show something like:
```
NEXT                         LEFT          LAST                         PASSED
Tue 2025-10-28 12:00:00 UTC  5h 30m left   Tue 2025-10-28 00:00:00 UTC  6h 30m ago
```

#### Set Up Reload Hook (Recommended)
Create a renewal hook to automatically reload NGINX after certificate renewal:

```bash
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy/
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Add this content:
```bash
#!/bin/bash
systemctl reload nginx
```

Make it executable:
```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Now NGINX will automatically reload when certificates renew (every 60 days).

## Maintenance Commands

### Check NGINX Status
```bash
sudo systemctl status nginx
```

### Reload NGINX (after config changes)
```bash
sudo nginx -t && sudo systemctl reload nginx
```

### View Logs
```bash
# Access logs
sudo tail -f /var/log/nginx/packetpipe.com.access.log

# Error logs
sudo tail -f /var/log/nginx/packetpipe.com.error.log
```

### Check SSL Certificate Expiry
```bash
sudo certbot certificates
```

## Security Notes

1. Certificates auto-renew 30 days before expiry
2. Certbot runs twice daily to check for renewals
3. HTTPS is enforced with HSTS header
4. Modern TLS 1.2+ only
5. Gzip compression enabled for performance

## Troubleshooting

### Certificate Renewal Fails
```bash
# Check timer logs
sudo journalctl -u certbot.timer

# Check renewal logs
sudo journalctl -u certbot.service

# Manually renew with verbose output
sudo certbot renew --verbose
```

### NGINX Won't Start
```bash
# Test configuration
sudo nginx -t

# Check for port conflicts
sudo netstat -tlnp | grep :80
sudo netstat -tlnp | grep :443
```

### DNS Not Resolving
```bash
# Check DNS
dig packetpipe.com
dig www.packetpipe.com

# Verify A records point to your server IP
```

## File Locations

- Website files: `/var/www/packetpipe.com/`
- NGINX config: `/etc/nginx/sites-available/packetpipe.com`
- SSL certificates: `/etc/letsencrypt/live/packetpipe.com/`
- Access logs: `/var/log/nginx/packetpipe.com.access.log`
- Error logs: `/var/log/nginx/packetpipe.com.error.log`
