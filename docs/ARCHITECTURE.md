# Architecture Documentation

## System Overview

This document provides detailed technical architecture information for the MEAN stack deployment.

## Component Architecture

### 1. Nginx Load Balancer

**Purpose**: Reverse proxy and load balancing for frontend replicas

**Configuration Highlights**:
```nginx
upstream frontend {
    least_conn;
    server frontend:80 max_fails=3 fail_timeout=30s;
}

resolver 127.0.0.11 valid=5s;
set $frontend_upstream http://frontend;
proxy_pass $frontend_upstream;
```

**Key Features**:
- **Dynamic DNS Resolution**: Resolves container names every 5 seconds
- **Least Connections**: Routes to replica with fewest active connections
- **Health Checks**: Marks containers as down after 3 failed requests
- **Failover**: Automatically removes unhealthy containers from pool

**Why Variable-Based proxy_pass?**
- Forces Nginx to use resolver on every request
- Without variable, Nginx caches DNS at startup
- Critical for handling container restarts/failures

### 2. Frontend Cluster (Angular 6)

**Replicas**: 3 instances for high availability

**Technology Stack**:
- Angular 6.0.3
- TypeScript 2.7.2
- Nginx (Alpine-based)

**Multi-Stage Build**:
```dockerfile
# Stage 1: Build
FROM node:10-alpine as builder
WORKDIR /app
COPY ./Angular6/package*.json ./
RUN npm install
COPY ./Angular6 .
RUN npm run build

# Stage 2: Serve
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**Benefits**:
- Small image size (~25MB vs 500MB+)
- Production-ready static files
- Fast container startup
- Reduced attack surface

**Client-Side Routing**:
```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```
- Handles Angular routes (e.g., /signup, /login)
- Prevents 404 on page refresh
- Falls back to index.html for SPA routing

### 3. Backend API (Node.js/Express)

**Technology Stack**:
- Node.js 10 (compatible with legacy dependencies)
- Express.js
- Mongoose ODM

**Endpoints**:
- `POST /api/user/signup` - User registration
- `POST /api/user/signin` - User authentication
- Health check at root

**Configuration**:
```json
{
  "mongoURI": "mongodb://mongo:27017/meanapp"
}
```

**Environment**: Production mode with optimized settings

### 4. MongoDB Database

**Version**: 4.4 (stable, production-ready)

**Storage**:
- Persistent volume: `mongodb_data`
- Location: `/data/db` in container
- Survives container restarts

**Access**:
- Internal only (no external exposure)
- Connected via Docker network
- Service name: `mongo`

## Network Architecture

### Docker Network: `meanapp`

**Type**: Custom bridge network

**Benefits**:
- Service discovery via container names
- DNS resolution (127.0.0.11)
- Isolated from other Docker networks
- Automatic container-to-container communication

**Service Communication**:
```
nginx → frontend (http://frontend:80)
frontend → backend (http://backend:80/api)
backend → mongo (mongodb://mongo:27017)
```

## Scaling Strategy

### Horizontal Scaling (Frontend)

```bash
docker-compose up -d --scale frontend=3
```

**How it works**:
1. Docker creates 3 frontend containers
2. Each assigned unique name: `frontend_1`, `frontend_2`, `frontend_3`
3. Nginx resolves `frontend` to all 3 IPs
4. Load balancer distributes requests

**Limitations**:
- Only frontend scales (stateless)
- Backend single instance (can be scaled with session management)
- MongoDB single instance (clustering requires replica sets)

### Vertical Scaling

Increase resources in docker-compose.yml:
```yaml
deploy:
  resources:
    limits:
      cpus: '0.50'
      memory: 512M
```

## High Availability Design

### Failure Scenarios

| Scenario | Impact | Recovery |
|----------|--------|----------|
| 1 frontend fails | No downtime | Nginx routes to 2 remaining |
| 2 frontends fail | No downtime | Nginx routes to 1 remaining |
| All frontends fail | Downtime | Manual restart required |
| Backend fails | Full downtime | Manual restart required |
| MongoDB fails | Full downtime | Data persists, restart required |
| Nginx fails | Full downtime | Entry point, restart required |

### Failover Testing Results

**Test**: Stop `frontend_1` container

**Timeline**:
- T+0s: Container stopped
- T+5s: Nginx DNS refresh detects failure
- T+5s: Requests route to frontend_2 and frontend_3
- T+0s: No user-visible downtime

**Configuration**:
```nginx
resolver 127.0.0.11 valid=5s;
set $frontend_upstream http://frontend;
proxy_pass $frontend_upstream;
```
- DNS re-resolved every 5 seconds
- Variable-based proxy_pass forces fresh lookups
- Automatic failover via Docker's service discovery

## Security Architecture

### Network Isolation

**External Access**:
- Port 80 (HTTP) → Nginx only
- Port 443 (HTTPS) → Not configured (future)

**Internal Only**:
- Backend: Port 80 (accessible only via Nginx)
- MongoDB: Port 27017 (no external access)
- Frontend replicas: Port 80 (via load balancer only)

### AWS Security Groups

**Inbound Rules**:
- SSH (22): Your IP only
- HTTP (80): 0.0.0.0/0 (public access)
- HTTPS (443): 0.0.0.0/0 (future SSL)

**Outbound Rules**:
- All traffic allowed (for package updates)

### Recommended Improvements

1. **HTTPS/SSL**:
   - Use Let's Encrypt for free SSL
   - Configure Nginx for HTTPS redirect
   - Update Security Groups

2. **Secrets Management**:
   - Use AWS Secrets Manager
   - Environment variables for sensitive data
   - Never commit credentials to Git

3. **Database Security**:
   - Enable MongoDB authentication
   - Create admin and app-specific users
   - Rotate credentials regularly

4. **Application Security**:
   - Input validation on backend
   - Rate limiting on Nginx
   - CORS configuration
   - XSS/CSRF protection

## Performance Optimization

### Current Optimizations

1. **Multi-Stage Builds**: Reduced image size by 95%
2. **Alpine Images**: Smaller attack surface, faster downloads
3. **Load Balancing**: Distributes load across 3 frontends
4. **Static File Serving**: Nginx serves Angular files efficiently

### Recommended Improvements

1. **Caching**:
   - Add Redis for session storage
   - Configure browser caching headers
   - Enable Nginx proxy caching

2. **CDN**:
   - CloudFront for static assets
   - Reduces EC2 bandwidth costs
   - Improves global latency

3. **Database**:
   - Add indexes for frequent queries
   - Connection pooling
   - Query optimization

4. **Monitoring**:
   - AWS CloudWatch for metrics
   - ELK stack for log aggregation
   - Prometheus/Grafana for dashboards

## Deployment Pipeline

### Current Process (Manual)

1. Code changes committed to GitHub
2. SSH into EC2 instance
3. Pull latest changes
4. Rebuild containers
5. Restart services

### Recommended CI/CD

```yaml
# GitHub Actions example
name: Deploy to AWS
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to EC2
        run: |
          ssh ${{ secrets.EC2_HOST }}
          cd mean-stack-aws-deployment
          git pull
          docker-compose build
          docker-compose up -d --scale frontend=3
```

## Disaster Recovery

### Backup Strategy

**Database Backups**:
```bash
# Create backup
docker exec mongodb mongodump --out /backup

# Restore backup
docker exec mongodb mongorestore /backup
```

**Configuration Backup**:
- All configs in Git (version controlled)
- EC2 snapshot for full system backup

### Recovery Procedures

**Container Failure**:
```bash
docker-compose restart SERVICE_NAME
```

**Complete System Failure**:
```bash
# Launch new EC2
# Install Docker
# Clone repository
docker-compose up -d --scale frontend=3
# Restore MongoDB backup
```

**Data Loss Prevention**:
- MongoDB volume persists across container restarts
- Regular database exports to S3
- Version control for all configuration

## Cost Analysis

### AWS Free Tier Usage

**EC2 t2.micro**:
- 750 hours/month (free)
- 1 vCPU, 1GB RAM
- Sufficient for demo/development

**EBS Storage**:
- 30GB free tier
- Current usage: ~8GB
- Well within limits

**Data Transfer**:
- 15GB outbound free
- Typical usage: <5GB/month

### Estimated Costs (Beyond Free Tier)

- t2.micro: ~$8.50/month
- 30GB EBS: ~$3/month
- Data transfer: ~$0.09/GB
- **Total**: ~$12-15/month

### Cost Optimization

1. **Stop when not needed**: Save ~60% on compute
2. **Reserved instances**: Save ~40% for 1-year commitment
3. **Right-sizing**: Monitor and adjust instance type
4. **Spot instances**: Save ~70% for non-critical workloads

## Monitoring and Observability

### Container Logs

```bash
# View logs
docker-compose logs -f SERVICE_NAME

# Last 100 lines
docker-compose logs --tail 100
```

### Resource Monitoring

```bash
# Container stats
docker stats

# System resources
top
df -h
free -m
```

### Recommended Tools

- **AWS CloudWatch**: System metrics, alarms
- **Datadog**: APM, distributed tracing
- **Sentry**: Error tracking
- **New Relic**: Performance monitoring

## Scalability Roadmap

### Phase 1: Current State
- 3 frontend replicas
- Single backend
- Single MongoDB

### Phase 2: Backend Scaling
- Multiple backend instances
- Session storage in Redis
- Sticky sessions or JWT tokens

### Phase 3: Database Scaling
- MongoDB replica set (3 nodes)
- Read replicas
- Automatic failover

### Phase 4: Multi-Region
- Route53 DNS routing
- EC2 instances in multiple regions
- Data replication across regions
- CloudFront CDN

## Technology Decisions

### Why Node.js 10?

**Reason**: Angular 6 compatibility
- Angular 6 requires Node 8.x or 10.x
- Newer Node versions break dependencies
- Pinned to ensure consistent builds

### Why Nginx (not Express static)?

**Benefits**:
- Faster static file serving
- Better resource utilization
- Production-grade features
- Easier to configure caching

### Why Docker Compose (not Kubernetes)?

**For this scale**:
- Simpler configuration
- Faster setup
- Cost-effective for small deployments
- Suitable for single-server deployments

**Kubernetes recommended when**:
- Multiple servers (cluster)
- Complex orchestration needs
- Auto-scaling requirements
- Enterprise-grade HA

## References

- [Nginx Documentation](https://nginx.org/en/docs/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [MongoDB Manual](https://docs.mongodb.com/manual/)
- [AWS EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [Angular Deployment Guide](https://angular.io/guide/deployment)
