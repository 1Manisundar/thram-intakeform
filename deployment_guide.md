# Deployment Guide - Production Setup

## Overview
This guide covers deploying the Insurance Verification System to AWS production environment.

**Target Infrastructure:**
- Frontend: AWS S3 + CloudFront
- Backend: AWS EC2
- Database: AWS RDS PostgreSQL
- n8n: AWS EC2 (separate instance)
- File Storage: AWS S3
- Email: AWS SES

---

## Prerequisites

### AWS Account Setup
- [ ] AWS account created
- [ ] AWS CLI installed and configured
- [ ] IAM user with appropriate permissions
- [ ] Domain name registered (optional but recommended)

### Required Tools
- AWS CLI v2
- SSH client
- Git
- Node.js 18+

---

## Phase 1: Database Setup (AWS RDS)

### 1.1 Create RDS PostgreSQL Instance

**Via AWS Console:**
1. Go to RDS Console
2. Click "Create database"
3. Select **PostgreSQL 15.x**
4. Choose **Production** template

**Configuration:**
```
DB Instance Identifier: insurance-verification-db
Master Username: dbadmin
Master Password: [Generate strong password]
DB Instance Class: db.t3.micro (for MVP) or db.t3.small
Storage: 20GB General Purpose SSD (gp3)
Multi-AZ: No (for MVP) / Yes (for production)
VPC: Create new or use existing
Public Access: No
VPC Security Group: Create new
```

**Security Group Rules:**
```
Inbound:
- Type: PostgreSQL
- Protocol: TCP
- Port: 5432
- Source: Backend EC2 security group
```

### 1.2 Connect and Setup Database

```bash
# Get RDS endpoint from AWS Console
# Example: insurance-verification-db.xxxxxx.us-east-1.rds.amazonaws.com

# Connect via psql
psql -h insurance-verification-db.xxxxxx.us-east-1.rds.amazonaws.com \
     -U dbadmin \
     -d postgres

# Create application database
CREATE DATABASE insurance_verification;

# Create application user
CREATE USER app_user WITH PASSWORD 'secure_password_here';
GRANT ALL PRIVILEGES ON DATABASE insurance_verification TO app_user;
```

**Save Connection String:**
```env
DATABASE_URL="postgresql://app_user:secure_password_here@insurance-verification-db.xxxxxx.us-east-1.rds.amazonaws.com:5432/insurance_verification?sslmode=require"
```

---

## Phase 2: Backend Deployment (AWS EC2)

### 2.1 Launch EC2 Instance

**Instance Configuration:**
```
AMI: Ubuntu 22.04 LTS
Instance Type: t3.small (2 vCPU, 2GB RAM)
Key Pair: Create new or use existing
Network: Same VPC as RDS
Auto-assign Public IP: Yes
Storage: 30GB gp3
```

**Security Group (Backend):**
```
Inbound Rules:
- SSH: Port 22 (Your IP only)
- HTTP: Port 80 (0.0.0.0/0)
- HTTPS: Port 443 (0.0.0.0/0)
- Custom TCP: Port 3000 (CloudFront or 0.0.0.0/0 temporarily)

Outbound Rules:
- All traffic (0.0.0.0/0)
```

### 2.2 Connect to EC2 Instance

```bash
# SSH into instance
ssh -i your-key.pem ubuntu@your-ec2-public-ip

# Update system
sudo apt update && sudo apt upgrade -y
```

### 2.3 Install Node.js

```bash
# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node --version
npm --version
```

### 2.4 Install Nginx

```bash
# Install Nginx
sudo apt install nginx -y

# Start Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 2.5 Deploy Backend Application

```bash
# Create app directory
sudo mkdir -p /var/www/insurance-backend
sudo chown -R ubuntu:ubuntu /var/www/insurance-backend

# Clone repository
cd /var/www/insurance-backend
git clone <your-repo-url> .

# Install dependencies
npm install

# Setup environment variables
nano .env
```

**Production .env:**
```env
# Database
DATABASE_URL="postgresql://app_user:password@rds-endpoint:5432/insurance_verification?sslmode=require"

# JWT
JWT_SECRET="generate-strong-secret-here"
JWT_EXPIRES_IN="7d"

# Server
PORT=3000
NODE_ENV="production"

# n8n
N8N_WEBHOOK_URL="http://n8n-private-ip:5678/webhook/insurance-verification"

# AWS S3
AWS_ACCESS_KEY_ID="your-access-key"
AWS_SECRET_ACCESS_KEY="your-secret-key"
AWS_S3_BUCKET="insurance-verification-files"
AWS_REGION="us-east-1"

# AWS SES
AWS_SES_REGION="us-east-1"
EMAIL_FROM="noreply@yourdomain.com"

# CORS
FRONTEND_URL="https://yourdomain.com"

# Webhook Secret
WEBHOOK_SECRET="generate-random-secret"
```

```bash
# Run Prisma migrations
npx prisma migrate deploy
npx prisma generate

# Build application
npm run build

# Test run
npm start
```

### 2.6 Setup PM2 (Process Manager)

```bash
# Install PM2 globally
sudo npm install -g pm2

# Start application with PM2
pm2 start dist/server.js --name insurance-backend

# Save PM2 configuration
pm2 save

# Setup PM2 startup script
pm2 startup
# Follow the command output to complete setup

# Check status
pm2 status
pm2 logs insurance-backend
```

### 2.7 Configure Nginx

```bash
# Create Nginx config
sudo nano /etc/nginx/sites-available/insurance-backend
```

**Nginx Configuration:**
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;  # Or your domain

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

    location / {
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # File upload size
    client_max_body_size 10M;
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/insurance-backend /etc/nginx/sites-enabled/

# Test Nginx config
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### 2.8 Setup SSL with Let's Encrypt

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get SSL certificate
sudo certbot --nginx -d api.yourdomain.com

# Test auto-renewal
sudo certbot renew --dry-run
```

---

## Phase 3: n8n Deployment (AWS EC2)

### 3.1 Launch EC2 Instance for n8n

**Instance Configuration:**
```
AMI: Ubuntu 22.04 LTS
Instance Type: t3.small
Key Pair: Same as backend
Network: Same VPC (private subnet recommended)
Auto-assign Public IP: No (use private IP)
Storage: 20GB gp3
```

**Security Group (n8n):**
```
Inbound Rules:
- SSH: Port 22 (Your IP only)
- Custom TCP: Port 5678 (Backend EC2 security group only)

Outbound Rules:
- All traffic (0.0.0.0/0)
```

### 3.2 Install Docker

```bash
# SSH into n8n instance
ssh -i your-key.pem ubuntu@n8n-private-ip

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker ubuntu

# Logout and login again
exit
ssh -i your-key.pem ubuntu@n8n-private-ip

# Install Docker Compose
sudo apt install docker-compose -y

# Verify
docker --version
docker-compose --version
```

### 3.3 Deploy n8n with Docker

```bash
# Create n8n directory
mkdir ~/n8n
cd ~/n8n

# Create docker-compose.yml
nano docker-compose.yml
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://${N8N_HOST}:5678/
      - GENERIC_TIMEZONE=America/New_York
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=${DB_HOST}
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${DB_NAME}
      - DB_POSTGRESDB_USER=${DB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

**Create .env file:**
```bash
nano .env
```

```env
N8N_PASSWORD=secure-admin-password
N8N_HOST=private-ip-of-n8n-instance
DB_HOST=rds-endpoint
DB_NAME=n8n
DB_USER=dbadmin
DB_PASSWORD=rds-password
```

**Start n8n:**
```bash
# Create n8n database in RDS first
psql -h rds-endpoint -U dbadmin -d postgres
CREATE DATABASE n8n;
\q

# Start n8n
docker-compose up -d

# Check logs
docker-compose logs -f

# Access n8n (from backend instance or via SSH tunnel)
# http://n8n-private-ip:5678
```

### 3.4 Import Workflow

1. Access n8n UI (via SSH tunnel if private)
2. Import `insurance-verification-workflow.json`
3. Update credentials (Mindee API, Stedi API)
4. Update PostgreSQL connection (use RDS endpoint)
5. Activate workflow

---

## Phase 4: Frontend Deployment (S3 + CloudFront)

### 4.1 Build Angular App

```bash
# On local machine
cd frontend/insurance-verification-frontend

# Update environment for production
nano src/environments/environment.prod.ts
```

```typescript
export const environment = {
  production: true,
  apiUrl: 'https://api.yourdomain.com/api'
};
```

```bash
# Build for production
ng build --configuration production

# Output will be in dist/ folder
```

### 4.2 Create S3 Bucket

**Via AWS Console:**
1. Go to S3 Console
2. Create bucket: `insurance-verification-frontend`
3. Region: `us-east-1` (or your preferred region)
4. Block all public access: **Uncheck** (we'll use CloudFront)
5. Create bucket

### 4.3 Upload Build Files

```bash
# Install AWS CLI if not already
pip3 install awscli

# Configure AWS CLI
aws configure

# Upload files
cd dist/insurance-verification-frontend
aws s3 sync . s3://insurance-verification-frontend/ --delete
```

### 4.4 Configure S3 for Static Website

**Bucket Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::insurance-verification-frontend/*"
    }
  ]
}
```

**Static Website Hosting:**
- Enable static website hosting
- Index document: `index.html`
- Error document: `index.html` (for Angular routing)

### 4.5 Setup CloudFront

**Create Distribution:**
1. Go to CloudFront Console
2. Create distribution
3. Origin domain: Select S3 bucket
4. Origin access: Origin access control (recommended)
5. Viewer protocol policy: Redirect HTTP to HTTPS
6. Allowed HTTP methods: GET, HEAD, OPTIONS
7. Cache policy: CachingOptimized
8. Price class: Use only North America and Europe (or as needed)
9. Alternate domain name (CNAME): `www.yourdomain.com`, `yourdomain.com`
10. SSL Certificate: Request new ACM certificate or use existing
11. Create distribution

**CloudFront Settings:**
- Default root object: `index.html`
- Error pages: Create custom error response
  - HTTP error code: 404
  - Customize error response: Yes
  - Response page path: `/index.html`
  - HTTP response code: 200

### 4.6 Update DNS

**Add DNS Records (Route 53 or your DNS provider):**
```
Type: A (Alias)
Name: yourdomain.com
Value: CloudFront distribution domain
Alias: Yes (if Route 53)

Type: A (Alias)
Name: www.yourdomain.com
Value: CloudFront distribution domain
Alias: Yes
```

---

## Phase 5: File Storage Setup (S3)

### 5.1 Create S3 Bucket for Uploads

```bash
# Create bucket via CLI
aws s3 mb s3://insurance-verification-files --region us-east-1
```

### 5.2 Configure Bucket Policy

**Policy for private files (backend access only):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBackendAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:user/backend-user"
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::insurance-verification-files/*"
    }
  ]
}
```

### 5.3 Create IAM User for Backend

1. Go to IAM Console
2. Create user: `backend-s3-user`
3. Attach policy: `AmazonS3FullAccess` (or custom policy)
4. Generate access keys
5. Save keys in backend `.env`

---

## Phase 6: Email Setup (AWS SES)

### 6.1 Verify Email Domain

1. Go to SES Console
2. Click "Verified identities"
3. Create identity
4. Choose "Domain"
5. Enter your domain
6. Add DNS records (TXT, CNAME) to your DNS provider
7. Wait for verification (up to 72 hours)

### 6.2 Request Production Access

**By default, SES is in sandbox mode:**
- Can only send to verified emails
- Limit: 200 emails per day

**Request production access:**
1. Go to SES Console
2. Click "Account dashboard"
3. Click "Request production access"
4. Fill out form (use case, compliance, etc.)
5. Wait for approval (24-48 hours)

### 6.3 Configure SMTP Credentials

1. Go to SES Console
2. Click "SMTP settings"
3. Create SMTP credentials
4. Save credentials

**Update backend .env:**
```env
AWS_SES_SMTP_USERNAME=smtp-username
AWS_SES_SMTP_PASSWORD=smtp-password
AWS_SES_SMTP_HOST=email-smtp.us-east-1.amazonaws.com
AWS_SES_SMTP_PORT=587
```

---

## Phase 7: Domain & SSL Setup

### 7.1 Register Domain (Optional)

Use Route 53 or external registrar (Namecheap, GoDaddy, etc.)

### 7.2 SSL Certificates

**For Backend (api.yourdomain.com):**
- Already done via Certbot in Phase 2.8

**For Frontend (yourdomain.com):**
- Request ACM certificate in CloudFront region (us-east-1)
- Verify via DNS (add CNAME records)
- Attach to CloudFront distribution

---

## Phase 8: Monitoring & Logging

### 8.1 CloudWatch Setup

**Backend Logs:**
```bash
# Install CloudWatch agent on EC2
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb

# Configure CloudWatch
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/config.json
```

**Config:**
```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/www/insurance-backend/logs/*.log",
            "log_group_name": "/aws/ec2/insurance-backend",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

### 8.2 Setup Alarms

**Create CloudWatch Alarms:**
- CPU usage > 80%
- Memory usage > 80%
- Disk usage > 85%
- HTTP 5xx errors > 10
- RDS connection count > threshold

---

## Phase 9: Backup Strategy

### 9.1 RDS Automated Backups

**Enable in RDS Console:**
- Backup retention: 7 days
- Backup window: 03:00-04:00 UTC (off-peak)
- Enable automatic backups

### 9.2 S3 Versioning

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket insurance-verification-files \
  --versioning-configuration Status=Enabled
```

### 9.3 Application Backups

**Create backup script on EC2:**
```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/insurance-backend"

# Backup application files
tar -czf ${BACKUP_DIR}/app_${DATE}.tar.gz /var/www/insurance-backend

# Backup to S3
aws s3 cp ${BACKUP_DIR}/app_${DATE}.tar.gz s3://your-backup-bucket/

# Delete local backups older than 7 days
find ${BACKUP_DIR} -name "app_*.tar.gz" -mtime +7 -delete
```

**Setup cron job:**
```bash
crontab -e

# Daily backup at 2 AM
0 2 * * * /path/to/backup.sh
```

---

## Phase 10: Security Hardening

### 10.1 EC2 Security

```bash
# Update system regularly
sudo apt update && sudo apt upgrade -y

# Setup fail2ban
sudo apt install fail2ban -y

# Configure firewall
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 10.2 Environment Variables

- Never commit `.env` files to Git
- Use AWS Secrets Manager for sensitive data (advanced)
- Rotate credentials regularly

### 10.3 Database Security

- Use strong passwords
- Enable SSL connections only
- Restrict access to backend security group only
- Regular security updates

---

## Deployment Checklist

### Pre-Deployment
- [ ] All code tested locally
- [ ] Database migrations tested
- [ ] Environment variables documented
- [ ] SSL certificates ready
- [ ] DNS records prepared
- [ ] AWS resources provisioned

### Deployment
- [ ] RDS database created and configured
- [ ] Backend deployed and running
- [ ] n8n deployed and workflow imported
- [ ] Frontend built and uploaded to S3
- [ ] CloudFront distribution created
- [ ] DNS records updated
- [ ] SSL certificates installed
- [ ] File storage (S3) configured
- [ ] Email service (SES) configured

### Post-Deployment
- [ ] Test user form submission
- [ ] Test admin login
- [ ] Test verification workflow end-to-end
- [ ] Verify email notifications
- [ ] Check CloudWatch logs
- [ ] Setup monitoring alarms
- [ ] Enable automated backups
- [ ] Document production URLs
- [ ] Train team on admin portal

---

## Rollback Plan

### If Deployment Fails

**Backend:**
```bash
# SSH into EC2
pm2 stop insurance-backend
git checkout previous-working-commit
npm install
npm run build
pm2 start insurance-backend
```

**Frontend:**
```bash
# Redeploy previous version
aws s3 sync previous-dist/ s3://insurance-verification-frontend/
aws cloudfront create-invalidation --distribution-id YOUR_DIST_ID --paths "/*"
```

**Database:**
```bash
# Restore from backup
# Use RDS snapshot restore feature
```

---

## Cost Estimation (Monthly)

**AWS Services:**
- EC2 t3.small (Backend): ~$15
- EC2 t3.small (n8n): ~$15
- RDS db.t3.micro: ~$15
- S3 Storage (100GB): ~$2.30
- CloudFront (1TB transfer): ~$85
- Route 53 (Hosted Zone): $0.50
- SES (10,000 emails): $1.00

**Total: ~$133.80/month** (can be optimized)

**Cost Optimization:**
- Use Reserved Instances (save ~40%)
- Use S3 Intelligent-Tiering
- Enable CloudFront compression
- Optimize database instance size

---

## Maintenance Schedule

**Daily:**
- Monitor CloudWatch logs
- Check PM2 status
- Review error rates

**Weekly:**
- Review failed verifications
- Check disk usage
- Update dependencies

**Monthly:**
- Security patches
- Database optimization
- Review AWS costs
- Rotate credentials

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **Target Platform:** AWS
- **Status:** Complete