# Deployment Guide

Complete step-by-step guide for deploying the MEAN stack application to AWS EC2.

## Prerequisites Checklist

- [ ] AWS Account (free tier eligible)
- [ ] Credit card on file (required even for free tier)
- [ ] SSH client installed (PowerShell on Windows, Terminal on Mac/Linux)
- [ ] Git installed locally
- [ ] Basic command line knowledge

## Part 1: AWS Setup

### 1.1 Launch EC2 Instance

1. **Login to AWS Console**
   - Go to https://console.aws.amazon.com
   - Navigate to EC2 Dashboard

2. **Launch Instance**
   ```
   Click: "Launch Instance"
   ```

3. **Configure Instance**

   **Name and tags**:
   - Name: `mean-stack-server`

   **Application and OS Images**:
   - AMI: Ubuntu Server 24.04 LTS (HVM)
   - Architecture: 64-bit (x86)

   **Instance type**:
   - Type: `t3.micro` (Free tier eligible)
   - vCPUs: 1
   - Memory: 1 GiB

   **Key pair (login)**:
   - Click "Create new key pair"
   - Name: `mean-stack-key`
   - Type: RSA
   - Format: .pem (for Mac/Linux) or .ppk (for PuTTY)
   - Download and save securely

   **Network settings**:
   - VPC: Default
   - Subnet: No preference
   - Auto-assign public IP: Enable
   - Firewall (security groups): Create new
     - Rule 1: SSH | TCP | 22 | My IP
     - Rule 2: HTTP | TCP | 80 | Anywhere (0.0.0.0/0)
     - Rule 3: HTTPS | TCP | 443 | Anywhere (0.0.0.0/0)

   **Configure storage**:
   - Size: 8 GiB (free tier: up to 30 GiB)
   - Volume type: gp3
   - Delete on termination: Yes

4. **Launch**
   - Review summary
   - Click "Launch instance"
   - Wait for "Instance state" = "Running"
   - Note the **Public IPv4 address**

### 1.2 Configure Security Group (If Needed)

If you forgot to add HTTP/HTTPS rules:

1. Go to EC2 → Security Groups
2. Select your instance's security group
3. Click "Edit inbound rules"
4. Add rules:
   ```
   Type: HTTP  | Protocol: TCP | Port: 80  | Source: 0.0.0.0/0
   Type: HTTPS | Protocol: TCP | Port: 443 | Source: 0.0.0.0/0
   ```
5. Save rules

## Part 2: Connect to EC2

### 2.1 Windows (PowerShell)

```powershell
# Navigate to key location
cd ~\Downloads

# Secure the key file (remove inherited permissions)
icacls mean-stack-key.pem /reset
icacls mean-stack-key.pem /grant:r "$($env:USERNAME):(R)"
icacls mean-stack-key.pem /inheritance:r

# Connect via SSH
ssh -i "mean-stack-key.pem" ubuntu@YOUR-EC2-PUBLIC-IP
```

**If you see "WARNING: UNPROTECTED PRIVATE KEY FILE":**
The above icacls commands should fix it. Make sure you run PowerShell as yourself (not Administrator).

### 2.2 Mac/Linux

```bash
# Navigate to key location
cd ~/Downloads

# Set permissions
chmod 400 mean-stack-key.pem

# Connect via SSH
ssh -i "mean-stack-key.pem" ubuntu@YOUR-EC2-PUBLIC-IP
```

### 2.3 First Connection

When connecting for the first time:
```
The authenticity of host 'x.x.x.x' can't be established.
ECDSA key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)?
```

Type: `yes` and press Enter

You should see:
```
Welcome to Ubuntu 24.04 LTS
...
ubuntu@ip-xxx-xxx-xxx-xxx:~$
```

## Part 3: Server Setup

### 3.1 Update System

```bash
sudo apt update
sudo apt upgrade -y
```

### 3.2 Install Docker

```bash
# Install Docker
sudo apt install -y docker.io

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group (optional, avoids sudo)
sudo usermod -aG docker ubuntu

# Verify installation
sudo docker --version
```

Expected output: `Docker version 24.x.x`

### 3.3 Install Docker Compose

```bash
# Install docker-compose
sudo apt install -y docker-compose

# Verify installation
docker-compose --version
```

Expected output: `docker-compose version 1.29.x`

## Part 4: Deploy Application

### 4.1 Clone Repository

```bash
# Clone the repository
git clone https://github.com/T-keyyy/mean-stack-aws-deployment.git

# Navigate to directory
cd mean-stack-aws-deployment

# Verify files
ls -la
```

Expected files:
```
backend/
frontend/
nginx/
docker-compose.yml
README.md
```

### 4.2 Build and Start Containers

```bash
# Build images (first time only)
sudo docker-compose build

# Start all services with 3 frontend replicas
sudo docker-compose up -d --scale frontend=3
```

**What this does**:
- Builds Docker images for backend and frontend
- Downloads MongoDB image
- Creates custom network
- Starts 6 containers: nginx, 3x frontend, backend, mongodb

### 4.3 Verify Deployment

```bash
# Check all containers are running
sudo docker-compose ps
```

Expected output:
```
Name                    State    Ports
meanapp_backend_1       Up      80/tcp
meanapp_frontend_1      Up      80/tcp
meanapp_frontend_2      Up      80/tcp
meanapp_frontend_3      Up      80/tcp
meanapp_mongo_1         Up      27017/tcp
meanapp_nginx_1         Up      0.0.0.0:80->80/tcp
```

**All should show "Up"**

### 4.4 Check Logs

```bash
# View all logs
sudo docker-compose logs

# View specific service
sudo docker-compose logs frontend

# Follow logs in real-time
sudo docker-compose logs -f

# Last 50 lines
sudo docker-compose logs --tail 50
```

## Part 5: Test Application

### 5.1 Access from Browser

Open browser and navigate to:
```
http://YOUR-EC2-PUBLIC-IP
```

You should see the Angular application homepage.

### 5.2 Test User Registration

1. Click "Sign Up"
2. Fill in details:
   - Name: Test User
   - Email: test@example.com
   - Password: password123
   - Phone: 1234567890
3. Click "Sign Up"
4. Check for success message

### 5.3 Verify API Connection

Check backend logs for registration:
```bash
sudo docker-compose logs backend | grep -i "post\|signup"
```

### 5.4 Test High Availability

```bash
# Stop one frontend
sudo docker stop meanapp_frontend_1

# Refresh browser - should still work

# Check Nginx routing
sudo docker-compose logs nginx | tail -20

# Restart frontend
sudo docker start meanapp_frontend_1
```

## Part 6: Common Issues

### Issue: "Permission denied" on docker commands

**Solution**: Use `sudo` or re-login after adding user to docker group
```bash
# If you added user to docker group
exit
# SSH back in
ssh -i "mean-stack-key.pem" ubuntu@YOUR-EC2-PUBLIC-IP
```

### Issue: "Port already in use"

**Solution**: Check what's using port 80
```bash
sudo lsof -i :80
# Stop the conflicting service
sudo systemctl stop apache2  # if Apache is running
```

### Issue: "Cannot connect to Docker daemon"

**Solution**: Start Docker service
```bash
sudo systemctl start docker
sudo systemctl status docker
```

### Issue: "npm ERR! code EACCES" during build

**Solution**: Fix permissions
```bash
cd mean-stack-aws-deployment
chmod -R 755 frontend
chmod -R 755 backend
sudo docker-compose build --no-cache
```

### Issue: Frontend shows 404 on routes

**Solution**: Verify frontend nginx.conf exists
```bash
cat frontend/nginx.conf
```

Should contain:
```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

### Issue: "Cannot connect to MongoDB"

**Solution**: Check backend config
```bash
cat backend/Project/config/config.json
```

Should be:
```json
{
  "mongoURI": "mongodb://mongo:27017/meanapp"
}
```

### Issue: Site not accessible from browser

**Checklist**:
1. [ ] EC2 instance is running
2. [ ] Security Group allows HTTP (port 80)
3. [ ] Nginx container is up: `sudo docker-compose ps`
4. [ ] Using correct public IP (check EC2 console)
5. [ ] Using http:// not https://

## Part 7: Maintenance

### View Status

```bash
# Container status
sudo docker-compose ps

# Resource usage
docker stats

# Disk space
df -h
```

### Restart Services

```bash
# Restart all
sudo docker-compose restart

# Restart specific service
sudo docker-compose restart frontend

# Full restart (rebuild)
sudo docker-compose down
sudo docker-compose up -d --scale frontend=3
```

### Update Application

```bash
# Pull latest code
cd mean-stack-aws-deployment
git pull

# Rebuild and restart
sudo docker-compose build
sudo docker-compose up -d --scale frontend=3
```

### Backup Database

```bash
# Create backup directory
mkdir -p ~/backups

# Export MongoDB data
sudo docker exec meanapp_mongo_1 mongodump --out /backup

# Copy from container
sudo docker cp meanapp_mongo_1:/backup ~/backups/mongodb-$(date +%Y%m%d)
```

### Restore Database

```bash
# Copy backup to container
sudo docker cp ~/backups/mongodb-20241121 meanapp_mongo_1:/restore

# Restore
sudo docker exec meanapp_mongo_1 mongorestore /restore
```

## Part 8: Cleanup

### Stop Application (Keep Data)

```bash
sudo docker-compose stop
```

Resume later:
```bash
sudo docker-compose start
```

### Stop Application (Remove Containers)

```bash
sudo docker-compose down
```

Data persists in volumes. Restart with:
```bash
sudo docker-compose up -d --scale frontend=3
```

### Complete Cleanup (Remove Everything)

```bash
# Stop and remove containers, networks, volumes
sudo docker-compose down -v

# Remove images
sudo docker system prune -a
```

### Terminate EC2 Instance

**⚠️ WARNING: This deletes everything permanently**

1. AWS Console → EC2 → Instances
2. Select your instance
3. Instance State → Terminate Instance
4. Confirm

**Remember**: Termination is permanent. Use "Stop" instead if you want to preserve the instance.

## Part 9: Cost Management

### Monitor Costs

1. AWS Console → Billing Dashboard
2. Check "Free tier usage"
3. Set up billing alerts:
   - Billing preferences → Alert preferences
   - Enable "Receive Billing Alerts"
   - Set threshold (e.g., $5)

### Stop Instance When Not Needed

**Stop instance** (no charges for compute, small storage fee):
```
AWS Console → EC2 → Instance State → Stop
```

**Start when needed**:
```
AWS Console → EC2 → Instance State → Start
```

**Note**: Public IP changes each time you stop/start (use Elastic IP to keep same IP, but costs $0.005/hour when not attached)

## Part 10: Next Steps

### Security Hardening

1. **Enable HTTPS**:
   - Install Certbot
   - Get Let's Encrypt SSL certificate
   - Update Nginx configuration

2. **Restrict SSH**:
   - Change Security Group SSH rule from "Anywhere" to "My IP"
   - Use key authentication only (already configured)

3. **Enable Firewall**:
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### Monitoring Setup

1. **CloudWatch Agent**:
```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

2. **Set up alarms** for:
   - CPU utilization > 80%
   - Disk space > 80%
   - Memory usage > 80%

### CI/CD Pipeline

Consider setting up:
- GitHub Actions for automated deployment
- Automated testing before deployment
- Rollback procedures

## Checklist Summary

Deployment complete when:
- [ ] EC2 instance running
- [ ] SSH connection working
- [ ] Docker installed
- [ ] 6 containers running
- [ ] Site accessible at http://PUBLIC-IP
- [ ] User registration working
- [ ] High availability tested (1 frontend stopped, site still works)
- [ ] Logs show no errors
- [ ] Monitoring/alerting configured

## Support

For issues:
1. Check logs: `sudo docker-compose logs`
2. Verify containers: `sudo docker-compose ps`
3. Check AWS Security Groups
4. Review troubleshooting section above

---

**Estimated deployment time**: 30-45 minutes (first time)

**Estimated cost**: $0 (within free tier for first 12 months)
