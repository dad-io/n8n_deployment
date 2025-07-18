# n8n: Multi-Component Docker Setup Guide

- n8n Community + Nginx + Redis + Postgres + Certbot (optional)
- Follow the guide - or ask your AI to
- Developed & tested with Claude Code / Opus 4
- Remember to update host references (e.g. N8N_HOST=your-domain.com -> localhost)
- NOTE/TODO: certbot setup currently untested 

## Expected Directory Structure
First, create this directory structure:
```
n8n-production/
├── docker-compose.yml
├── .env
├── nginx/
│   └── default.conf
├── postgres-init/
│   └── init-db.sh
├── certs/
├── n8n_data/
├── postgres_data/
├── backup/
└── certbot/
    ├── www/
    └── conf/
```

## 1. Environment Variables (.env)
Create `.env` file with your sensitive data:
```bash
# Domain Configuration
N8N_HOST=your-domain.com
SUBDOMAIN=n8n
DOMAIN_EMAIL=admin@your-domain.com

# Database Passwords (generate strong passwords)
POSTGRES_PASSWORD=CHANGE_ME_strong_password_123!
POSTGRES_NON_ROOT_PASSWORD=CHANGE_ME_different_password_456!

# n8n Security Keys (generate these!)
N8N_ENCRYPTION_KEY=CHANGE_ME_generate_with_openssl_rand_hex_32
N8N_JWT_SECRET=CHANGE_ME_generate_with_openssl_rand_hex_32

# SMTP Configuration (example for Gmail)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-specific-password
SMTP_SENDER=n8n Automation <your-email@gmail.com>

# Timezone
TIMEZONE=America/New_York
```

## 2. Docker Compose Configuration
`docker-compose.yml`:
```yaml
version: '3.8'

services:
  # PostgreSQL Database
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

  # Redis for queue mode (optional but recommended for production)
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

  # n8n Application
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
      
      # User Management (enabled for production)
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

  # Nginx Reverse Proxy
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

  # Certbot for Let's Encrypt (optional)
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
```

## 3. Nginx Configuration
`nginx/default.conf`:
```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name your-domain.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    server_name your-domain.com;

    # SSL Configuration (update paths based on your cert location)
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    
    # Or for Let's Encrypt:
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
```

## 4. Setup Scripts

### PostgreSQL User Initialization
`postgres-init/init-db.sh`:
```bash
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE USER n8n_user WITH PASSWORD '$POSTGRES_NON_ROOT_PASSWORD';
    GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n_user;
    ALTER DATABASE n8n OWNER TO n8n_user;
EOSQL
```

### Generate Security Keys
`generate-keys.sh`:
```bash
#!/bin/bash
echo "Generating secure keys for n8n..."
echo ""
echo "N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)"
echo "N8N_JWT_SECRET=$(openssl rand -hex 32)"
echo "POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '=')"
echo "POSTGRES_NON_ROOT_PASSWORD=$(openssl rand -base64 32 | tr -d '=')"
```

### Database Backup Script
`backup-db.sh`:
```bash
#!/bin/bash
BACKUP_DIR="./backup"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/n8n_backup_$TIMESTAMP.sql"

echo "Creating backup: $BACKUP_FILE"
docker exec n8n_postgres pg_dump -U n8n n8n > "$BACKUP_FILE"

# Compress backup
gzip "$BACKUP_FILE"

# Keep only last 7 days of backups
find "$BACKUP_DIR" -name "n8n_backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: ${BACKUP_FILE}.gz"
```

### Health Check Script
`health-check.sh`:
```bash
#!/bin/bash
# Add to crontab for monitoring

check_service() {
    if docker ps | grep -q $1; then
        echo "✓ $1 is running"
    else
        echo "✗ $1 is down!"
        # Send alert (email, webhook, etc.)
    fi
}

check_service "n8n"
check_service "n8n_postgres"
check_service "n8n_redis"
check_service "n8n_nginx"
```

## 5. Initial Setup Commands

```bash
# 1. Create directories
mkdir -p n8n-production/{nginx,certs,n8n_data,postgres_data,postgres-init,backup,certbot/{www,conf}}
cd n8n-production

# 2. Create PostgreSQL initialization script
mkdir -p postgres-init
cat > postgres-init/init-db.sh << 'EOF'
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE USER n8n_user WITH PASSWORD '$POSTGRES_NON_ROOT_PASSWORD';
    GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n_user;
    ALTER DATABASE n8n OWNER TO n8n_user;
EOSQL
EOF
chmod +x postgres-init/init-db.sh

# 3. Generate keys and update .env
chmod +x generate-keys.sh
./generate-keys.sh

# 4. Create self-signed cert (for initial setup)
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout certs/privkey.pem \
  -out certs/fullchain.pem \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=your-domain.com"

# 5. Start services
docker-compose up -d

# 6. Check logs
docker-compose logs -f

# 7. Initialize admin user (visit https://your-domain.com)
```

## 6. Let's Encrypt Setup (Production SSL)

```bash
# Initial certificate request
docker-compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email admin@your-domain.com \
  --agree-tos \
  --no-eff-email \
  -d your-domain.com

# Update nginx config to use Let's Encrypt certs
# Then reload nginx
docker-compose exec nginx nginx -s reload
```

## 7. Maintenance Commands

```bash
# View logs
docker-compose logs -f n8n

# Restart services
docker-compose restart

# Update n8n
docker-compose pull n8n
docker-compose up -d n8n

# Database backup
./backup-db.sh

# Database restore
docker exec -i n8n_postgres psql -U n8n n8n < backup/backup_file.sql

# Clean up old executions
docker exec n8n n8n executeBatch:clearExecutionData --days=30

# Monitor resources
docker stats
```

## 8. Security Checklist

- [ ] Strong passwords in .env file
- [ ] Encryption key generated and secured
- [ ] SSL/TLS configured (self-signed or Let's Encrypt)
- [ ] Firewall rules configured (only 80/443 open)
- [ ] Regular backups scheduled
- [ ] Monitoring/alerting configured
- [ ] User management enabled
- [ ] SMTP configured for notifications
- [ ] Resource limits set in docker-compose
- [ ] Log rotation configured

## 9. Performance Tuning

Add to n8n service in docker-compose.yml:
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

## 10. Troubleshooting

Common issues:
1. **502 Bad Gateway**: n8n not ready yet, check logs
2. **Database connection failed**: Check postgres health
   - If you see "Role n8n_user does not exist", ensure postgres-init script is properly mounted
   - May need to remove postgres_data folder and restart to reinitialize database
3. **Workflows not executing**: Check redis connection
4. **High memory usage**: Adjust execution data retention
5. **SSL errors**: Verify certificate paths and permissions
