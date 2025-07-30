# n8n Multi-Component Deployment Guide
## n8n/web server/cache layer/database

### Project Context
- **PROJECT_NAME**: n8n Production Deployment
- **PLATFORM**: Web/Docker/Self-Hosted
- **TECH_STACK**: Docker, Docker Compose, PostgreSQL, Redis, Nginx, n8n, Certbot
- **COMPLIANCE**: SSL/TLS, Security Headers, Database Encryption, JWT Authentication

---

## 🎯 Overview
This guide provides a complete, production-ready deployment of n8n with:
- n8n Community Edition (workflow automation platform)
- PostgreSQL (primary database)
- Redis (queue management)
- Nginx (reverse proxy with SSL)
- Certbot (Let's Encrypt SSL certificates)
- Automated backups and monitoring

---

## 📋 Pre-Deployment Checklist

### System Requirements
- [ ] Ubuntu 20.04+ or similar Linux distribution
- [ ] Docker 20.10+ installed
- [ ] Docker Compose 2.0+ installed
- [ ] Domain name pointing to your server
- [ ] Ports 80 and 443 open in firewall
- [ ] At least 2GB RAM (4GB recommended)
- [ ] At least 20GB disk space

### Prerequisites Installation
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installations
docker --version
docker-compose --version
```

---

## 🏗️ Step 1: Create Directory Structure

### Agent 1: Product & UX Agent - Directory Design
Create an organized, intuitive directory structure:

```bash
# Create main project directory
mkdir -p ~/n8n-production
cd ~/n8n-production

# Create all required subdirectories
mkdir -p nginx postgres-init certs n8n_data postgres_data backup certbot/{www,conf}

# Verify structure
tree -L 2
```

Expected structure:
```
n8n-production/
├── docker-compose.yml      # Main orchestration file
├── .env                    # Environment variables (sensitive data)
├── nginx/                  # Nginx configuration
│   └── default.conf       # Nginx server config
├── postgres-init/         # Database initialization
│   └── init-db.sh        # PostgreSQL user setup
├── certs/                # SSL certificates
├── n8n_data/            # n8n persistent data
├── postgres_data/       # PostgreSQL data
├── backup/              # Database backups
└── certbot/            # Let's Encrypt
    ├── www/           # Webroot for challenges
    └── conf/          # Certbot configuration
```

---

## 🔐 Step 2: Generate Security Keys & Configuration

### Agent 2: Engineering & Architecture Agent - Security Setup

#### Create Key Generation Script
```bash
cat > generate-keys.sh << 'EOF'
#!/bin/bash
# Generate secure keys for n8n deployment

echo "🔐 Generating secure keys for n8n..."
echo ""
echo "Copy these values to your .env file:"
echo "===================================="
echo ""
echo "# Generated Security Keys"
echo "N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)"
echo "N8N_JWT_SECRET=$(openssl rand -hex 32)"
echo ""
echo "# Generated Database Passwords"
echo "POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '=')"
echo "POSTGRES_NON_ROOT_PASSWORD=$(openssl rand -base64 32 | tr -d '=')"
echo ""
echo "===================================="
echo "⚠️  IMPORTANT: Save these keys securely! They cannot be changed after initial setup."
EOF

chmod +x generate-keys.sh
```

#### Run Key Generation
```bash
./generate-keys.sh
```

#### Create Environment File
Create `.env` file with your generated values:

```bash
cat > .env << 'EOF'
# ======================
# Domain Configuration
# ======================
N8N_HOST=your-domain.com  # CHANGE THIS to your actual domain
SUBDOMAIN=n8n
DOMAIN_EMAIL=admin@your-domain.com  # CHANGE THIS to your email

# ======================
# Security Keys (REQUIRED - Use generated values)
# ======================
# Database Passwords
POSTGRES_PASSWORD=PASTE_GENERATED_VALUE_HERE
POSTGRES_NON_ROOT_PASSWORD=PASTE_GENERATED_VALUE_HERE

# n8n Encryption Keys
N8N_ENCRYPTION_KEY=PASTE_GENERATED_VALUE_HERE
N8N_JWT_SECRET=PASTE_GENERATED_VALUE_HERE

# ======================
# Email Configuration (Optional but recommended)
# ======================
# Gmail Example:
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-specific-password
SMTP_SENDER=n8n Automation <your-email@gmail.com>

# For other providers:
# Office 365: smtp.office365.com:587
# SendGrid: smtp.sendgrid.net:587
# Amazon SES: email-smtp.region.amazonaws.com:587

# ======================
# System Configuration
# ======================
TIMEZONE=America/New_York  # Change to your timezone
EOF
```

⚠️ **CRITICAL**: Replace all placeholder values with your actual configuration!

---

## 🐳 Step 3: Create Docker Compose Configuration

### Agent 2: Engineering & Architecture Agent - Container Orchestration

Create `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # ======================
  # PostgreSQL Database
  # ======================
  postgres:
    image: postgres:15-alpine
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=n8n
      - POSTGRES_NON_ROOT_PASSWORD=${POSTGRES_NON_ROOT_PASSWORD}
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
      - ./postgres-init:/docker-entrypoint-initdb.d
      - ./backup:/backup
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -U n8n"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  # ======================
  # Redis for Queue Mode
  # ======================
  redis:
    image: redis:7-alpine
    container_name: n8n_redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  # ======================
  # n8n Application
  # ======================
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      # Database Configuration
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n_user
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_NON_ROOT_PASSWORD}
      
      # Redis Configuration
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - EXECUTIONS_MODE=queue
      
      # n8n Configuration
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/
      - N8N_EDITOR_BASE_URL=https://${N8N_HOST}/
      
      # Security
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_USER_MANAGEMENT_JWT_SECRET=${N8N_JWT_SECRET}
      
      # User Management
      - N8N_USER_MANAGEMENT_DISABLED=false
      - N8N_USER_MANAGEMENT_JWT_DURATION_HOURS=168
      
      # Email Configuration
      - N8N_EMAIL_MODE=smtp
      - N8N_SMTP_HOST=${SMTP_HOST}
      - N8N_SMTP_PORT=${SMTP_PORT}
      - N8N_SMTP_USER=${SMTP_USER}
      - N8N_SMTP_PASS=${SMTP_PASS}
      - N8N_SMTP_SENDER=${SMTP_SENDER}
      - N8N_SMTP_SSL=false
      
      # Execution Configuration
      - EXECUTIONS_PROCESS=main
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - EXECUTIONS_DATA_SAVE_ON_PROGRESS=true
      - EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
      - EXECUTIONS_DATA_MAX_AGE=336  # 14 days
      - EXECUTIONS_DATA_PRUNE_TIMEOUT=3600
      
      # Performance
      - N8N_CONCURRENCY_LIMIT=10
      - N8N_PAYLOAD_SIZE_MAX=16
      
      # Timezone
      - GENERIC_TIMEZONE=${TIMEZONE}
      - TZ=${TIMEZONE}
      
      # Diagnostics
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_PERSONALIZATION_ENABLED=false
      
      # Version Notifications
      - N8N_VERSION_NOTIFICATIONS_ENABLED=true
      
      # Logs
      - N8N_LOG_LEVEL=info
      - N8N_LOG_OUTPUT=console
    volumes:
      - ./n8n_data:/home/node/.n8n
      - ./backup:/backup:ro
    expose:
      - "5678"
    networks:
      - n8n-network

  # ======================
  # Nginx Reverse Proxy
  # ======================
  nginx:
    image: nginx:alpine
    container_name: n8n_nginx
    restart: unless-stopped
    depends_on:
      - n8n
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
      - ./certbot/www:/var/www/certbot:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    networks:
      - n8n-network

  # ======================
  # Certbot for SSL
  # ======================
  certbot:
    image: certbot/certbot
    container_name: n8n_certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  redis_data:

networks:
  n8n-network:
    driver: bridge
EOF
```

---

## 🌐 Step 4: Configure Nginx

### Agent 1: Product & UX Agent - Web Server Configuration

Create `nginx/default.conf`:

```bash
cat > nginx/default.conf << 'EOF'
# HTTP Server - Redirect to HTTPS
server {
    listen 80;
    server_name your-domain.com;  # CHANGE THIS
    
    # Let's Encrypt verification
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    # Redirect all other traffic to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    server_name your-domain.com;  # CHANGE THIS
    
    # SSL Configuration (self-signed initially)
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    
    # After Let's Encrypt setup, update to:
    # ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # SSL Security Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Proxy Settings
    client_max_body_size 16M;
    proxy_read_timeout 300s;
    proxy_connect_timeout 75s;
    
    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_buffering off;
    }
}
EOF
```

⚠️ **IMPORTANT**: Replace `your-domain.com` with your actual domain!

---

## 🗄️ Step 5: Configure Database

### Agent 2: Engineering & Architecture Agent - Database Setup

Create PostgreSQL initialization script:

```bash
cat > postgres-init/init-db.sh << 'EOF'
#!/bin/bash
set -e

echo "🗄️ Initializing n8n database..."

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    -- Create n8n user with limited privileges
    CREATE USER n8n_user WITH PASSWORD '$POSTGRES_NON_ROOT_PASSWORD';
    
    -- Grant necessary permissions
    GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n_user;
    
    -- Transfer ownership
    ALTER DATABASE n8n OWNER TO n8n_user;
    
    -- Log successful initialization
    \echo 'Database initialization complete!'
EOSQL
EOF

chmod +x postgres-init/init-db.sh
```

---

## 🔒 Step 6: Create Self-Signed Certificate (Initial Setup)

### Agent 3: DevOps & Quality Assurance Agent - SSL Setup

```bash
# Generate self-signed certificate for initial HTTPS
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
    -keyout certs/privkey.pem \
    -out certs/fullchain.pem \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=your-domain.com"

# Set proper permissions
chmod 644 certs/fullchain.pem
chmod 600 certs/privkey.pem
```

---

## 🚀 Step 7: Deploy n8n

### Agent 3: DevOps & Quality Assurance Agent - Deployment

#### Pre-deployment Checks
```bash
# Verify all files are in place
ls -la
ls -la nginx/
ls -la postgres-init/
ls -la certs/

# Check .env file
grep -E "N8N_HOST|POSTGRES_PASSWORD" .env

# Verify Docker is running
docker ps
```

#### Start Services
```bash
# Pull latest images
docker-compose pull

# Start services in detached mode
docker-compose up -d

# Monitor startup logs
docker-compose logs -f
```

#### Verify Deployment
```bash
# Check all containers are running
docker-compose ps

# Expected output:
# NAME             STATUS    PORTS
# n8n              running   5678/tcp
# n8n_nginx        running   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
# n8n_postgres     running   5432/tcp
# n8n_redis        running   6379/tcp
```

---

## 🎯 Step 8: Initial Setup & Admin Configuration

### Agent 1: Product & UX Agent - User Setup

1. **Access n8n Interface**
   ```
   https://your-domain.com
   ```
   Note: You'll see a certificate warning (self-signed). Accept and continue.

2. **Create Admin Account**
   - Enter admin email
   - Create strong password
   - Save credentials securely

3. **Verify Email Configuration**
   - Test email sending from Settings
   - Check SMTP configuration if needed

---

## 🔐 Step 9: Setup Let's Encrypt SSL (Production)

### Agent 3: DevOps & Quality Assurance Agent - SSL Automation

#### Initial Certificate Request
```bash
# Request certificate from Let's Encrypt
docker-compose run --rm certbot certonly \
    --webroot \
    --webroot-path=/var/www/certbot \
    --email admin@your-domain.com \
    --agree-tos \
    --no-eff-email \
    -d your-domain.com
```

#### Update Nginx Configuration
```bash
# Edit nginx/default.conf
# Update SSL certificate paths:
sed -i 's|ssl_certificate /etc/nginx/certs/fullchain.pem;|ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;|' nginx/default.conf
sed -i 's|ssl_certificate_key /etc/nginx/certs/privkey.pem;|ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;|' nginx/default.conf

# Reload Nginx
docker-compose exec nginx nginx -s reload
```

---

## 💾 Step 10: Setup Automated Backups

### Agent 3: DevOps & Quality Assurance Agent - Backup Automation

#### Create Backup Script
```bash
cat > backup-db.sh << 'EOF'
#!/bin/bash
# Automated n8n database backup script

BACKUP_DIR="./backup"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/n8n_backup_$TIMESTAMP.sql"

echo "🔄 Starting backup at $(date)"
echo "📁 Backup file: $BACKUP_FILE"

# Create backup
docker exec n8n_postgres pg_dump -U n8n n8n > "$BACKUP_FILE"

# Compress backup
gzip "$BACKUP_FILE"
echo "✅ Backup compressed: ${BACKUP_FILE}.gz"

# Keep only last 7 days of backups
find "$BACKUP_DIR" -name "n8n_backup_*.sql.gz" -mtime +7 -delete
echo "🗑️ Old backups cleaned up"

echo "✅ Backup completed successfully!"
EOF

chmod +x backup-db.sh
```

#### Schedule Automated Backups
```bash
# Add to crontab
(crontab -l 2>/dev/null; echo "0 2 * * * cd ~/n8n-production && ./backup-db.sh >> backup/backup.log 2>&1") | crontab -
```

---

## 🔍 Step 11: Monitoring & Health Checks

### Agent 3: DevOps & Quality Assurance Agent - Monitoring

#### Create Health Check Script
```bash
cat > health-check.sh << 'EOF'
#!/bin/bash
# n8n deployment health check

echo "🏥 n8n Health Check - $(date)"
echo "================================"

check_service() {
    if docker ps --format "table {{.Names}}" | grep -q "^$1$"; then
        echo "✅ $1 is running"
        return 0
    else
        echo "❌ $1 is DOWN!"
        return 1
    fi
}

# Check all services
SERVICES=("n8n" "n8n_postgres" "n8n_redis" "n8n_nginx")
FAILED=0

for service in "${SERVICES[@]}"; do
    if ! check_service "$service"; then
        FAILED=$((FAILED + 1))
    fi
done

echo "================================"

if [ $FAILED -eq 0 ]; then
    echo "✅ All services healthy!"
else
    echo "⚠️  $FAILED service(s) need attention!"
fi

# Check disk space
echo ""
echo "💾 Disk Usage:"
df -h | grep -E "^/dev/.*/$"

# Check memory
echo ""
echo "🧠 Memory Usage:"
free -h | grep -E "^Mem:"

# Check n8n workflows (optional)
echo ""
echo "📊 n8n Statistics:"
docker exec n8n_postgres psql -U n8n -d n8n -t -c "SELECT COUNT(*) as workflow_count FROM workflow_entity;" 2>/dev/null || echo "Unable to fetch workflow count"
EOF

chmod +x health-check.sh
```

#### Schedule Health Checks
```bash
# Add to crontab for hourly checks
(crontab -l 2>/dev/null; echo "0 * * * * cd ~/n8n-production && ./health-check.sh >> backup/health.log 2>&1") | crontab -
```

---

## 🛡️ Step 12: Security Hardening

### Agent 2: Engineering & Architecture Agent - Security

#### Firewall Configuration
```bash
# Allow only necessary ports
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

#### Resource Limits
Add to docker-compose.yml under n8n service:
```yaml
deploy:
  resources:
    limits:
      cpus: '2'
      memory: 4G
    reservations:
      cpus: '1'
      memory: 2G
```

---

## 📚 Maintenance Commands

### Daily Operations
```bash
# View logs
docker-compose logs -f n8n
docker-compose logs -f postgres

# Restart services
docker-compose restart n8n

# Update n8n
docker-compose pull n8n
docker-compose up -d n8n

# Check resource usage
docker stats

# Manual backup
./backup-db.sh
```

### Troubleshooting
```bash
# Container issues
docker-compose down
docker-compose up -d

# Database connection test
docker exec n8n_postgres psql -U n8n -d n8n -c "SELECT 1;"

# Redis connection test
docker exec n8n_redis redis-cli ping

# Clear old executions
docker exec n8n n8n executeBatch:clearExecutionData --days=30

# Restore from backup
gunzip < backup/n8n_backup_TIMESTAMP.sql.gz | docker exec -i n8n_postgres psql -U n8n n8n
```

---

## ⚠️ Common Issues & Solutions

### Issue: 502 Bad Gateway
**Solution**: n8n is still starting. Wait 30-60 seconds and refresh.

### Issue: Database Connection Failed
**Solution**: 
```bash
# Check postgres logs
docker-compose logs postgres
# Restart postgres
docker-compose restart postgres
```

### Issue: Workflows Not Executing
**Solution**: Check Redis connection
```bash
docker-compose logs redis
docker-compose restart redis
```

### Issue: High Memory Usage
**Solution**: Adjust execution retention
```bash
# Clear old executions
docker exec n8n n8n executeBatch:clearExecutionData --days=7
```

---

## 🎉 Success Criteria

✅ **All services running**: `docker-compose ps` shows all healthy  
✅ **Web interface accessible**: https://your-domain.com loads  
✅ **SSL certificate valid**: No browser warnings  
✅ **Admin account created**: Can login successfully  
✅ **Email working**: Test emails send successfully  
✅ **Backups automated**: Cron job creates daily backups  
✅ **Monitoring active**: Health checks run hourly  

---

## 📞 Support Resources

- **n8n Documentation**: https://docs.n8n.io
- **n8n Community Forum**: https://community.n8n.io
- **Docker Documentation**: https://docs.docker.com

---

## 🚀 Next Steps

1. **Import Workflows**: Start building automations
2. **Configure Integrations**: Connect your apps and services
3. **Set Up Webhooks**: Enable external triggers
4. **Create Users**: Add team members with appropriate permissions
5. **Monitor Performance**: Review logs and metrics regularly

---
