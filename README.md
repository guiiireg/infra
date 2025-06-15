# 📄 Documentation – Advanced Web Server Configuration with NGINX

## Table of Contents
1. [Initial System Setup](#1-initial-system-setup)
2. [VM Network Configuration](#2-vm-network-configuration)
3. [NGINX Installation](#3-nginx-installation)
4. [VM Architecture Setup](#4-vm-architecture-setup)
5. [Application Deployment](#5-application-deployment)
6. [NGINX Reverse Proxy Configuration](#6-nginx-reverse-proxy-configuration)
7. [SSL/TLS Certificate Management](#7-ssltls-certificate-management)
8. [Subdomain Configuration](#8-subdomain-configuration)
9. [Testing and Verification](#9-testing-and-verification)
10. [Adding New Applications](#10-adding-new-applications)
11. [Optional: Load Balancing](#11-optional-load-balancing)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Initial System Setup

### 📌 1.1 Keyboard Configuration

#### 🧰 Command:
```bash
sudo dpkg-reconfigure keyboard-configuration
```

#### 📝 Explanation:
This command launches an interactive tool to select the keyboard type and desired layout. Follow the on-screen steps to choose:
- The keyboard model (e.g., "Generic 105-key (Intl) PC")
- The country and layout (e.g., "French")
- Special keys (compose, Alt key, etc.)

### 📌 1.2 System Update and Upgrade

#### 🧰 Command:
```bash
sudo apt update && sudo apt upgrade -y
```

#### 📝 Explanation:
- `apt update`: Updates the list of available packages
- `apt upgrade -y`: Installs the latest available versions of the packages already installed, without manual confirmation due to `-y`

---

## 2. VM Network Configuration (VirtualBox)

### 🎯 Objective:
Access your web server from your host machine.

### 🔧 NAT (default)
The VM is isolated, but you can access it via port forwarding.

### 🔄 Steps:
1. Shut down the VM if it is running
2. Go to VirtualBox > VM Settings > Network > Advanced > Port Forwarding
3. Add rules as follows:

| Protocol | Host IP   | Host Port | Guest IP | Guest Port | Description |
|----------|-----------|-----------|----------|------------|-------------|
| TCP      | 127.0.0.1 | 8080      |          | 80         | HTTP        |
| TCP      | 127.0.0.1 | 8443      |          | 443        | HTTPS       |

💡 **Example**: Access the NGINX server via `http://127.0.0.1:8080` on your host machine.

---

## 3. NGINX Installation

### 📌 3.1 Remove Apache (if installed)

#### 🧰 Commands:
```bash
sudo systemctl stop apache2
sudo apt purge apache2 apache2-bin apache2-data apache2-utils -y
sudo apt autoremove --purge apache2 -y
sudo rm -rf /etc/apache2
```

#### 📝 Explanation:
- Stops Apache service completely
- Removes all Apache packages and configuration files
- Clears any remaining dependencies
- Removes configuration directory

### 📌 3.2 Install NGINX

#### 🧰 Commands:
```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

#### 📝 Explanation:
- `apt install nginx -y`: Installs the NGINX web server
- `systemctl start nginx`: Starts the NGINX service immediately
- `systemctl enable nginx`: Enables NGINX to start automatically at each VM boot
- `systemctl status nginx`: Verifies that NGINX is running properly

### 📌 3.3 Configure Firewall

#### 🧰 Commands:
```bash
sudo ufw allow 'Nginx Full'
sudo ufw status
```

#### 📝 Explanation:
- Opens both HTTP (port 80) and HTTPS (port 443) ports
- Verifies firewall status

---

## 4. VM Architecture Setup

### 🏗️ Architecture Overview

This setup uses two VMs:

1. **VM "Applications"** (IP: 10.0.0.2)
   - Hosts all web applications
   - Each application runs on different internal ports
   - Not directly accessible from the internet

2. **VM "Reverse Proxy"** (IP: 10.0.0.3)
   - Runs NGINX as reverse proxy
   - Manages SSL certificates
   - Routes traffic to appropriate applications
   - Only VM exposed to the internet

### 📌 4.1 VM Network Configuration

#### For Applications VM:
```bash
# Set static IP (example for Ubuntu/Debian)
sudo nano /etc/netplan/01-network-manager-all.yaml
```

Example configuration:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.2/24]
      gateway4: 10.0.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Apply configuration:
```bash
sudo netplan apply
```

---

## 5. Application Deployment

### 📌 5.1 WordPress (PHP) - Port 8081

#### 🧰 Installation Commands:
```bash
# Install PHP and dependencies
sudo apt install php-fpm php-mysql php-curl php-gd php-xml php-mbstring -y

# Download WordPress
cd /var/www/
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo chown -R www-data:www-data wordpress/

# Configure PHP-FPM to listen on port 8081
sudo nano /etc/php/*/fpm/pool.d/wordpress.conf
```

WordPress pool configuration:
```ini
[wordpress]
user = www-data
group = www-data
listen = 127.0.0.1:8081
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

Restart PHP-FPM:
```bash
sudo systemctl restart php*-fpm
```

### 📌 5.2 Wekan (Node.js) - Port 8082

#### 🧰 Installation Commands:
```bash
# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y

# Install MongoDB
sudo apt install mongodb -y
sudo systemctl start mongodb
sudo systemctl enable mongodb

# Install Wekan
sudo snap install wekan
```

Configure Wekan to run on port 8082:
```bash
sudo snap set wekan port=8082
sudo snap restart wekan
```

### 📌 5.3 Blogango (Python) - Port 8083

#### 🧰 Installation Commands:
```bash
# Install Python and pip
sudo apt install python3 python3-pip python3-venv -y

# Create virtual environment
cd /opt/
sudo mkdir blogango
cd blogango
sudo python3 -m venv venv
sudo ./venv/bin/pip install django

# Create Django project
sudo ./venv/bin/django-admin startproject blogango .

# Configure to run on port 8083
sudo nano blogango/settings.py
```

Add to settings.py:
```python
ALLOWED_HOSTS = ['*']
```

Create systemd service:
```bash
sudo nano /etc/systemd/system/blogango.service
```

Service configuration:
```ini
[Unit]
Description=Blogango Django Application
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/blogango
ExecStart=/opt/blogango/venv/bin/python manage.py runserver 0.0.0.0:8083
Restart=always

[Install]
WantedBy=multi-user.target
```

Start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl start blogango
sudo systemctl enable blogango
```

### 📌 5.4 Verify Applications

#### 🧰 Test Commands:
```bash
# Test each application locally
curl http://127.0.0.1:8081  # WordPress
curl http://127.0.0.1:8082  # Wekan
curl http://127.0.0.1:8083  # Blogango
```

---

## 6. NGINX Reverse Proxy Configuration

### 📌 6.1 Create SSL Directory

#### 🧰 Command:
```bash
sudo mkdir -p /etc/nginx/ssl
```

### 📌 6.2 Basic HTTP Configuration

Create configuration files for each subdomain:

#### For main domain (b1.com):
```bash
sudo nano /etc/nginx/sites-available/b1.com
```

Configuration:
```nginx
server {
    listen 80;
    server_name b1.com;

    location / {
        proxy_pass http://10.0.0.2:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### For blog subdomain (blog.b1.com):
```bash
sudo nano /etc/nginx/sites-available/blog.b1.com
```

Configuration:
```nginx
server {
    listen 80;
    server_name blog.b1.com;

    location / {
        proxy_pass http://10.0.0.2:8083;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### For Wekan subdomain (wekan.b1.com):
```bash
sudo nano /etc/nginx/sites-available/wekan.b1.com
```

Configuration:
```nginx
server {
    listen 80;
    server_name wekan.b1.com;

    location / {
        proxy_pass http://10.0.0.2:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 📌 6.3 Enable Sites

#### 🧰 Commands:
```bash
sudo ln -s /etc/nginx/sites-available/b1.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/blog.b1.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/wekan.b1.com /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload NGINX
sudo systemctl reload nginx
```

---

## 7. SSL/TLS Certificate Management

### 📌 7.1 Generate Self-Signed Certificates

#### For main domain (b1.com):
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/b1.com.key -out /etc/nginx/ssl/b1.com.crt
```

#### For blog subdomain:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/blog.b1.com.key -out /etc/nginx/ssl/blog.b1.com.crt
```

#### For Wekan subdomain:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/wekan.b1.com.key -out /etc/nginx/ssl/wekan.b1.com.crt
```

### 📌 7.2 Update NGINX Configuration for HTTPS

#### Update b1.com configuration:
```bash
sudo nano /etc/nginx/sites-available/b1.com
```

Complete HTTPS configuration:
```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name b1.com;
    return 301 https://$host$request_uri;
}

# HTTPS configuration
server {
    listen 443 ssl;
    server_name b1.com;

    ssl_certificate /etc/nginx/ssl/b1.com.crt;
    ssl_certificate_key /etc/nginx/ssl/b1.com.key;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://10.0.0.2:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Apply similar configuration to other subdomains, adjusting:
- Server name
- SSL certificate paths
- Proxy pass destination ports

---

## 8. Subdomain Configuration

### 📌 8.1 DNS Configuration

For production, configure DNS records with your domain provider:

| Type  | Name   | Value                    |
|-------|--------|--------------------------|
| A     | b1.com | [Your VM Public IP]      |
| CNAME | blog   | b1.com                   |
| CNAME | wekan  | b1.com                   |

### 📌 8.2 Local Testing with Hosts File

For local testing, modify the hosts file on your client machine:

#### Linux/Mac:
```bash
sudo nano /etc/hosts
```

#### Windows:
```
C:\Windows\System32\drivers\etc\hosts
```

Add entries:
```
127.0.0.1    b1.com
127.0.0.1    blog.b1.com
127.0.0.1    wekan.b1.com
```

---

## 9. Testing and Verification

### 📌 9.1 Configuration Testing

#### 🧰 Commands:
```bash
# Test NGINX configuration
sudo nginx -t

# Check NGINX status
sudo systemctl status nginx

# Check listening ports
sudo lsof -i :80
sudo lsof -i :443

# Check firewall status
sudo ufw status
```

### 📌 9.2 Application Testing

#### HTTP Testing:
```bash
curl -L http://b1.com
curl -L http://blog.b1.com
curl -L http://wekan.b1.com
```

#### HTTPS Testing:
```bash
curl -Lk https://b1.com
curl -Lk https://blog.b1.com
curl -Lk https://wekan.b1.com
```

### 📌 9.3 Browser Testing

1. Open browser and navigate to:
   - `https://b1.com` (should show WordPress)
   - `https://blog.b1.com` (should show Blogango)
   - `https://wekan.b1.com` (should show Wekan)

2. Accept self-signed certificate warnings (click "Advanced" → "Continue")

3. Verify certificate details by clicking the lock icon in the address bar

---

## 10. Adding New Applications

### 📌 10.1 Step-by-Step Process

#### 🔄 Steps:
1. **Deploy application on Applications VM**
   - Choose an available port (e.g., 8084)
   - Configure application to listen on chosen port
   - Test local access: `curl http://127.0.0.1:8084`

2. **Generate SSL certificate**
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /etc/nginx/ssl/newapp.b1.com.key \
     -out /etc/nginx/ssl/newapp.b1.com.crt
   ```

3. **Create NGINX configuration**
   ```bash
   sudo nano /etc/nginx/sites-available/newapp.b1.com
   ```

   Configuration template:
   ```nginx
   server {
       listen 80;
       server_name newapp.b1.com;
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl;
       server_name newapp.b1.com;

       ssl_certificate /etc/nginx/ssl/newapp.b1.com.crt;
       ssl_certificate_key /etc/nginx/ssl/newapp.b1.com.key;

       location / {
           proxy_pass http://10.0.0.2:8084;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

4. **Enable the site**
   ```bash
   sudo ln -s /etc/nginx/sites-available/newapp.b1.com /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

5. **Update DNS/Hosts file**
   - Add CNAME record: `newapp.b1.com → b1.com`
   - Or add to hosts file: `127.0.0.1 newapp.b1.com`

6. **Test the new application**
   ```bash
   curl -Lk https://newapp.b1.com
   ```

---

## 11. Optional: Load Balancing

### 📌 11.1 Multiple Application Instances

For high availability, deploy multiple instances of the same application:

#### Example with WordPress:
```nginx
upstream wordpress_backend {
    server 10.0.0.2:8081;
    server 10.0.0.2:8084;  # Second WordPress instance
    server 10.0.0.2:8085;  # Third WordPress instance
}

server {
    listen 443 ssl;
    server_name b1.com;

    ssl_certificate /etc/nginx/ssl/b1.com.crt;
    ssl_certificate_key /etc/nginx/ssl/b1.com.key;

    location / {
        proxy_pass http://wordpress_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 📌 11.2 Load Balancing Methods

Add to upstream block:
```nginx
upstream wordpress_backend {
    # Round-robin (default)
    server 10.0.0.2:8081;
    server 10.0.0.2:8084;
    
    # Weighted round-robin
    # server 10.0.0.2:8081 weight=3;
    # server 10.0.0.2:8084 weight=1;
    
    # IP hash (session persistence)
    # ip_hash;
    
    # Least connections
    # least_conn;
}
```

---

## 12. Troubleshooting

### 📌 12.1 Common Issues and Solutions

#### 🔧 "Failed to connect to domain port 443"
**Causes:**
- Domain not pointing to your server
- Port 443 blocked by firewall
- NGINX not listening on 443

**Solutions:**
```bash
# Check firewall
sudo ufw status

# Check listening ports
sudo lsof -i :443

# Check NGINX configuration
sudo nginx -t

# Check hosts file (for local testing)
cat /etc/hosts | grep b1.com
```

#### 🔧 "502 Bad Gateway"
**Causes:**
- Backend application not running
- Wrong proxy_pass URL/port
- Network connectivity issues

**Solutions:**
```bash
# Check application status
curl http://10.0.0.2:8081  # Test backend directly

# Check NGINX error logs
sudo tail -f /var/log/nginx/error.log

# Verify proxy_pass configuration
sudo nginx -T | grep proxy_pass
```

#### 🔧 SSL Certificate Warnings
**Causes:**
- Self-signed certificates (expected)
- Certificate name mismatch
- Expired certificates

**Solutions:**
```bash
# Check certificate details
openssl x509 -in /etc/nginx/ssl/b1.com.crt -text -noout

# Verify certificate matches domain
openssl x509 -in /etc/nginx/ssl/b1.com.crt -noout -subject
```

### 📌 12.2 Useful Debugging Commands

#### 🧰 Log Monitoring:
```bash
# NGINX access logs
sudo tail -f /var/log/nginx/access.log

# NGINX error logs
sudo tail -f /var/log/nginx/error.log

# System logs
sudo journalctl -u nginx -f
```

#### 🧰 Network Debugging:
```bash
# Test connectivity to applications VM
ping 10.0.0.2

# Check port accessibility
telnet 10.0.0.2 8081

# Monitor network traffic
sudo tcpdump -i any port 443
```

#### 🧰 Configuration Verification:
```bash
# Show complete NGINX configuration
sudo nginx -T

# Test configuration syntax
sudo nginx -t

# Check active sites
ls -la /etc/nginx/sites-enabled/
```
