# Certbot Renewal Setup for Multi-Site Server

## Problem
You have multiple sites with HTTP→HTTPS redirects, so certbot standalone mode won't work for renewals (port 80 is in use).

## Solution
Use **standalone for initial certificate**, then switch to **webroot for renewals**.

---

## Part 1: Get Initial Certificate (One-Time)

```bash
# Stop NGINX temporarily
sudo systemctl stop nginx

# Get certificate with standalone
sudo certbot certonly --standalone -d packetpipe.com -d www.packetpipe.com

# Start NGINX
sudo systemctl start nginx

# Install the HTTPS config
sudo cp packetpipe.com.conf /etc/nginx/sites-available/packetpipe.com
sudo nginx -t
sudo systemctl reload nginx
```

---

## Part 2: Configure for Automatic Renewals

### Step 1: Update ALL NGINX Configs to Allow ACME Challenges

For **EVERY site** on your server, add the ACME challenge location **BEFORE** any redirects.

#### For packetpipe.com
Your config already has this in `packetpipe.com.conf` - the HTTP server block includes:

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/packetpipe.com;
}
```

#### For Your Other Sites
Edit each site config and add this **BEFORE** the `if ($scheme = http)` redirect:

```nginx
server {
    listen 85.10.207.197:80;
    server_name api.metrics.gmbh;

    # ACME challenge location - MUST be BEFORE redirect
    location /.well-known/acme-challenge/ {
        root /var/www/api.metrics.gmbh;  # adjust to your webroot
    }

    # Now the redirect
    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }

    # ... rest of config
}
```

**Important:** The `location /.well-known/acme-challenge/` MUST come before any redirect logic, otherwise the redirect catches it first.

### Step 2: Test ACME Challenge Access

```bash
# Create test file for packetpipe.com
echo "test" | sudo tee /var/www/packetpipe.com/.well-known/acme-challenge/test.txt
sudo chmod -R 755 /var/www/packetpipe.com/.well-known

# Test from your local machine (NOT the server)
curl http://packetpipe.com/.well-known/acme-challenge/test.txt

# Should return "test" (not a 301 redirect)
```

### Step 3: Change Certbot to Use Webroot for Renewals

Edit the certbot renewal configuration:

```bash
sudo nano /etc/letsencrypt/renewal/packetpipe.com.conf
```

Find and change these lines:

**Before:**
```ini
authenticator = standalone
```

**After:**
```ini
authenticator = webroot

[[webroot_map]]
packetpipe.com = /var/www/packetpipe.com
www.packetpipe.com = /var/www/packetpipe.com
```

Save and exit (Ctrl+X, Y, Enter).

### Step 4: Test Renewal

```bash
# Dry run (doesn't actually renew, just tests)
sudo certbot renew --dry-run

# Should succeed with output like:
# - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
# Congratulations, all simulated renewals succeeded
# - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

If this succeeds, automatic renewals will work!

---

## How Automatic Renewals Work

1. Certbot timer runs twice daily: `sudo systemctl list-timers | grep certbot`
2. When certificate is 30 days from expiry, certbot renews it
3. Certbot writes challenge files to `/var/www/packetpipe.com/.well-known/acme-challenge/`
4. Let's Encrypt accesses `http://packetpipe.com/.well-known/acme-challenge/[random-file]`
5. NGINX serves the file (thanks to the location block you added)
6. Certbot gets the certificate and reloads NGINX
7. Your site never goes down

---

## Troubleshooting Renewals

### Test Manual Renewal
```bash
sudo certbot renew --cert-name packetpipe.com --force-renewal
```

### Check Renewal Logs
```bash
sudo journalctl -u certbot.service
```

### Verify NGINX Serves Challenge Files
```bash
# Create a test file
echo "renewal-test" | sudo tee /var/www/packetpipe.com/.well-known/acme-challenge/renewal-test.txt

# Test externally (from your local machine)
curl http://packetpipe.com/.well-known/acme-challenge/renewal-test.txt

# Should return "renewal-test" without redirect
```

### Check Renewal Configuration
```bash
sudo cat /etc/letsencrypt/renewal/packetpipe.com.conf
```

Should show:
```ini
authenticator = webroot
[[webroot_map]]
packetpipe.com = /var/www/packetpipe.com
www.packetpipe.com = /var/www/packetpipe.com
```

---

## Important Notes

1. **Initial certificate:** Use standalone (requires stopping NGINX once)
2. **Renewals:** Use webroot (NGINX stays running)
3. **All sites:** Must have ACME challenge location BEFORE redirects
4. **Permissions:** Ensure webroot directories are readable by www-data
5. **DNS:** Both packetpipe.com and www.packetpipe.com must point to your server

---

## Quick Reference Commands

```bash
# Check when certificate expires
sudo certbot certificates

# Test renewal without actually renewing
sudo certbot renew --dry-run

# Force renewal (for testing)
sudo certbot renew --cert-name packetpipe.com --force-renewal

# Check certbot timer status
sudo systemctl status certbot.timer

# View renewal configuration
sudo cat /etc/letsencrypt/renewal/packetpipe.com.conf
```
