
# FastAPI Application Deployment Guide

## VPS Deployment on AWS with PostgreSQL, Gunicorn, Nginx, and SSL

This guide provides step-by-step instructions for deploying a FastAPI application on an AWS VPS (Ubuntu) with PostgreSQL database, Gunicorn as the application server, Nginx as a reverse proxy, and Let's Encrypt SSL certification.

---

## Prerequisites

- AWS VPS instance running Ubuntu (22.04 or later recommended)
- Domain name pointed to your VPS IP address
- SSH access to your server
- sudo privileges

---

## 1. System Updates and Dependencies

Update system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

Install Python virtual environment support:

```bash
sudo apt install -y python3.12-venv
```

---

## 2. Application Setup

### 2.1 Create Virtual Environment

```bash
python3 -m venv venv
```

### 2.2 Activate Virtual Environment

```bash
source venv/bin/activate
```

### 2.3 Install Build Dependencies

```bash
sudo apt install -y libpq-dev python3-dev build-essential
```

### 2.4 Install Python Requirements

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

---

## 3. Environment Configuration

Create a `.env` file with your application configuration:

```bash
nano .env
```

Add your environment variables (example structure):

```env
DATABASE_URL=postgresql://postgres:YOUR_PASSWORD@localhost/cardlabs
SECRET_KEY=your-secret-key-here
DEBUG=False
```

**Security Note:** Never commit the `.env` file to version control. Add it to `.gitignore`.

---

## 4. Database Setup (PostgreSQL)

### 4.1 Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
```

### 4.2 Secure PostgreSQL

Access PostgreSQL prompt:

```bash
sudo -i -u postgres psql
```

Create database and set a secure password:

```sql
CREATE DATABASE cardlabs;
ALTER USER postgres PASSWORD 'your_secure_password_here';
\q
```

**Security Best Practice:** Use a strong password (minimum 16 characters with mixed case, numbers, and symbols).

---

## 5. Additional System Libraries

Install OpenCV dependencies:

```bash
sudo apt install -y libgl1 libglib2.0-0 libsm6 libxrender1 libxext6
```

---

## 6. Test Application with Gunicorn

Test your application manually before creating the service:

```bash
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000
```

### Troubleshooting Database Authentication

If you encounter authentication errors, reset the PostgreSQL password:

```bash
sudo -u postgres psql
```

```sql
ALTER USER postgres PASSWORD 'your_password';
\q
```

Update your `.env` file with the correct password.

---

## 7. Systemd Service Configuration

Create a systemd service to run your application in the background:

```bash
sudo nano /etc/systemd/system/fastapi.service
```

Add the following configuration:

```ini
[Unit]
Description=FastAPI Application
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=notify
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/cardlabsv3.0
Environment="PATH=/home/ubuntu/cardlabsv3.0/venv/bin"
ExecStart=/home/ubuntu/cardlabsv3.0/venv/bin/gunicorn \
    -w 4 \
    -k uvicorn.workers.UvicornWorker \
    main:app \
    --bind 127.0.0.1:8000 \
    --access-logfile /var/log/fastapi/access.log \
    --error-logfile /var/log/fastapi/error.log
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Create Log Directory

```bash
sudo mkdir -p /var/log/fastapi
sudo chown ubuntu:ubuntu /var/log/fastapi
```

### Enable and Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable fastapi
sudo systemctl start fastapi
sudo systemctl status fastapi
```

---

## 8. Nginx Reverse Proxy Setup

### 8.1 Install Nginx

```bash
sudo apt install -y nginx
```

### 8.2 Configure Nginx

Create a new site configuration:

```bash
sudo nano /etc/nginx/sites-available/fastapi
```

Add the following configuration:

```nginx
server {
    listen 80;
    server_name cardlabs-sandbox.duckdns.org www.cardlabs-sandbox.duckdns.org;

    client_max_body_size 20M;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        
        # WebSocket support (if needed)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 8.3 Enable Site and Test Configuration

```bash
sudo ln -s /etc/nginx/sites-available/fastapi /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 9. SSL Certificate with Let's Encrypt

### 9.1 Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 9.2 Generate SSL Certificate

```bash
sudo certbot --nginx -d cardlabs-sandbox.duckdns.org -d www.cardlabs-sandbox.duckdns.org
```

Follow the prompts:
1. Enter your email address
2. Agree to Terms of Service (Y)
3. Choose whether to share your email (Y/N)
4. Select option 2 to redirect HTTP to HTTPS

### 9.3 Verify Auto-Renewal

```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

---

## 10. Firewall Configuration (Optional but Recommended)

Configure UFW firewall:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

---

## 11. Useful Management Commands

### Application Service

```bash
# Restart application
sudo systemctl restart fastapi

# View logs
sudo journalctl -u fastapi -f

# Stop application
sudo systemctl stop fastapi
```

### Nginx

```bash
# Restart Nginx
sudo systemctl restart nginx

# Check configuration
sudo nginx -t

# View error logs
sudo tail -f /var/log/nginx/error.log
```

### PostgreSQL

```bash
# Access database
sudo -u postgres psql -d cardlabs

# Backup database
sudo -u postgres pg_dump cardlabs > backup.sql

# Restore database
sudo -u postgres psql cardlabs < backup.sql
```

---

## 12. Security Recommendations

1. **Use environment variables** for all sensitive configuration
2. **Set strong passwords** for database users (minimum 16 characters)
3. **Enable firewall** (UFW) and only allow necessary ports
4. **Keep system updated** with regular `apt update && apt upgrade`
5. **Monitor logs** regularly for suspicious activity
6. **Set up automated backups** for your database
7. **Use SSH key authentication** instead of passwords
8. **Disable root login** via SSH
9. **Consider fail2ban** to prevent brute force attacks
10. **Remove sensitive tokens** from documentation (GitHub tokens, API keys)

---

## Troubleshooting

### Application won't start

```bash
sudo journalctl -u fastapi -n 50
```

### Database connection issues

```bash
sudo systemctl status postgresql
sudo -u postgres psql -d cardlabs
```

### Nginx errors

```bash
sudo nginx -t
sudo tail -f /var/log/nginx/error.log
```

---

## Notes

- Replace `cardlabs-sandbox.duckdns.org` with your actual domain
- Adjust worker count (`-w 4`) based on your server's CPU cores (recommendation: 2-4 × CPU cores)
- Update file paths if your application is located elsewhere
- SSL certificates auto-renew via systemd timer

---

**Last Updated:** February 2026  
**Tested On:** Ubuntu 24.04 LTS
