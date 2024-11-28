# Deployment Guide for VPS

## Prerequisites

- Node.js 18+
- PostgreSQL 12+
- Nginx (for reverse proxy)
- PM2 (for process management)
- Ubuntu 20.04 or newer

## Initial Server Setup

1. Update system and install essential packages:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential postgresql nginx
```

2. Install Node.js 18:
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

3. Install PM2 globally:
```bash
sudo npm install -g pm2
```

## Database Setup

1. Configure PostgreSQL:
```bash
sudo -u postgres psql
```

2. Create database and user:
```sql
CREATE DATABASE subscription_manager;
CREATE USER subscription_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE subscription_manager TO subscription_user;
\c subscription_manager
GRANT ALL ON ALL TABLES IN SCHEMA public TO subscription_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO subscription_user;
\q
```

3. Configure PostgreSQL for remote connections (if needed):
```bash
sudo nano /etc/postgresql/12/main/postgresql.conf
```
Update:
```
listen_addresses = '*'
```

4. Configure client authentication:
```bash
sudo nano /etc/postgresql/12/main/pg_hba.conf
```
Add:
```
host    subscription_manager    subscription_user    0.0.0.0/0    md5
```

5. Restart PostgreSQL:
```bash
sudo systemctl restart postgresql
```

## Application Setup

1. Create application directory:
```bash
sudo mkdir -p /var/www/subscription-manager
sudo chown $USER:$USER /var/www/subscription-manager
```

2. Clone and setup application:
```bash
cd /var/www/subscription-manager
git clone <repository-url> .
npm install
```

3. Create production environment file:
```bash
cp .env.example .env
nano .env
```

Update with production values:
```env
NODE_ENV=production
PORT=3000

# Database
DB_USER=subscription_user
DB_HOST=localhost
DB_NAME=subscription_manager
DB_PASSWORD=your_secure_password
DB_PORT=5432

# JWT
JWT_SECRET=your-secure-random-string

# Frontend
VITE_API_URL=/api
```

4. Build the application:
```bash
npm run build
```

## Nginx Configuration

1. Create Nginx configuration:
```bash
sudo nano /etc/nginx/sites-available/subscription-manager
```

2. Add configuration:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Increase max body size for file uploads
    client_max_body_size 10M;

    # Enable gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

3. Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/subscription-manager /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## PM2 Setup

1. Create PM2 ecosystem file:
```bash
nano ecosystem.config.js
```

2. Add configuration:
```javascript
module.exports = {
  apps: [{
    name: 'subscription-manager',
    script: 'server.js',
    instances: 'max',
    exec_mode: 'cluster',
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    }
  }]
}
```

3. Start the application:
```bash
# Run database migrations
npm run migrate

# Start with PM2
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

## SSL Configuration (Let's Encrypt)

1. Install Certbot:
```bash
sudo apt install -y certbot python3-certbot-nginx
```

2. Obtain SSL certificate:
```bash
sudo certbot --nginx -d your-domain.com
```

## Security Measures

1. Configure firewall:
```bash
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

2. Set up automatic security updates:
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

3. Secure PostgreSQL:
```bash
sudo nano /etc/postgresql/12/main/postgresql.conf
```
Add:
```
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
```

## Monitoring

1. Setup PM2 monitoring:
```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
```

2. Monitor application:
```bash
pm2 monit
```

3. View logs:
```bash
pm2 logs subscription-manager
```

## Backup Strategy

1. Setup automated database backups:
```bash
sudo nano /etc/cron.daily/backup-subscription-manager
```

Add:
```bash
#!/bin/bash
BACKUP_DIR="/var/backups/subscription-manager"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# Database backup
pg_dump subscription_manager > $BACKUP_DIR/db_backup_$TIMESTAMP.sql

# Compress backup
gzip $BACKUP_DIR/db_backup_$TIMESTAMP.sql

# Keep only last 7 days of backups
find $BACKUP_DIR -type f -mtime +7 -delete
```

Make executable:
```bash
sudo chmod +x /etc/cron.daily/backup-subscription-manager
```

## Maintenance

1. Regular updates:
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Update Node.js dependencies
npm audit
npm update

# Rebuild application
npm run build

# Restart application
pm2 restart subscription-manager
```

2. Monitor disk space:
```bash
df -h
```

3. Monitor system resources:
```bash
htop
```

4. Check logs regularly:
```bash
tail -f /var/log/nginx/error.log
pm2 logs
```