# Docker Deployment Guide

Complete guide for deploying the User Management Application using Docker Compose.

## 📋 Table of Contents
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Deployment to EC2](#deployment-to-ec2)
- [Commands](#commands)
- [Troubleshooting](#troubleshooting)

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    Internet                         │
└──────────────────┬──────────────────────────────────┘
                   │
         ┌─────────▼────────┐
         │  Nginx (Port 80) │  ← Reverse Proxy
         │  Load Balancer   │
         └────┬─────────┬───┘
              │         │
    ┌─────────▼─┐   ┌──▼──────────┐
    │ Frontend  │   │  Backend    │
    │ Next.js   │   │  Node.js    │
    │ Port 3000 │   │  Port 5000  │
    └───────────┘   └──┬──────┬───┘
                       │      │
                  ┌────▼──┐ ┌─▼─────┐
                  │MongoDB│ │ Redis │
                  │ 27017 │ │ 6379  │
                  └───────┘ └───────┘
```

### Network Isolation
- **frontend-network**: nginx ↔ frontend
- **backend-network**: nginx ↔ backend ↔ mongodb ↔ redis

### Data Persistence
- `mongodb_data`: Database files
- `mongodb_config`: MongoDB configuration
- `redis_data`: Redis persistence (AOF enabled)

## ✅ Prerequisites

### For Local Development
- Docker Engine 20.10+
- Docker Compose 2.0+
- Git

### For EC2 Deployment
- AWS EC2 instance (t3.small or larger)
- Ubuntu 22.04 LTS
- At least 2GB RAM, 20GB storage
- Security group allowing ports: 22, 80, 443

## 🚀 Quick Start

### 1. Clone the Repository
```bash
git clone <your-repo-url>
cd user-management
```

### 2. Configure Environment Variables
```bash
# Copy example environment file
cp .env.example .env

# Edit with your values
nano .env
```

**Important:** Update these values:
- `MONGO_ROOT_PASSWORD`: Strong password for MongoDB
- `JWT_SECRET`: Random 32+ character string
- `EMAIL_USER`: Your Gmail address
- `EMAIL_PASSWORD`: Gmail app-specific password
- `CORS_ORIGIN`: Your domain or EC2 public IP

### 3. Build and Start
```bash
# Build all containers
docker-compose build

# Start all services
docker-compose up -d

# Check status
docker-compose ps
```

### 4. Verify Deployment
```bash
# Check logs
docker-compose logs -f

# Test health endpoints
curl http://localhost/health           # Nginx
curl http://localhost/api/health       # Backend
```

### 5. Access Application
- **Frontend**: http://localhost
- **Backend API**: http://localhost/api
- **Health Check**: http://localhost/health

## ⚙️ Configuration

### Environment Variables (.env)

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `MONGO_ROOT_USER` | MongoDB admin username | admin | Yes |
| `MONGO_ROOT_PASSWORD` | MongoDB admin password | - | Yes |
| `JWT_SECRET` | JWT signing secret | - | Yes |
| `JWT_EXPIRES_IN` | JWT expiration time | 7d | No |
| `EMAIL_HOST` | SMTP server | smtp.gmail.com | Yes |
| `EMAIL_PORT` | SMTP port | 587 | No |
| `EMAIL_USER` | Email address | - | Yes |
| `EMAIL_PASSWORD` | Email password/app password | - | Yes |
| `EMAIL_FROM` | From email address | - | No |
| `CORS_ORIGIN` | Allowed CORS origin | http://localhost | Yes |
| `BCRYPT_ROUNDS` | Password hashing rounds | 10 | No |
| `OTP_EXPIRY_MINUTES` | OTP validity period | 5 | No |

### Gmail App Password Setup
1. Go to https://myaccount.google.com/apppasswords
2. Select "Mail" and your device
3. Generate password
4. Use generated password in `EMAIL_PASSWORD`

## 🌐 Deployment to EC2

### Step 1: Launch EC2 Instance
```bash
# Instance type: t3.small or t3.medium
# AMI: Ubuntu Server 22.04 LTS
# Storage: 20GB GP3
```

### Step 2: Configure Security Group
```bash
# Inbound Rules:
# - SSH (22): Your IP
# - HTTP (80): 0.0.0.0/0
# - HTTPS (443): 0.0.0.0/0
```

### Step 3: SSH to EC2
```bash
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

### Step 4: Install Docker & Docker Compose
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker ubuntu

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker --version
docker-compose --version

# Logout and login again for group changes
exit
```

### Step 5: Clone and Configure
```bash
# SSH back to EC2
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>

# Clone repository
git clone <your-repo-url>
cd user-management

# Create .env file
cp .env.example .env
nano .env  # Update with production values

# Update CORS_ORIGIN in .env
CORS_ORIGIN=http://<YOUR-EC2-PUBLIC-IP>
```

### Step 6: Deploy
```bash
# Build images
docker-compose build

# Start services
docker-compose up -d

# Check logs
docker-compose logs -f

# Verify all containers are healthy
docker-compose ps
```

### Step 7: Verify Deployment
```bash
# Test from EC2
curl http://localhost/health

# Test from your computer
curl http://<EC2-PUBLIC-IP>/health
```

## 📝 Commands

### Container Management
```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# Restart a specific service
docker-compose restart backend

# View logs
docker-compose logs -f
docker-compose logs -f backend  # Specific service

# Check container status
docker-compose ps

# Execute command in container
docker-compose exec backend sh
docker-compose exec mongodb mongosh
```

### Build & Rebuild
```bash
# Build all images
docker-compose build

# Rebuild specific service
docker-compose build backend

# Rebuild without cache
docker-compose build --no-cache

# Pull latest base images
docker-compose pull
```

### Data Management
```bash
# Backup MongoDB
docker-compose exec mongodb mongodump --out=/data/backup

# Restore MongoDB
docker-compose exec mongodb mongorestore /data/backup

# View volumes
docker volume ls

# Remove all data (CAUTION!)
docker-compose down -v
```

### Cleanup
```bash
# Stop and remove containers
docker-compose down

# Remove containers and volumes (deletes data!)
docker-compose down -v

# Remove unused images
docker image prune -a

# Complete cleanup
docker system prune -a --volumes
```

## 🔧 Troubleshooting

### Container Won't Start
```bash
# Check logs
docker-compose logs <service-name>

# Check container status
docker-compose ps

# Inspect container
docker inspect <container-name>
```

### Port Already in Use
```bash
# Find process using port 80
sudo lsof -i :80
sudo netstat -tulpn | grep :80

# Kill process
sudo kill -9 <PID>
```

### Permission Issues
```bash
# Fix Docker permissions
sudo usermod -aG docker $USER
newgrp docker

# Fix volume permissions
sudo chown -R $USER:$USER ./data
```

### MongoDB Connection Issues
```bash
# Check if MongoDB is healthy
docker-compose ps mongodb

# Test MongoDB connection
docker-compose exec mongodb mongosh --eval "db.adminCommand('ping')"

# View MongoDB logs
docker-compose logs mongodb
```

### Backend Can't Connect to MongoDB
- Verify `MONGODB_URI` in docker-compose.yml
- Check MongoDB container is healthy: `docker-compose ps`
- Ensure backend and MongoDB are on same network
- Check MongoDB credentials match

### Frontend Can't Reach Backend
- Verify Nginx is routing correctly: `docker-compose logs nginx`
- Check `NEXT_PUBLIC_API_URL=/api` in frontend environment
- Test backend directly: `curl http://localhost:5000/health`
- Check CORS settings in backend

### High Memory Usage
```bash
# Check memory usage
docker stats

# Limit Redis memory (already configured to 256MB)
# Limit container memory in docker-compose.yml:
# deploy:
#   resources:
#     limits:
#       memory: 512M
```

### Logs Growing Too Large
```bash
# Check log sizes
docker system df

# Clean logs
docker-compose down
sudo truncate -s 0 /var/lib/docker/containers/*/*-json.log

# Logs are already limited to 10MB x 3 files per container
```

### SSL/HTTPS Setup (Production)
```bash
# Install Certbot
sudo apt install certbot

# Get SSL certificate
sudo certbot certonly --standalone -d your-domain.com

# Update nginx/conf.d/default.conf
# Uncomment HTTPS server block
# Update paths to certificates

# Restart nginx
docker-compose restart nginx
```

## 📊 Monitoring

### Health Checks
All services have built-in health checks:
- Nginx: `http://localhost/health`
- Frontend: `http://localhost:3000`
- Backend: `http://localhost:5000/health`
- MongoDB: `mongosh ping`
- Redis: `redis-cli ping`

### View Resource Usage
```bash
# Real-time stats
docker stats

# Disk usage
docker system df

# Network usage
docker network ls
docker network inspect user-management-backend-net
```

## 🔒 Security Best Practices

1. **Change Default Passwords**: Update all passwords in `.env`
2. **Use Strong JWT Secret**: Minimum 32 random characters
3. **Enable Firewall**: Only allow necessary ports
4. **Regular Updates**: Keep Docker and images updated
5. **Don't Expose Database Ports**: Keep MongoDB/Redis internal
6. **Use HTTPS in Production**: Set up SSL certificates
7. **Backup Data Regularly**: Schedule MongoDB backups
8. **Monitor Logs**: Check for suspicious activity

## 📦 Next Steps

1. ✅ Set up domain name and DNS
2. ✅ Configure SSL/HTTPS with Let's Encrypt
3. ✅ Set up automated backups
4. ✅ Configure monitoring (e.g., Prometheus + Grafana)
5. ✅ Set up CI/CD with GitHub Actions
6. ✅ Configure log aggregation (e.g., ELK stack)
7. ✅ Implement rate limiting in Nginx
8. ✅ Set up alerting for downtime

## 📚 Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [Next.js Documentation](https://nextjs.org/docs)
