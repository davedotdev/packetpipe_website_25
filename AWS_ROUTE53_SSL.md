# SSL Certificates with AWS Route53 DNS-01 Challenge

## Overview

The **certbot-dns-route53** plugin allows fully automated SSL certificate management using AWS Route53 DNS validation. Perfect for APIs, microservices, and wildcard certificates.

---

## Installation

### 1. Install the Route53 Plugin

```bash
sudo apt update
sudo apt install python3-certbot-dns-route53
```

Or using pip (if the apt package isn't available):
```bash
sudo pip3 install certbot-dns-route53
```

---

## Setup AWS Credentials

You need AWS credentials with Route53 permissions. There are several ways to configure this:

### Method 1: IAM Instance Role (Recommended for EC2)

If your server is an EC2 instance, attach an IAM role with Route53 permissions. No credentials file needed!

**IAM Policy needed:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:GetChange"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": "arn:aws:route53:::hostedzone/*"
        }
    ]
}
```

**To attach IAM role to EC2:**
1. Go to AWS Console → IAM → Roles
2. Create new role → EC2 as trusted entity
3. Attach the policy above
4. Go to EC2 Console → Select your instance → Actions → Security → Modify IAM Role
5. Select the role you created

### Method 2: AWS CLI Credentials File

If not on EC2, or prefer explicit credentials:

```bash
# Install AWS CLI if not already installed
sudo apt install awscli

# Configure AWS credentials
aws configure
```

You'll be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Default output format (just press Enter)

This creates `~/.aws/credentials` and `~/.aws/config`

**For certbot to use root's AWS credentials:**
```bash
# Copy to root's home directory
sudo mkdir -p /root/.aws
sudo cp ~/.aws/credentials /root/.aws/
sudo cp ~/.aws/config /root/.aws/
sudo chmod 600 /root/.aws/credentials
```

### Method 3: Dedicated Credentials File

Create a dedicated credentials file for certbot:

```bash
sudo mkdir -p /etc/letsencrypt/aws
sudo nano /etc/letsencrypt/aws/credentials.ini
```

Add:
```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
```

Secure it:
```bash
sudo chmod 600 /etc/letsencrypt/aws/credentials.ini
```

---

## Creating AWS IAM User for Certbot

If you want a dedicated IAM user (recommended for non-EC2):

### 1. Create IAM Policy

Go to AWS Console → IAM → Policies → Create Policy

Use this JSON:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:GetChange"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": "arn:aws:route53:::hostedzone/YOUR_HOSTED_ZONE_ID"
        }
    ]
}
```

Replace `YOUR_HOSTED_ZONE_ID` with your actual hosted zone ID (found in Route53 console).

Name it: `CertbotRoute53Policy`

### 2. Create IAM User

1. Go to IAM → Users → Add User
2. Username: `certbot-route53`
3. Access type: **Programmatic access** (creates API keys)
4. Attach the policy you just created
5. Copy the Access Key ID and Secret Access Key
6. Store them in your credentials file (Method 3 above)

---

## Getting Certificates

### Single Domain

```bash
sudo certbot certonly \
  --dns-route53 \
  -d api.yourdomain.com
```

### Multiple Domains

```bash
sudo certbot certonly \
  --dns-route53 \
  -d api.yourdomain.com \
  -d api2.yourdomain.com \
  -d api3.yourdomain.com
```

### Wildcard Certificate

```bash
sudo certbot certonly \
  --dns-route53 \
  -d yourdomain.com \
  -d "*.yourdomain.com"
```

This covers: `yourdomain.com`, `api.yourdomain.com`, `app.yourdomain.com`, etc.

### Using Specific Credentials File

If you created a dedicated credentials file:

```bash
sudo certbot certonly \
  --dns-route53 \
  --dns-route53-credentials /etc/letsencrypt/aws/credentials.ini \
  -d api.yourdomain.com
```

---

## How It Works

1. Certbot requests a certificate from Let's Encrypt
2. Let's Encrypt asks for DNS validation
3. Certbot automatically creates a TXT record in your Route53 hosted zone
4. Let's Encrypt checks the TXT record
5. Certificate is issued
6. Certbot removes the TXT record
7. Certificate files are saved to `/etc/letsencrypt/live/yourdomain.com/`

**All of this happens automatically!**

---

## Automatic Renewals

Renewals are **completely automatic**. The certbot timer runs twice daily and:

1. Checks if certificates are due for renewal (30 days before expiry)
2. Uses the same DNS-01 method to renew
3. Updates TXT records in Route53 automatically
4. Reloads your services (if hooks configured)

### Verify Automatic Renewal Works

```bash
# Dry run test
sudo certbot renew --dry-run

# Should show:
# Congratulations, all simulated renewals succeeded
```

### Set Up Renewal Hooks

Create a script to reload your services after renewal:

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-services.sh
```

Add:
```bash
#!/bin/bash

# Reload NGINX (if used)
systemctl reload nginx 2>/dev/null

# Reload your API services
systemctl reload your-api-service 2>/dev/null

# Reload Node.js apps
systemctl reload your-nodejs-app 2>/dev/null

echo "Services reloaded after certificate renewal"
```

Make it executable:
```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-services.sh
```

---

## Using Certificates in Your Apps

Certificates are stored at:
```
/etc/letsencrypt/live/yourdomain.com/fullchain.pem  (certificate)
/etc/letsencrypt/live/yourdomain.com/privkey.pem    (private key)
/etc/letsencrypt/live/yourdomain.com/chain.pem      (intermediate)
```

### For Node.js/Express:

```javascript
const https = require('https');
const fs = require('fs');
const express = require('express');

const app = express();

const options = {
  key: fs.readFileSync('/etc/letsencrypt/live/yourdomain.com/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/yourdomain.com/fullchain.pem')
};

https.createServer(options, app).listen(443, () => {
  console.log('HTTPS Server running on port 443');
});
```

### For NGINX Reverse Proxy:

```nginx
server {
    listen 85.10.207.197:443 ssl http2;
    server_name api.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/api.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Troubleshooting

### Error: Unable to locate credentials

**Solution:** Ensure AWS credentials are configured for root user:
```bash
sudo aws configure
# Or copy your credentials to /root/.aws/
```

### Error: Access Denied (Route53)

**Solution:** Check IAM permissions. The IAM user/role needs:
- `route53:ListHostedZones`
- `route53:GetChange`
- `route53:ChangeResourceRecordSets`

### Error: Domain not found in Route53

**Solution:** Ensure your domain's hosted zone exists in Route53:
```bash
aws route53 list-hosted-zones
```

### Check Certbot Can Access AWS

```bash
# Test AWS CLI access
sudo aws route53 list-hosted-zones

# Should list your hosted zones
```

---

## Best Practices

### 1. Use IAM Instance Role (EC2)
- Most secure
- No credentials to manage
- Automatic credential rotation

### 2. Use Dedicated IAM User (Non-EC2)
- Principle of least privilege
- Only Route53 permissions
- Easy to revoke if compromised

### 3. Restrict to Specific Hosted Zones
In IAM policy, replace:
```json
"Resource": "arn:aws:route53:::hostedzone/*"
```

With:
```json
"Resource": "arn:aws:route53:::hostedzone/Z1234567890ABC"
```

### 4. Monitor Certificate Expiry
```bash
# Check expiry dates
sudo certbot certificates

# Set up monitoring/alerts
```

### 5. Test Renewals Regularly
```bash
# Run dry-run quarterly
sudo certbot renew --dry-run
```

---

## Quick Reference

```bash
# Install plugin
sudo apt install python3-certbot-dns-route53

# Configure AWS credentials
sudo aws configure

# Get certificate
sudo certbot certonly --dns-route53 -d yourdomain.com

# Get wildcard certificate
sudo certbot certonly --dns-route53 -d "*.yourdomain.com" -d yourdomain.com

# Test renewal
sudo certbot renew --dry-run

# Force renewal (testing)
sudo certbot renew --cert-name yourdomain.com --force-renewal

# List certificates
sudo certbot certificates

# Delete certificate
sudo certbot delete --cert-name yourdomain.com
```

---

## Comparison: Route53 vs Webroot

| Feature | Route53 DNS-01 | Webroot HTTP-01 |
|---------|---------------|-----------------|
| Requires port 80 | ❌ No | ✅ Yes |
| Wildcard certs | ✅ Yes | ❌ No |
| Works behind firewall | ✅ Yes | ❌ No |
| Requires DNS access | ✅ Yes | ❌ No |
| Setup complexity | Medium | Easy |
| Renewal | Fully automatic | Fully automatic |

---

## Cost

- **Route53**: ~$0.50/month per hosted zone
- **Let's Encrypt Certificates**: Free
- **DNS queries**: $0.40 per million queries (renewal TXT records are minimal)

**Total cost**: ~$0.50/month for unlimited certificates on a domain

---

## Summary

**Route53 DNS-01 is perfect when you need:**
- ✅ Wildcard certificates
- ✅ Certificates for APIs without web servers
- ✅ Certificates behind firewalls
- ✅ Fully automated renewals
- ✅ Multiple subdomains with one cert

**Setup steps:**
1. Install `python3-certbot-dns-route53`
2. Configure AWS credentials (IAM role or AWS CLI)
3. Run `certbot certonly --dns-route53 -d yourdomain.com`
4. Done! Renewals happen automatically

No HTTP server required. No port 80 needed. Completely automated.
