# Troubleshooting Guide

Common issues and their solutions for the MEAN stack deployment.

## Table of Contents

1. [Container Issues](#container-issues)
2. [Network Issues](#network-issues)
3. [Build Issues](#build-issues)
4. [AWS Issues](#aws-issues)
5. [Application Issues](#application-issues)
6. [Database Issues](#database-issues)

---

## Container Issues

### Container Won't Start

**Symptoms**:
```bash
$ sudo docker-compose ps
Name                State
frontend_1          Exit 1
```

**Diagnosis**:
```bash
sudo docker-compose logs frontend
```

**Common Causes**:

1. **Port conflict**
   ```
   Error: bind: address already in use
   ```
   **Solution**:
   ```bash
   # Find process
   sudo lsof -i :80
   # Kill process or stop service
   sudo systemctl stop apache2
   ```

2. **Permission issues**
   ```
   Error: EACCES: permission denied
   ```
   **Solution**:
   ```bash
   chmod -R 755 frontend
   chmod -R 755 backend
   sudo docker-compose up -d
   ```

3. **Missing files**
   ```
   Error: ENOENT: no such file or directory
   ```
   **Solution**: Verify all files are present
   ```bash
   ls -R frontend/
   ls -R backend/
   ```

### Container Keeps Restarting

**Symptoms**:
```bash
$ sudo docker-compose ps
Name                State
backend_1           Restarting
```

**Diagnosis**:
```bash
# Check exit code
sudo docker inspect backend_1 | grep -A 5 "State"

# View recent logs
sudo docker-compose logs --tail 100 backend
```

**Solutions**:

1. **Application error**: Fix code errors
2. **Missing dependencies**: Rebuild container
   ```bash
   sudo docker-compose build --no-cache backend
   sudo docker-compose up -d
   ```
3. **Resource limits**: Increase in docker-compose.yml

### Cannot Stop Container

**Symptoms**:
```bash
$ sudo docker stop frontend_1
# Hangs...
```

**Solution**:
```bash
# Force kill
sudo docker kill frontend_1

# Remove stuck container
sudo docker rm -f frontend_1

# Restart service
sudo docker-compose up -d --scale frontend=3
```

---

## Network Issues

### Cannot Access Site from Browser

**Checklist**:

1. **Verify EC2 is running**
   - AWS Console → EC2 → Instance state = "Running"

2. **Check Security Group**
   ```
   Inbound rules must include:
   HTTP (80) | 0.0.0.0/0
   ```

3. **Verify Nginx is running**
   ```bash
   sudo docker-compose ps nginx
   # Should show "Up"
   ```

4. **Check Nginx logs**
   ```bash
   sudo docker-compose logs nginx
   ```

5. **Test locally on server**
   ```bash
   curl http://localhost
   # Should return HTML
   ```

6. **Verify public IP**
   - Check EC2 console for correct public IP
   - Use http:// not https:// (unless SSL configured)

### 502 Bad Gateway

**Symptoms**:
Browser shows "502 Bad Gateway" from Nginx

**Causes**:

1. **Backend not running**
   ```bash
   sudo docker-compose ps backend
   ```
   **Solution**: Start backend
   ```bash
   sudo docker-compose up -d backend
   ```

2. **Frontend containers all down**
   ```bash
   sudo docker-compose ps frontend
   ```
   **Solution**: Start frontends
   ```bash
   sudo docker-compose up -d --scale frontend=3
   ```

3. **Wrong upstream configuration**
   ```bash
   cat nginx/nginx.conf | grep upstream
   ```
   **Should contain**:
   ```nginx
   upstream frontend {
       least_conn;
       server frontend:80;
   }
   ```

### 504 Gateway Timeout

**Symptoms**:
Browser shows "504 Gateway Timeout"

**Causes**:
- Backend is slow or hanging
- Database connection timeout

**Solutions**:

1. **Check backend logs**
   ```bash
   sudo docker-compose logs backend
   ```

2. **Restart backend**
   ```bash
   sudo docker-compose restart backend
   ```

3. **Check MongoDB connection**
   ```bash
   sudo docker exec backend_1 ping mongo
   ```

### DNS Resolution Failures

**Symptoms**:
```
nginx: [emerg] host not found in upstream "frontend"
```

**Solution**: Ensure custom network exists
```bash
sudo docker network ls | grep meanapp
```

If missing:
```bash
sudo docker-compose down
sudo docker-compose up -d
```

---

## Build Issues

### npm Install Failures

**Symptoms**:
```
npm ERR! code EACCES
npm ERR! syscall access
npm ERR! errno -13
```

**Solutions**:

1. **File permissions**
   ```bash
   chmod -R 755 frontend/Angular6
   chmod -R 755 backend/Project
   ```

2. **Clear cache and rebuild**
   ```bash
   sudo docker-compose build --no-cache
   ```

3. **Node version mismatch**
   - Ensure Dockerfile uses `node:10-alpine`

### Angular Build Fails

**Symptoms**:
```
ERROR in src/app/...
```

**Common Issues**:

1. **Missing comma in environment.prod.ts**
   ```typescript
   // Wrong
   export const environment = {
     production: true
     apiBaseUrl: '/api'
   };
   
   // Correct
   export const environment = {
     production: true,
     apiBaseUrl: '/api'
   };
   ```

2. **TypeScript errors**
   - Check syntax in all .ts files
   - Verify imports are correct

### Docker Build "no space left on device"

**Symptoms**:
```
Error: no space left on device
```

**Solutions**:

1. **Clean up Docker**
   ```bash
   # Remove unused images
   sudo docker image prune -a
   
   # Remove unused volumes
   sudo docker volume prune
   
   # Full cleanup
   sudo docker system prune -a
   ```

2. **Check disk space**
   ```bash
   df -h
   ```

3. **Increase EC2 storage**
   - AWS Console → EC2 → Volumes
   - Modify volume size
   - Resize filesystem:
   ```bash
   sudo growpart /dev/xvda 1
   sudo resize2fs /dev/xvda1
   ```

---

## AWS Issues

### Cannot SSH into EC2

**Symptoms**:
```
Connection timed out
```

**Solutions**:

1. **Check Security Group**
   - Ensure SSH (22) rule exists
   - Source should be "My IP" or 0.0.0.0/0

2. **Verify instance is running**
   - AWS Console → EC2 → Instance State

3. **Check key file permissions**
   ```bash
   # Windows
   icacls mean-stack-key.pem
   # Should show only your username
   
   # Mac/Linux
   ls -la mean-stack-key.pem
   # Should be -r-------- (400)
   ```

4. **Use correct username**
   ```bash
   # Ubuntu AMI uses 'ubuntu'
   ssh -i "key.pem" ubuntu@PUBLIC-IP
   
   # NOT 'ec2-user' or 'root'
   ```

### "WARNING: UNPROTECTED PRIVATE KEY FILE"

**Windows Solution**:
```powershell
icacls mean-stack-key.pem /reset
icacls mean-stack-key.pem /grant:r "$($env:USERNAME):(R)"
icacls mean-stack-key.pem /inheritance:r
```

**Mac/Linux Solution**:
```bash
chmod 400 mean-stack-key.pem
```

### EC2 Instance Keeps Stopping

**Causes**:
- Out of memory (t2.micro has only 1GB)
- Manual stop

**Solutions**:

1. **Check memory usage**
   ```bash
   free -m
   ```

2. **Add swap space**
   ```bash
   sudo fallocate -l 1G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   # Make permanent
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```

3. **Upgrade instance type** (costs money)
   - Stop instance
   - Actions → Instance Settings → Change instance type
   - Select t2.small or larger

### Charges on Free Tier

**Common causes**:
- Running instance outside 750 hours/month
- Using non-free tier instance type
- Data transfer exceeded 15GB
- Multiple instances running

**Prevention**:
1. Set up billing alerts
2. Stop instance when not needed
3. Use Cost Explorer
4. Monitor Free Tier usage

---

## Application Issues

### Cannot Register User

**Symptoms**:
- Form submission fails
- No success message

**Diagnosis**:

1. **Check browser console** (F12)
   - Look for API errors
   - CORS issues
   - Network failures

2. **Check backend logs**
   ```bash
   sudo docker-compose logs backend | grep -i error
   ```

3. **Test API directly**
   ```bash
   curl -X POST http://localhost/api/user/signup \
     -H "Content-Type: application/json" \
     -d '{"name":"Test","email":"test@test.com","password":"pass123","phone":"1234567890"}'
   ```

**Solutions**:

1. **MongoDB not connected**
   - Check backend logs for connection errors
   - Verify mongo container is running
   ```bash
   sudo docker-compose ps mongo
   ```

2. **CORS issues**
   - Verify backend allows requests from frontend
   - Check `app.js` for CORS configuration

3. **Validation errors**
   - Ensure all required fields filled
   - Check password requirements
   - Verify email format

### Frontend Shows 404 on Refresh

**Symptoms**:
- Homepage loads fine
- Navigate to /signup works
- Refresh on /signup shows 404

**Cause**: Frontend nginx not configured for SPA routing

**Solution**: Verify `frontend/nginx.conf` exists and contains:
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Rebuild frontend:
```bash
sudo docker-compose build --no-cache frontend
sudo docker-compose up -d --scale frontend=3
```

### API Requests Fail with 404

**Symptoms**:
```
GET http://PUBLIC-IP/api/user/signup 404
```

**Causes**:

1. **Wrong nginx configuration**
   ```bash
   cat nginx/nginx.conf | grep "location /api"
   ```
   
   **Should contain**:
   ```nginx
   location /api {
       proxy_pass http://backend;
   }
   ```

2. **Backend wrong port**
   - Backend should expose port 80 (not 3000)
   - Check backend Dockerfile: `EXPOSE 80`

3. **Backend route missing**
   - Verify route exists in backend code

---

## Database Issues

### MongoDB Connection Refused

**Symptoms**:
```
MongoNetworkError: failed to connect to server [mongo:27017]
```

**Solutions**:

1. **Check mongo container**
   ```bash
   sudo docker-compose ps mongo
   # Should show "Up"
   ```

2. **Verify network connectivity**
   ```bash
   sudo docker exec backend_1 ping mongo
   ```

3. **Check connection string**
   ```bash
   cat backend/Project/config/config.json
   ```
   
   **Should be**:
   ```json
   {
     "mongoURI": "mongodb://mongo:27017/meanapp"
   }
   ```
   
   **NOT**: `localhost` or `127.0.0.1`

4. **Restart containers**
   ```bash
   sudo docker-compose restart mongo backend
   ```

### Data Not Persisting

**Symptoms**:
- Create user
- Restart containers
- User data gone

**Cause**: Volume not configured

**Solution**: Verify docker-compose.yml:
```yaml
services:
  mongo:
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

**Recreate volume**:
```bash
sudo docker-compose down
sudo docker volume ls
sudo docker-compose up -d
```

### Cannot Access MongoDB Shell

**Access mongo shell**:
```bash
sudo docker exec -it meanapp_mongo_1 mongo
```

**List databases**:
```javascript
show dbs
use meanapp
show collections
db.users.find()
```

**Exit**:
```javascript
exit
```

---

## Performance Issues

### Site Loads Slowly

**Diagnosis**:

1. **Check container resources**
   ```bash
   docker stats
   ```

2. **Check CPU/memory on host**
   ```bash
   top
   free -m
   ```

3. **Check network latency**
   ```bash
   ping YOUR-EC2-IP
   ```

**Solutions**:

1. **Add swap** (see EC2 section)
2. **Optimize images**
3. **Enable caching** in Nginx
4. **Upgrade instance type**

### High Memory Usage

**Solutions**:

1. **Limit container memory**
   ```yaml
   # docker-compose.yml
   services:
     frontend:
       deploy:
         resources:
           limits:
             memory: 256M
   ```

2. **Add swap space** (see EC2 section)

3. **Reduce container count**
   ```bash
   sudo docker-compose up -d --scale frontend=2
   ```

---

## Quick Diagnostics Commands

```bash
# Overall health check
sudo docker-compose ps
sudo docker-compose logs --tail 50

# Network connectivity
sudo docker exec frontend_1 ping backend
sudo docker exec backend_1 ping mongo

# Resource usage
docker stats --no-stream

# Disk space
df -h

# System resources
free -m
top

# Check ports
sudo lsof -i :80
sudo netstat -tulpn | grep 80

# Verify DNS resolution
sudo docker exec nginx_1 nslookup frontend

# Test local access
curl -I http://localhost
```

## Still Having Issues?

1. **Full restart**:
   ```bash
   sudo docker-compose down
   sudo docker-compose build --no-cache
   sudo docker-compose up -d --scale frontend=3
   ```

2. **Check all logs**:
   ```bash
   sudo docker-compose logs > ~/debug-logs.txt
   cat ~/debug-logs.txt
   ```

3. **Verify all components**:
   ```bash
   # Network
   sudo docker network ls
   
   # Volumes
   sudo docker volume ls
   
   # Images
   sudo docker images
   ```

4. **Nuclear option** (deletes everything):
   ```bash
   sudo docker-compose down -v
   sudo docker system prune -a
   git pull
   sudo docker-compose up -d --scale frontend=3
   ```

---

**Remember**: Always check logs first with `sudo docker-compose logs [service]`
