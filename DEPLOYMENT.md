# Production Deployment Guide

## Deployment Options

Choose one of the following deployment strategies based on your infrastructure:

### Option 1: Deploy on VPS (DigitalOcean, Linode, Vultr, etc.)

#### Prerequisites
- Ubuntu 20.04+ or CentOS 8+ VPS
- SSH access
- Domain name (DNS configured)

#### Step 1: Initial Server Setup

```bash
# SSH into server
ssh root@your_server_ip

# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Add user to docker group
usermod -aG docker $USER
newgrp docker

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Install Certbot for SSL
apt install certbot python3-certbot-nginx -y
```

#### Step 2: Clone Repository

```bash
cd /home/ubuntu
git clone https://github.com/visiotechnology26-hash/School_admission_system.git
cd School_admission_system
```

#### Step 3: Setup Environment

```bash
# Create production environment file
cat > .env.prod << EOF
# Database
DB_NAME=admissions_system
DB_USER=admin
DB_PASSWORD=$(openssl rand -base64 32)

# Redis
REDIS_PASSWORD=$(openssl rand -base64 32)

# Flask
SECRET_KEY=$(openssl rand -base64 32)
JWT_SECRET_KEY=$(openssl rand -base64 32)
FLASK_ENV=production

# Frontend
REACT_APP_API_URL=https://yourdomain.com/api
NODE_ENV=production

# Registry (GitHub Container Registry)
REGISTRY=ghcr.io

# API URL for Nginx
API_URL=https://yourdomain.com/api
EOF
```

#### Step 4: Generate SSL Certificate

```bash
# Generate self-signed certificate or use Let's Encrypt
mkdir -p ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ssl/key.pem \
  -out ssl/cert.pem

# Or use Let's Encrypt (recommended)
certbot certonly --standalone -d yourdomain.com
# Copy certs to ssl/ directory
cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem ssl/cert.pem
cp /etc/letsencrypt/live/yourdomain.com/privkey.pem ssl/key.pem
```

#### Step 5: Deploy Application

```bash
# Use production compose file
docker-compose -f docker-compose.prod.yml up -d

# Check logs
docker-compose -f docker-compose.prod.yml logs -f

# Initialize database
docker-compose -f docker-compose.prod.yml exec backend flask db upgrade
```

#### Step 6: Setup Auto-Renewal for SSL (Let's Encrypt)

```bash
# Create renewal script
sudo tee /etc/letsencrypt/renewal-hooks/post/docker-reload.sh > /dev/null << 'EOF'
#!/bin/bash
cd /home/ubuntu/School_admission_system
cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem ssl/cert.pem
cp /etc/letsencrypt/live/yourdomain.com/privkey.pem ssl/key.pem
docker-compose -f docker-compose.prod.yml exec -T nginx nginx -s reload
EOF

sudo chmod +x /etc/letsencrypt/renewal-hooks/post/docker-reload.sh

# Test renewal
sudo certbot renew --dry-run
```

#### Step 7: Setup Auto-Restart on Reboot

```bash
# Create systemd service
sudo tee /etc/systemd/system/school-admissions.service > /dev/null << 'EOF'
[Unit]
Description=School Admissions System
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
User=ubuntu
WorkingDirectory=/home/ubuntu/School_admission_system
ExecStart=/usr/local/bin/docker-compose -f docker-compose.prod.yml up -d
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable school-admissions
sudo systemctl start school-admissions
```

---

### Option 2: Deploy on AWS

#### Using AWS Lightsail (Easiest)

```bash
# 1. Create Lightsail instance (Ubuntu 20.04)
# 2. SSH into instance
# 3. Follow VPS setup steps above
# 4. Use Elastic IP for static IP
# 5. Configure Route 53 for DNS
```

#### Using AWS ECS

```bash
# 1. Create ECR repositories
aws ecr create-repository --repository-name school-admissions/backend --region us-east-1
aws ecr create-repository --repository-name school-admissions/frontend --region us-east-1

# 2. Push images
docker tag school-admissions/backend:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/school-admissions/backend:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/school-admissions/backend:latest

# 3. Create ECS task definitions and services
# 4. Configure Application Load Balancer
# 5. Setup RDS for database
# 6. Setup ElastiCache for Redis
```

---

### Option 3: Deploy on Azure

#### Using Azure Container Apps

```bash
# 1. Create resource group
az group create -n admissions-rg -l eastus

# 2. Create container registry
az acr create -n admissionsacr -g admissions-rg --sku Basic

# 3. Push images
az acr build -r admissionsacr -t school-admissions/backend:latest ./backend
az acr build -r admissionsacr -t school-admissions/frontend:latest ./frontend

# 4. Create container apps environment
az containerapp env create -n admissions-env -g admissions-rg -l eastus

# 5. Create apps
az containerapp create -n backend -g admissions-rg \
  --environment admissions-env \
  --image admissionsacr.azurecr.io/school-admissions/backend:latest \
  --target-port 5000
```

---

### Option 4: Deploy on Heroku

```bash
# 1. Install Heroku CLI
curl https://cli.heroku.com/install.sh | sh

# 2. Login to Heroku
heroku login

# 3. Create app
heroku create school-admissions-system
heroku container:login

# 4. Push containers
heroku container:push web --app school-admissions-system
heroku container:release web --app school-admissions-system

# 5. Set environment variables
heroku config:set DB_NAME=admissions_system --app school-admissions-system
```

---

## Production Checklist

- [ ] Update all secrets in `.env.prod`
- [ ] Configure SSL/TLS certificates
- [ ] Setup domain name and DNS
- [ ] Configure firewall rules
- [ ] Enable automatic backups
- [ ] Setup monitoring and alerting
- [ ] Configure log aggregation
- [ ] Enable security headers in Nginx
- [ ] Setup rate limiting
- [ ] Configure CORS properly
- [ ] Setup database backups
- [ ] Configure email notifications
- [ ] Test disaster recovery procedures
- [ ] Document deployment process
- [ ] Setup CI/CD pipeline

## Monitoring & Maintenance

### Check Application Status

```bash
# View logs
docker-compose -f docker-compose.prod.yml logs backend
docker-compose -f docker-compose.prod.yml logs frontend

# Check container health
docker-compose -f docker-compose.prod.yml ps

# Monitor resources
docker stats
```

### Database Backups

```bash
# Manual backup
docker-compose -f docker-compose.prod.yml exec postgres pg_dump -U admin admissions_system > backup.sql

# Restore backup
docker-compose -f docker-compose.prod.yml exec -T postgres psql -U admin admissions_system < backup.sql
```

### Update Application

```bash
# Pull latest code
git pull origin main

# Rebuild and restart
docker-compose -f docker-compose.prod.yml pull
docker-compose -f docker-compose.prod.yml up -d

# Run migrations
docker-compose -f docker-compose.prod.yml exec backend flask db upgrade
```

### Scaling

```bash
# Scale backend services
docker-compose -f docker-compose.prod.yml up -d --scale backend=3

# Load balancing configured in Nginx
```

## Support

For deployment issues:
- Check logs: `docker-compose logs -f`
- Verify environment variables
- Check disk space: `df -h`
- Monitor CPU/Memory: `docker stats`
- Contact: support@admissionssystem.com

