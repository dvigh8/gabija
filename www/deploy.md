# Gabija Website Deployment Guide

This guide covers deploying the Gabija coming soon website to your Arch Linux server with nginx and SSL.

## Prerequisites

### Server Requirements
- Arch Linux server with ssh access
- Domain `gabijja.app` pointing to server IP
- nginx installed and configured
- Let's Encrypt certificates (already configured for gabijja.app)
- sudo/root access on server

### Local Requirements
- Git for version control
- SSH client for server access

## Directory Structure

The website follows this structure:
```
/var/www/gabijja.app/
├── index.html                 # Main coming soon page
├── css/
│   └── style.css             # Main stylesheet
├── assets/
│   ├── favicon.ico           # Favicon (to be added)
│   ├── images/              # Future images
│   └── icons/               # PNG icons
└── deploy.md                # This deployment guide
```

## Deployment Steps

### 1. Upload Files to Server

#### Method A: Using scp (Recommended for single deployment)
```bash
# From your local machine, navigate to the website directory
cd /Users/josephsd/work/gabija/main/www

# Upload entire directory to server
scp -r . user@your-server-ip:/var/www/gabijja.app/
```

#### Method B: Using Git (Recommended for ongoing updates)
```bash
# Clone the repository on the server (if using git)
git clone https://github.com/dvigh8/gabija.git /var/www/gabijja.app

# Or if you have the files locally
rsync -avz --delete ./ user@your-server-ip:/var/www/gabijja.app/
```

### 2. Set Correct Permissions

On the Arch Linux server, ensure proper ownership and permissions:

```bash
# Set ownership to nginx user (http on Arch Linux)
sudo chown -R http:http /var/www/gabijja.app

# Set proper permissions
sudo find /var/www/gabijja.app -type d -exec chmod 755 {} \;
sudo find /var/www/gabijja.app -type f -exec chmod 644 {} \;

# Verify permissions
ls -la /var/www/gabijja.app
```

### 3. Configure nginx

Create or update nginx configuration for gabijja.app:

```bash
# Create nginx config file
sudo nano /etc/nginx/sites-available/gabijja.app
```

Add this configuration:

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name gabijja.app www.gabijja.app;
    return 301 https://$server_name$request_uri;
}

# HTTPS server block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name gabijja.app www.gabijja.app;
    
    root /var/www/gabijja.app;
    index index.html;
    
    # SSL configuration (managed by Certbot)
    ssl_certificate /etc/letsencrypt/live/gabijja.app/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/gabijja.app/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https:; connect-src 'self'; script-src 'self';" always;
    
    # Performance optimizations
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;
    
    # Cache static assets
    location ~* \.(css|js|ico|png|jpg|jpeg|gif|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # Main location
    location / {
        try_files $uri $uri/ =404;
        
        # Add security headers for HTML
        location ~* \.html$ {
            add_header Cache-Control "no-cache, no-store, must-revalidate";
            add_header Pragma "no-cache";
            add_header Expires "0";
        }
    }
    
    # Error pages
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    
    # Logging
    access_log /var/log/nginx/gabijja.app.access.log;
    error_log /var/log/nginx/gabijja.app.error.log;
}
```

Enable the site:

```bash
# Create symbolic link if not exists
sudo ln -s /etc/nginx/sites-available/gabijja.app /etc/nginx/sites-enabled/

# Test nginx configuration
sudo nginx -t

# If test passes, reload nginx
sudo systemctl reload nginx
```

### 4. SSL Certificate Setup

If SSL certificates aren't already set up:

```bash
# Install certbot if not already installed
sudo pacman -S certbot certbot-nginx

# Obtain certificate
sudo certbot --nginx -d gabijja.app -d www.gabijja.app

# Set up auto-renewal (Arch Linux specific)
sudo systemctl enable certbot-renew.timer
sudo systemctl start certbot-renew.timer

# Check timer status
sudo systemctl status certbot-renew.timer
```

### 5. Verify Deployment

#### Test nginx configuration:
```bash
sudo nginx -t
sudo systemctl status nginx
```

#### Check file permissions:
```bash
sudo -u http ls -la /var/www/gabijja.app/
sudo -u http cat /var/www/gabijja.app/index.html
```

#### Test website access:
```bash
# Test HTTP response
curl -I http://gabijja.app

# Test HTTPS response
curl -I https://gabijja.app

# Test locally on server
curl -I http://localhost
```

## Ongoing Maintenance

### Update Process

When making changes to the website:

1. **Make changes locally** in `/Users/josephsd/work/gabija/main/www/`
2. **Test locally** by opening `index.html` in browser
3. **Upload changes** using your preferred method:

   ```bash
   # Using rsync for updates
   rsync -avz --delete /Users/josephsd/work/gabija/main/www/ user@server:/var/www/gabijja.app/
   
   # Or using scp
   scp -r css/ assets/ index.html user@server:/var/www/gabijja.app/
   ```

4. **Reload nginx** if needed:
   ```bash
   sudo systemctl reload nginx
   ```

### Monitoring

#### Log monitoring:
```bash
# View access logs
sudo tail -f /var/log/nginx/gabijja.app.access.log

# View error logs
sudo tail -f /var/log/nginx/gabijja.app.error.log

# Check nginx status
sudo systemctl status nginx
```

#### SSL certificate renewal:
```bash
# Check certificate status
sudo certbot certificates

# Test renewal (dry run)
sudo certbot renew --dry-run

# View renewal timer
sudo systemctl list-timers --all | grep certbot
```

## Troubleshooting

### Common Issues

#### 403 Forbidden Errors
```bash
# Check file permissions
ls -la /var/www/gabijja.app/

# Fix ownership (nginx user is 'http' on Arch)
sudo chown -R http:http /var/www/gabijja.app/
sudo chmod -R 755 /var/www/gabijja.app/
```

#### 502 Bad Gateway
```bash
# Check nginx status
sudo systemctl status nginx

# Check nginx logs
sudo journalctl -u nginx -f

# Test nginx configuration
sudo nginx -t
```

#### SSL Certificate Issues
```bash
# Check certificate expiration
sudo certbot certificates

# Renew certificates manually
sudo certbot renew

# Check certbot logs
sudo journalctl -u certbot -f
```

#### Static Assets Not Loading
```bash
# Check file paths
ls -la /var/www/gabijja.app/css/
ls -la /var/www/gabijja.app/assets/

# Test file access
sudo -u http cat /var/www/gabijja.app/css/style.css
```

### Performance Optimization

#### Enable browser caching:
```nginx
# Add to nginx config for static assets
location ~* \.(css|js|ico|png|jpg|jpeg|gif|svg|woff|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}
```

#### Compress responses:
```nginx
# Add to nginx config
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;
```

## Security Considerations

1. **Keep packages updated**: `sudo pacman -Syu`
2. **Use HTTPS only** (redirect HTTP to HTTPS)
3. **Implement security headers** (included in config)
4. **Regular backups** of nginx configuration
5. **Monitor access logs** for suspicious activity
6. **Firewall configuration** to allow only necessary ports

## Contact & Support

If you encounter issues during deployment:

1. Check nginx logs: `/var/log/nginx/gabijja.app.error.log`
2. Verify file permissions and ownership
3. Test SSL certificate status
4. Check domain DNS resolution
5. Ensure firewall allows traffic on ports 80 and 443

## Next Steps

After deploying the coming soon page:

1. Monitor website performance and uptime
2. Set up analytics if desired
3. Prepare for future Gabija application deployment
4. Document any customizations or modifications
5. Consider setting up automated deployment pipeline

---

**Note**: This deployment assumes you have an existing Arch Linux server with nginx and SSL already configured for gabijja.app. Adjust the steps as needed for your specific server environment.