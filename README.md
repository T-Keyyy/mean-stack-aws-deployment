# MEAN Stack AWS Deployment

A fully containerized MEAN (MongoDB, Express, Angular, Node.js) stack application deployed on AWS with load balancing and high availability.

🌐 **Live Demo**: http://18.118.159.237

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Requirements Met](#requirements-met)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Deployment to AWS](#deployment-to-aws)
- [Project Structure](#project-structure)
- [High Availability Testing](#high-availability-testing)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

This project demonstrates a production-ready MEAN stack application with:
- **Dockerized frontend and backend**
- **Internet-exposed via AWS EC2**
- **Nginx load balancer** for traffic distribution
- **High availability** with 3 frontend replicas
- **Cost-effective** deployment using AWS free tier

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  AWS EC2 (t3.micro) │
              │  Public IP: x.x.x.x │
              └─────────┬───────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  Nginx (Port 80) │
              │  Load Balancer   │
              └─────────┬───────────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐
    │Frontend │   │Frontend │   │Frontend │
    │  (1)    │   │  (2)    │   │  (3)    │
    └────┬────┘   └────┬────┘   └────┬────┘
         │             │             │
         └─────────────┼─────────────┘
                       │ /api
                       ▼
              ┌─────────────────┐
              │  Backend API     │
              │  (Node/Express)  │
              └─────────┬────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  MongoDB 4.4     │
              │  (Persistent)    │
              └──────────────────┘
```

### Component Breakdown

- **Nginx**: Reverse proxy and load balancer (least_conn algorithm)
- **Frontend (x3)**: Angular 6 SPA served by Nginx (multi-stage Docker build)
- **Backend**: Node.js/Express API
- **MongoDB**: NoSQL database with persistent volume

## ✅ Requirements Met

| Requirement | Implementation | Status |
|------------|----------------|--------|
| Dockerized frontend & backend | Multi-stage Dockerfiles | ✅ |
| Internet exposed | AWS EC2 with public IP | ✅ |
| Load balancer | Nginx with upstream configuration | ✅ |
| High availability | 3 frontend replicas with failover | ✅ |
| Cost effective | AWS free tier (t3.micro) | ✅ |

## 📦 Prerequisites

### Local Development
- Docker Desktop
- Node.js 10.x (for Angular 6)
- Git

### AWS Deployment
- AWS Account (free tier)
- SSH client (PowerShell on Windows)

## 🚀 Quick Start

### Local Development

1. **Clone the repository**
```bash
git clone https://github.com/T-keyyy/mean-stack-aws-deployment.git
cd mean-stack-aws-deployment
```

2. **Start all services**
```bash
docker-compose up -d --scale frontend=3
```

3. **Access the application**
- Frontend: http://localhost
- Backend API: http://localhost/api
- MongoDB: localhost:27017

> **Note**: This is for local development. For production deployment on AWS, see [Deployment to AWS](#️-deployment-to-aws) section below.

4. **Stop services**
```bash
docker-compose down
```

## ☁️ Deployment to AWS

### Step 1: Launch EC2 Instance

1. Go to AWS Console → EC2 → Launch Instance
2. Configure:
   - **Name**: mean-stack-server
   - **AMI**: Ubuntu Server 24.04 LTS
   - **Instance type**: t2.micro (free tier)
   - **Key pair**: Create new (download .pem file)
   - **Security Group**: Allow SSH (22), HTTP (80), HTTPS (443)
3. Launch instance

### Step 2: Connect to EC2

**Windows (PowerShell):**
```powershell
# Secure the key file
cd ~\Downloads
icacls mean-stack-key.pem /reset
icacls mean-stack-key.pem /grant:r "$($env:USERNAME):(R)"
icacls mean-stack-key.pem /inheritance:r

# Connect via SSH
ssh -i "mean-stack-key.pem" ubuntu@MY-EC2-PUBLIC-IP
```

**Mac/Linux**:
```bash
chmod 400 mean-stack-key.pem
ssh -i "mean-stack-key.pem" ubuntu@MY-EC2-PUBLIC-IP
```

### Step 3: Install Docker

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

### Step 4: Deploy Application

```bash
# Clone repository
git clone https://github.com/T-keyyy/mean-stack-aws-deployment.git
cd mean-stack-aws-deployment

# Start all services
sudo docker-compose up -d --scale frontend=3

# Verify all containers are running
sudo docker-compose ps
```

### Step 5: Access My App

**Production URL**: `http://MY-EC2-PUBLIC-IP`

Example: http://18.118.159.237 (current deployment)

The application is now **live on the internet** and accessible from anywhere!

## 📁 Project Structure

```
mean-stack-aws-deployment/
├── backend/                 # Node.js/Express API
│   ├── Dockerfile          # Backend container config
│   └── Project/            # Application code
│       ├── app.js         # Express server
│       ├── config/        # MongoDB connection
│       └── controllers/   # API endpoints
├── frontend/               # Angular 6 SPA
│   ├── Dockerfile         # Multi-stage build
│   ├── nginx.conf         # Angular routing config
│   └── Angular6/          # Source code
│       └── src/
│           └── environments/  # API endpoint config
├── nginx/                 # Load balancer
│   └── nginx.conf        # Upstream & routing config
├── docker-compose.yml    # Orchestration config
├── .env.example          # Environment template
└── README.md            # This file
```

## 🔧 Configuration Files

### docker-compose.yml
- Defines 6 services: nginx, 3x frontend, backend, mongodb
- Enables frontend scaling
- Creates custom network for service discovery
- Configures persistent MongoDB volume

### nginx/nginx.conf
- Load balancer with `least_conn` algorithm
- Dynamic DNS resolution (`resolver 127.0.0.11`)
- Routes `/api` to backend
- Routes `/` to frontend cluster

### Frontend Dockerfile
- Multi-stage build (node:10-alpine → nginx:alpine)
- Builds Angular app
- Serves static files via Nginx
- Handles client-side routing

### Backend Dockerfile
- Single-stage build (node:10-alpine)
- Installs dependencies
- Exposes API on port 80

## 🧪 High Availability Testing

### Test Failover

1. **Check running containers**
```bash
sudo docker-compose ps
```

2. **Stop one frontend**
```bash
sudo docker stop ubuntu_frontend_1
```

3. **Refresh browser** - Site should remain accessible

4. **Restore container**
```bash
sudo docker start ubuntu_frontend_1
```

### How It Works

- Nginx uses dynamic DNS resolution with `valid=5s`
- Variable-based proxy_pass forces DNS re-resolution on each request
- When a container stops, Nginx detects failure within seconds
- Traffic automatically routes to healthy containers via Docker's service discovery

## 🐛 Troubleshooting

### Container Won't Start

```bash
# Check logs
sudo docker-compose logs SERVICE_NAME

# Rebuild specific service
sudo docker-compose build --no-cache SERVICE_NAME
sudo docker-compose up -d
```

### MongoDB Connection Issues

- Verify `MONGODB_URI` in docker-compose.yml
- Check backend config: `backend/Project/config/config.json`
- Ensure format: `mongodb://mongo:27017/meanapp`

### Frontend 404 Errors

- Verify `nginx.conf` in frontend container handles Angular routing
- Check `environment.prod.ts` has `apiBaseUrl: '/api'`

### Port Already in Use

```bash
# Find process using port 80
sudo lsof -i :80
# Or on Windows:
netstat -ano | findstr :80

# Stop conflicting service or change port in docker-compose.yml
```

### Permission Denied (npm build)

```bash
# Fix file permissions
chmod -R 755 frontend
chmod -R 755 backend
```

## 📊 Monitoring

### View Logs
```bash
# All services
sudo docker-compose logs

# Specific service
sudo docker-compose logs frontend

# Follow logs (real-time)
sudo docker-compose logs -f

# Last 50 lines
sudo docker-compose logs --tail 50
```

### Check Resource Usage
```bash
docker stats
```

## 🛑 Stopping the Application

### Stop all containers
```bash
sudo docker-compose down
```

### Stop but keep data
```bash
sudo docker-compose stop
```

### Remove everything (including volumes)
```bash
sudo docker-compose down -v
```

## 💰 AWS Cost Management

**Free Tier Limits:**
- 750 hours/month of t2.micro (covers 1 instance 24/7)
- 30 GB EBS storage
- 15 GB data transfer out

**To avoid charges:**
- Stop EC2 instance when not in use
- Delete instance when project complete
- Monitor AWS billing dashboard

**Stop EC2 instance:**
1. AWS Console → EC2 → Instances
2. Select instance → Instance State → Stop

## 🔐 Security Notes

**For Production:**
- Use HTTPS with SSL certificate (Let's Encrypt)
- Restrict SSH to my IP only
- Use environment variables for secrets
- Enable AWS CloudWatch for monitoring
- Regular security updates: `sudo apt update && sudo apt upgrade`

## 📝 License

This project is open source and available for educational purposes.

## 📧 Contact

Created for technical assessment demonstration.

---

**Built with:** MongoDB • Express • Angular 6 • Node.js • Docker • Nginx • AWS EC2
