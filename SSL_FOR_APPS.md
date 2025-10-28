# SSL Certificates for React/Next.js/APIs

## The Challenge You're Describing: DNS-01

What you're referring to (like Caard.io uses) is the **DNS-01 challenge** method. Instead of serving files via HTTP, you prove domain ownership by creating a TXT record in DNS.

---

## Methods for Different App Types

### Method 1: DNS-01 Challenge (Recommended for APIs & Apps)

**Best for:**
- APIs without web servers
- Apps that can't serve HTTP challenges
- Wildcard certificates (`*.yourdomain.com`)
- Multiple subdomains

**How it works:**
1. Certbot asks you to create a TXT record in DNS
2. Let's Encrypt verifies the TXT record
3. Certificate is issued
4. Your app uses the certificate files

**Setup with DNS automation:**

```bash
# Install certbot with your DNS provider plugin
# Example for Cloudflare:
sudo apt install python3-certbot-dns-cloudflare

# For other providers:
# python3-certbot-dns-route53 (AWS Route53)
# python3-certbot-dns-google (Google Cloud DNS)
# python3-certbot-dns-digitalocean
# python3-certbot-dns-ovh
```

**Example: Cloudflare (most common)**

```bash
# 1. Create API token in Cloudflare dashboard
#    - My Profile → API Tokens → Create Token
#    - Use "Edit zone DNS" template
#    - Permissions: Zone / DNS / Edit
#    - Zone Resources: Include / Specific zone / yourdomain.com

# 2. Create credentials file
sudo mkdir -p /etc/letsencrypt/cloudflare
sudo nano /etc/letsencrypt/cloudflare/credentials.ini
```

Add this content:
```ini
dns_cloudflare_api_token = YOUR_API_TOKEN_HERE
```

```bash
# 3. Secure the file
sudo chmod 600 /etc/letsencrypt/cloudflare/credentials.ini

# 4. Get certificate (single domain)
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d api.yourdomain.com

# 5. Get wildcard certificate (covers *.yourdomain.com)
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d yourdomain.com \
  -d "*.yourdomain.com"
```

**Automatic renewals:** Work automatically! Certbot uses the API token to update DNS TXT records.

---

### Method 2: Reverse Proxy (NGINX) - Current Setup Extended

**Best for:**
- React/Next.js apps running on different ports
- APIs behind NGINX
- Multiple apps on same server

**How it works:**
- NGINX handles SSL termination
- Apps run on localhost ports (3000, 4000, etc.)
- NGINX proxies requests to apps

**Example for React app on port 3000:**

```nginx
server {
    listen 85.10.207.197:80;
    server_name app.yourdomain.com;

    location ^~ /.well-known/acme-challenge/ {
        allow all;
        root /var/www/app.yourdomain.com;
        default_type "text/plain";
        try_files $uri =404;
    }

    location / {
        return 301 https://$server_name$request_uri;
    }
}

server {
    listen 85.10.207.197:443 ssl http2;
    server_name app.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/app.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.yourdomain.com/privkey.pem;

    location ^~ /.well-known/acme-challenge/ {
        allow all;
        root /var/www/app.yourdomain.com;
        default_type "text/plain";
        try_files $uri =404;
    }

    # Proxy to React app
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Get certificate:**
```bash
# Create webroot directory
sudo mkdir -p /var/www/app.yourdomain.com/.well-known/acme-challenge

# Get certificate using webroot
sudo certbot certonly --webroot \
  -w /var/www/app.yourdomain.com \
  -d app.yourdomain.com
```

---

### Method 3: Next.js with Built-in Server

**For Next.js apps that handle their own HTTPS:**

Next.js can serve HTTPS directly. Point certbot certificates to your Next.js config:

```javascript
// server.js
const { createServer } = require('https');
const { parse } = require('url');
const next = require('next');
const fs = require('fs');

const dev = process.env.NODE_ENV !== 'production';
const app = next({ dev });
const handle = app.getRequestHandler();

const httpsOptions = {
  key: fs.readFileSync('/etc/letsencrypt/live/yourdomain.com/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/yourdomain.com/fullchain.pem')
};

app.prepare().then(() => {
  createServer(httpsOptions, (req, res) => {
    const parsedUrl = parse(req.url, true);
    handle(req, res, parsedUrl);
  }).listen(3000, (err) => {
    if (err) throw err;
    console.log('> Ready on https://localhost:3000');
  });
});
```

**Get certificate with standalone:**
```bash
# Stop Next.js temporarily
sudo systemctl stop your-nextjs-app

# Get certificate
sudo certbot certonly --standalone -d yourdomain.com

# Start Next.js
sudo systemctl start your-nextjs-app
```

**For renewals:** Use DNS-01 or HTTP-01 with webroot, then reload your app.

---

## Comparison of Methods

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **DNS-01** | No HTTP needed, wildcards, works with firewalls | Requires DNS API access | APIs, microservices, wildcards |
| **HTTP-01 + NGINX Proxy** | Simple, works with existing setup | Need port 80 open | Apps behind reverse proxy |
| **Direct App SSL** | No proxy needed | App must handle renewals | Standalone apps |

---

## Recommended Setup for Your Use Case

Given you already have NGINX and multiple sites, I recommend:

### For React/Next.js Apps:
**Use NGINX reverse proxy** (Method 2)
- NGINX handles SSL
- Apps run on localhost ports
- Use webroot for renewals (already working)
- Centralized SSL management

### For APIs:
**Use DNS-01** (Method 1)
- No HTTP server needed
- Works even if API is internal
- Can get wildcard certs
- Fully automated with DNS provider plugin

---

## Step-by-Step: Setting Up DNS-01 (Cloudflare Example)

### 1. Install Cloudflare Plugin
```bash
sudo apt install python3-certbot-dns-cloudflare
```

### 2. Get Cloudflare API Token
- Go to https://dash.cloudflare.com/profile/api-tokens
- Click "Create Token"
- Use "Edit zone DNS" template
- Select your domain
- Copy the token

### 3. Create Credentials File
```bash
sudo mkdir -p /etc/letsencrypt/cloudflare
sudo nano /etc/letsencrypt/cloudflare/credentials.ini
```

Add:
```ini
dns_cloudflare_api_token = your_token_here
```

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare/credentials.ini
```

### 4. Get Certificate
```bash
# Single domain
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d api.yourdomain.com

# Wildcard (covers all subdomains)
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d yourdomain.com \
  -d "*.yourdomain.com"
```

### 5. Configure Your App
Point your app to use the certificates:
```
Certificate: /etc/letsencrypt/live/yourdomain.com/fullchain.pem
Private Key: /etc/letsencrypt/live/yourdomain.com/privkey.pem
```

### 6. Set Up Renewal Hook
Create a script to reload your app after renewal:

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-apps.sh
```

Add:
```bash
#!/bin/bash
# Reload your apps after certificate renewal
systemctl reload your-api-service
systemctl reload your-nextjs-app
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-apps.sh
```

### 7. Test Renewal
```bash
sudo certbot renew --dry-run
```

---

## DNS Providers Supported by Certbot

- Cloudflare (most popular)
- AWS Route53
- Google Cloud DNS
- DigitalOcean
- OVH
- Linode
- And many more...

Full list: https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins

---

## Quick Reference Commands

```bash
# List all certificates
sudo certbot certificates

# Renew all certificates
sudo certbot renew

# Renew specific certificate
sudo certbot renew --cert-name yourdomain.com

# Delete a certificate
sudo certbot delete --cert-name yourdomain.com

# Test renewal (dry run)
sudo certbot renew --dry-run
```

---

## Summary

**For your setup:**

1. **Static sites** (like packetpipe.com): Use webroot ✅ (already working)
2. **React/Next.js apps**: Use NGINX reverse proxy + webroot
3. **APIs**: Use DNS-01 challenge with your DNS provider plugin
4. **Wildcard certs**: Use DNS-01 (only method that supports wildcards)

The DNS-01 method is what Caard.io and similar services use - it's fully automated and doesn't require any HTTP server access!
