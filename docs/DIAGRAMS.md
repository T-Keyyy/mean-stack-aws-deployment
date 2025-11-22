# Architecture Diagrams

Visual representations of the MEAN stack deployment architecture.

## System Architecture Overview

```
                                    INTERNET
                                       │
                                       │
                                       ▼
                        ┌──────────────────────────┐
                        │     AWS EC2 Instance     │
                        │      (t2.micro)          │
                        │   Ubuntu 24.04 LTS       │
                        │  Public IP: x.x.x.x      │
                        └──────────┬───────────────┘
                                   │
                                   │ Port 80
                                   ▼
                        ┌──────────────────────────┐
                        │   Nginx Load Balancer    │
                        │   (Docker Container)     │
                        │  - Reverse Proxy         │
                        │  - Load Balancing        │
                        │  - Health Checks         │
                        └──────────┬───────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
          ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
          │  Frontend 1 │ │  Frontend 2 │ │  Frontend 3 │
          │  (Angular)  │ │  (Angular)  │ │  (Angular)  │
          │   Nginx     │ │   Nginx     │ │   Nginx     │
          │   Port 80   │ │   Port 80   │ │   Port 80   │
          └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                 │                │                │
                 └────────────────┼────────────────┘
                                  │
                                  │ /api requests
                                  ▼
                       ┌──────────────────────┐
                       │  Backend API Server  │
                       │  (Node.js/Express)   │
                       │     Port 80          │
                       └──────────┬───────────┘
                                  │
                                  │ MongoDB Protocol
                                  ▼
                       ┌──────────────────────┐
                       │   MongoDB 4.4        │
                       │   Database           │
                       │   Port 27017         │
                       │   Persistent Volume  │
                       └──────────────────────┘
```

## Docker Network Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Host (EC2)                        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Custom Bridge Network: "meanapp"                │ │
│  │              DNS: 127.0.0.11                            │ │
│  │                                                         │ │
│  │   ┌──────────┐      ┌──────────┐      ┌──────────┐    │ │
│  │   │ Nginx    │      │Frontend 1│      │Frontend 2│    │ │
│  │   │ (Port    │──────│ (Port 80)│      │ (Port 80)│    │ │
│  │   │ 80->80)  │      └──────────┘      └──────────┘    │ │
│  │   └────┬─────┘              │                │         │ │
│  │        │                    │                │         │ │
│  │        │      ┌──────────┐  │     ┌──────────┐        │ │
│  │        └──────│Frontend 3│──┴─────│ Backend  │        │ │
│  │               │ (Port 80)│        │ (Port 80)│        │ │
│  │               └──────────┘        └────┬─────┘        │ │
│  │                                        │              │ │
│  │                                 ┌──────────┐          │ │
│  │                                 │ MongoDB  │          │ │
│  │                                 │(Port     │          │ │
│  │                                 │ 27017)   │          │ │
│  │                                 └──────────┘          │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─────────────┐                                            │
│  │   Volume    │                                            │
│  │ mongodb_data│ ← Mounted to /data/db in MongoDB          │
│  └─────────────┘                                            │
└──────────────────────────────────────────────────────────────┘
                          │
                          │ Port Mappings
                          ▼
                    Host Port 80 → Nginx Container Port 80
```

## Request Flow Diagram

### User Request Flow

```
1. User Browser
   │
   │ HTTP GET http://x.x.x.x/
   ▼
2. AWS Security Group
   │ (Allow port 80)
   ▼
3. EC2 Instance
   │
   │ Port 80
   ▼
4. Nginx Container
   │
   │ Check route: / (not /api)
   │ Round-robin to frontend cluster
   ▼
5. Frontend Container (1, 2, or 3)
   │
   │ Serve index.html
   │ Angular app loads
   ▼
6. User Browser
   │ Renders Angular SPA
   │
   │ User fills signup form
   │ Submit
   ▼
7. Angular App
   │
   │ POST http://x.x.x.x/api/user/signup
   │ {name, email, password, phone}
   ▼
8. Nginx Container
   │
   │ Check route: /api
   │ Proxy to backend
   ▼
9. Backend Container
   │
   │ Express route: /api/user/signup
   │ Validate data
   ▼
10. MongoDB Container
    │
    │ db.users.insertOne({...})
    │ Save user
    ▼
11. Response chain back to browser
    Backend → Nginx → Frontend → Browser
    │
    ▼ SUCCESS: User registered!
```

## Load Balancing Flow

```
Request 1 ──┐
Request 2 ──┤
Request 3 ──┤──→  Nginx Load Balancer
Request 4 ──┤        │ (least_conn algorithm)
Request 5 ──┤        │
Request 6 ──┘        │
                     ├─→ Frontend 1 (2 active connections)
                     ├─→ Frontend 2 (1 active connection) ✓ Selected
                     └─→ Frontend 3 (3 active connections)
```

## High Availability Scenario

### Normal Operation

```
                     Nginx
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   Frontend 1     Frontend 2     Frontend 3
   [HEALTHY]      [HEALTHY]      [HEALTHY]
   
   Traffic: 33%   Traffic: 33%   Traffic: 34%
```

### Frontend 1 Fails

```
                     Nginx
                       │
                       │ (Detects failure via health check)
        ┌──────────────┼──────────────┐
        │              │              │
        ✗              ▼              ▼
   Frontend 1     Frontend 2     Frontend 3
   [FAILED]       [HEALTHY]      [HEALTHY]
   
   Traffic: 0%    Traffic: 50%   Traffic: 50%
   
   ⚠️ No user-visible downtime!
```

### Recovery

```
                     Nginx
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   Frontend 1     Frontend 2     Frontend 3
   [RECOVERED]    [HEALTHY]      [HEALTHY]
   
   Traffic: 33%   Traffic: 33%   Traffic: 34%
   
   ✓ Full capacity restored
```

## Deployment Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                   Local Development                         │
│                                                              │
│  1. Code changes in VS Code                                 │
│  2. Test locally: docker-compose up                         │
│  3. Verify at http://localhost                              │
│  4. Commit to Git                                           │
│  5. Push to GitHub                                          │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       │ git push
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Repository                        │
│                                                              │
│  - Source code versioned                                    │
│  - Dockerfiles                                              │
│  - docker-compose.yml                                       │
│  - Configuration files                                      │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       │ git pull (manual)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    AWS EC2 Instance                         │
│                                                              │
│  1. SSH into server                                         │
│  2. cd mean-stack-aws-deployment                            │
│  3. git pull                                                │
│  4. docker-compose build                                    │
│  5. docker-compose up -d --scale frontend=3                 │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       │ Live!
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Production Site                          │
│                                                              │
│  http://PUBLIC-IP                                           │
│  - 6 containers running                                     │
│  - Load balanced                                            │
│  - High availability                                        │
└─────────────────────────────────────────────────────────────┘
```

## Scaling Architecture

### Current State (3 Frontends)

```
Users: ~100 concurrent
┌──────┐
│Nginx │
└───┬──┘
    ├──→ F1 (33 users)
    ├──→ F2 (33 users)
    └──→ F3 (34 users)
```

### Scaled Up (5 Frontends)

```
Users: ~500 concurrent
┌──────┐
│Nginx │
└───┬──┘
    ├──→ F1 (100 users)
    ├──→ F2 (100 users)
    ├──→ F3 (100 users)
    ├──→ F4 (100 users)
    └──→ F5 (100 users)

Command: docker-compose up -d --scale frontend=5
```

## Security Layers

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              AWS Security Group (Firewall)                  │
│  - Allow: SSH (22) from My IP                               │
│  - Allow: HTTP (80) from 0.0.0.0/0                          │
│  - Allow: HTTPS (443) from 0.0.0.0/0                        │
│  - Block: All other ports                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  EC2 Instance (Ubuntu)                      │
│  - UFW Firewall (optional)                                  │
│  - SSH Key Authentication Only                              │
│  - No password authentication                               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Docker Network Isolation                       │
│  - Containers in private network                            │
│  - Only Nginx exposed to host port 80                       │
│  - Backend/MongoDB not directly accessible                  │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow

### Write Operation (User Registration)

```
Browser                Nginx               Frontend            Backend             MongoDB
   │                     │                    │                   │                   │
   │  POST /api/signup   │                    │                   │                   │
   │────────────────────>│                    │                   │                   │
   │                     │                    │                   │                   │
   │                     │  Forward to        │                   │                   │
   │                     │  backend           │                   │                   │
   │                     │───────────────────────────────────────>│                   │
   │                     │                    │                   │                   │
   │                     │                    │                   │  db.users.insert  │
   │                     │                    │                   │──────────────────>│
   │                     │                    │                   │                   │
   │                     │                    │                   │  {acknowledged:   │
   │                     │                    │                   │<──────────────────│
   │                     │                    │                   │   true}           │
   │                     │                    │                   │                   │
   │                     │  {success: true}   │                   │                   │
   │<────────────────────────────────────────────────────────────│                   │
   │                     │                    │                   │                   │
```

### Read Operation (Static Files)

```
Browser                Nginx               Frontend 
   │                     │                    │
   │  GET /signup        │                    │
   │────────────────────>│                    │
   │                     │                    │
   │                     │  Round-robin       │
   │                     │  to Frontend 2     │
   │                     │───────────────────>│
   │                     │                    │
   │                     │  index.html        │
   │<────────────────────────────────────────│
   │  (Angular SPA)      │                    │
   │                     │                    │
```

---

## Container Relationships

```
┌──────────────┐
│   nginx      │ Depends on: frontend, backend
└──────┬───────┘
       │
       ├─────────┐
       │         │
       ▼         ▼
┌──────────┐ ┌──────────┐
│ frontend │ │ backend  │ Depends on: mongo
└──────────┘ └────┬─────┘
                  │
                  ▼
             ┌──────────┐
             │  mongo   │ No dependencies
             └──────────┘
```

**Start order** (automatically handled by docker-compose):
1. mongo
2. backend (waits for mongo)
3. frontend (independent)
4. nginx (waits for frontend and backend)

---

These diagrams illustrate the complete architecture from infrastructure to application flow. All components work together to provide a highly available, load-balanced MEAN stack application.
