# Certbot Connection Timeout - Troubleshooting Guide

## Issue
Certbot is timing out when trying to reach your server, indicating a firewall or connectivity problem.

## Step-by-Step Troubleshooting

### 1. Check if Port 80 is Open in Firewall

```bash
# Check UFW status (Ubuntu firewall)
sudo ufw status

# If UFW is active and port 80 is not allowed, add it:
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

### 2. Check if NGINX is Listening on Port 80

```bash
# Check if NGINX is listening
sudo netstat -tlnp | grep :80

# Or use ss command
sudo ss -tlnp | grep :80

# Should show something like:
# tcp  0  0  0.0.0.0:80  0.0.0.0:*  LISTEN  12345/nginx
```

### 3. Test Local Access

```bash
# Test if NGINX is serving files locally
curl http://localhost/

# You should see your HTML content
```

### 4. Create Test File and Verify External Access

```bash
# Create the .well-known/acme-challenge directory manually
sudo mkdir -p /var/www/packetpipe.com/.well-known/acme-challenge

# Create a test file
echo "test" | sudo tee /var/www/packetpipe.com/.well-known/acme-challenge/test.txt

# Set proper permissions
sudo chown -R www-data:www-data /var/www/packetpipe.com/.well-known
sudo chmod -R 755 /var/www/packetpipe.com/.well-known

# Test locally first
curl http://localhost/.well-known/acme-challenge/test.txt

# Test from outside (from your local machine, not the server)
curl http://packetpipe.com/.well-known/acme-challenge/test.txt
```

If the external curl works, certbot should work too.

### 5. Check Cloud Provider Firewall/Security Groups

If using AWS, Google Cloud, Azure, or other cloud providers:

**AWS EC2:**
- Go to EC2 Console → Security Groups
- Find the security group attached to your instance
- Ensure inbound rules allow TCP port 80 and 443 from 0.0.0.0/0

**Google Cloud:**
- Go to VPC Network → Firewall Rules
- Ensure you have rules allowing tcp:80 and tcp:443

**Azure:**
- Go to Network Security Groups
- Check inbound security rules for ports 80 and 443

**DigitalOcean:**
- Go to Networking → Firewalls
- Check if ports 80 and 443 are allowed

### 6. Check DNS Resolution

```bash
# Verify DNS is pointing to your server
dig packetpipe.com +short
dig www.packetpipe.com +short

# Should return your server's public IP address
# Compare with:
curl -4 ifconfig.me
```

### 7. Check NGINX Error Logs

```bash
# Check for any NGINX errors
sudo tail -f /var/log/nginx/error.log

# Check access logs to see if Let's Encrypt is reaching your server
sudo tail -f /var/log/nginx/access.log
```

Look for Let's Encrypt bot requests from IP addresses like 66.133.109.*

### 8. Test with Certbot in Verbose Mode

```bash
# Run certbot with verbose logging
sudo certbot certonly --webroot -w /var/www/packetpipe.com \
  -d packetpipe.com -d www.packetpipe.com --verbose
```

## Common Issues and Solutions

### Issue: Port 80 blocked by cloud firewall
**Solution:** Add inbound rule for TCP port 80 from 0.0.0.0/0 in your cloud provider's security group/firewall

### Issue: UFW blocking connections
**Solution:**
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### Issue: DNS not resolving correctly
**Solution:** Wait for DNS propagation (can take up to 48 hours) or update DNS records

### Issue: NGINX not running
**Solution:**
```bash
sudo systemctl status nginx
sudo systemctl start nginx
```

### Issue: Wrong webroot permissions
**Solution:**
```bash
sudo chown -R www-data:www-data /var/www/packetpipe.com
sudo chmod -R 755 /var/www/packetpipe.com
```

## Alternative: Use Standalone Mode

If webroot continues to fail, use standalone mode (requires stopping NGINX temporarily):

```bash
# Stop NGINX
sudo systemctl stop nginx

# Run certbot in standalone mode (it will start its own web server)
sudo certbot certonly --standalone -d packetpipe.com -d www.packetpipe.com

# Start NGINX
sudo systemctl start nginx

# Then switch to the HTTPS config
sudo cp packetpipe.com.conf /etc/nginx/sites-available/packetpipe.com
sudo nginx -t
sudo systemctl reload nginx
```

## Quick Diagnostic Script

Run this to check everything at once:

```bash
#!/bin/bash
echo "=== NGINX Status ==="
sudo systemctl status nginx | head -3

echo -e "\n=== Port 80 Listening ==="
sudo netstat -tlnp | grep :80

echo -e "\n=== Firewall Status ==="
sudo ufw status | grep -E "80|443|Status"

echo -e "\n=== DNS Resolution ==="
dig packetpipe.com +short
dig www.packetpipe.com +short

echo -e "\n=== Server Public IP ==="
curl -4 -s ifconfig.me

echo -e "\n=== Test Local Access ==="
curl -I http://localhost/ 2>&1 | head -1

echo -e "\n=== Test Challenge Directory ==="
ls -la /var/www/packetpipe.com/.well-known/acme-challenge/ 2>&1 | head -5
```

Save this as `diagnose.sh`, make it executable with `chmod +x diagnose.sh`, and run with `./diagnose.sh`
