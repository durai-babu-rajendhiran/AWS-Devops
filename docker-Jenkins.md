# MERN Stack Docker & Jenkins CI/CD Deployment Guide

A step-by-step guide to containerizing, pushing, and deploying a MERN (MongoDB/Express/React/Node) stack application from a local environment to an AWS EC2 instance using Docker Hub and Jenkins.

---

## Technical Overview & Architecture

```
+------------------+         +------------------+         +------------------+
|  Local Machine   |  Push   |    Docker Hub    |  Pull   |  AWS EC2 Host    |
|                  | ------> | duraibabu200/    | ------> |  (Ubuntu Linux)  |
| - React Frontend |         |  mern-frontend   |         |                  |
| - Node.js API    |         | duraibabu200/    |         | - Host Nginx     |
| - Docker Engine  |         |  mern-backend    |         | - Jenkins CI/CD  |
+------------------+         +------------------+         +------------------+
```

---

## Phase 1: Local Development & Configuration

### 1. Structure Your `docker-compose.yml`

To ensure local builds automatically apply your Docker Hub tags, update your local `docker-compose.yml`:

```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    image: duraibabu200/mern-backend:latest
    container_name: mern_backend
    ports:
      - "3000:3000"
    networks:
      - mern_network

  frontend:
    build: ./frontend
    image: duraibabu200/mern-frontend:latest
    container_name: mern_frontend
    ports:
      - "3001:5173"
    depends_on:
      - backend
    networks:
      - mern_network

networks:
  mern_network:
    driver: bridge
```

### 2. Build, Tag, and Push Locally

Run these commands in your local project root folder (`C:\Users\Durai Babu\OneDrive\Documents\jenkins`):

```bash
# 1. Build local container images
docker-compose up -d --build

# 2. Login to Docker Hub
docker login

# 3. Tag images explicitly (if not using the image tag in docker-compose)
docker tag jenkins-backend duraibabu200/mern-backend:latest
docker tag jenkins-frontend duraibabu200/mern-frontend:latest

# 4. Push images to Docker Hub registry
docker push duraibabu200/mern-backend:latest
docker push duraibabu200/mern-frontend:latest
```

---

## Phase 2: AWS EC2 Server Setup

### 1. Update Inbound Security Group Rules (AWS Console)
Go to **AWS EC2 Console > Instances > Security Groups > Edit Inbound Rules** and add:
- **HTTP (Port 80):** `0.0.0.0/0`
- **HTTPS (Port 443):** `0.0.0.0/0`
- **Jenkins (Port 8080):** `0.0.0.0/0`
- **SSH (Port 22):** `Your IP`

### 2. Connect via SSH
```bash
ssh -i /path/to/your-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>
```

### 3. Install Docker Engine, Nginx, and Jenkins
Run the following script on your EC2 Ubuntu terminal:

```bash
# Update base system packages
sudo apt update && sudo apt upgrade -y

# Install Docker Engine, Docker Compose, Nginx, and OpenJDK (for Jenkins)
sudo apt install -y docker.io docker-compose-v2 nginx fontconfig openjdk-17-jre curl

# Install Jenkins
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins

# Enable and start services
sudo systemctl enable --now docker
sudo systemctl enable --now jenkins
sudo systemctl enable --now nginx
```

### 4. Configure User Permissions (CRITICAL)
Add both `ubuntu` and `jenkins` system users to the `docker` group to grant privileges without needing `sudo`:

```bash
# Add users to docker group
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins

# Grant explicit permissions to Docker socket
sudo chmod 666 /var/run/docker.sock

# Restart Jenkins to apply group membership changes
sudo systemctl restart jenkins
```

---

## Phase 3: Host Nginx Reverse Proxy Setup

Configure Nginx on the EC2 host to route incoming HTTP traffic on port 80 to the containers.

1. Create a configuration file:
   ```bash
   sudo nano /etc/nginx/sites-available/mern-app
   ```

2. Add the configuration content:
   ```nginx
   server {
       listen 80;
       server_name _;

       # Route web traffic to Frontend container
       location / {
           proxy_pass http://127.0.0.1:5173;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

       # Route API requests to Backend container
       location /api/ {
           proxy_pass http://127.0.0.1:3000/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

3. Enable the configuration and restart Nginx:
   ```bash
   sudo rm -f /etc/nginx/sites-enabled/default
   sudo ln -s /etc/nginx/sites-available/mern-app /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

## Phase 4: Jenkins CI/CD Job Setup

1. Open Jenkins in your web browser:
   `http://<YOUR_EC2_PUBLIC_IP>:8080`
2. Get initial admin password from EC2 terminal:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Create a **Freestyle Project** named `mern-deployment`.
4. Under **Build Steps**, select **Execute shell**.
5. Paste the automated deployment script below:

```bash
#!/bin/bash
set -e

echo "=== Starting MERN Stack Container Deployment ==="

# 1. Create Docker internal bridge network if missing
docker network create mern_network || true

# 2. Stop and remove legacy container instances
echo "Stopping existing containers..."
docker rm -f mern_frontend || true
docker rm -f mern_backend || true

# 3. Pull latest builds from Docker Hub
echo "Pulling latest docker images from Docker Hub..."
docker pull duraibabu200/mern-backend:latest
docker pull duraibabu200/mern-frontend:latest

# 4. Spin up Backend container
echo "Starting Backend container..."
docker run -d \
  --name mern_backend \
  --network mern_network \
  -p 3000:3000 \
  --restart always \
  duraibabu200/mern-backend:latest

# 5. Spin up Frontend container
echo "Starting Frontend container..."
docker run -d \
  --name mern_frontend \
  --network mern_network \
  -p 5173:5173 \
  --restart always \
  duraibabu200/mern-frontend:latest

echo "=== Deployment Successfully Completed ==="
```

---

## Phase 5: Verification & Verification Commands

To verify that all services are operational on your EC2 instance, execute these diagnostic checks:

```bash
# Check container status (Both status columns must show "Up")
docker ps

# Stream Backend logs
docker logs -f mern_backend

# Stream Frontend logs
docker logs -f mern_frontend

# Test internal Nginx connectivity
curl -I http://127.0.0.1:5173
curl -I http://127.0.0.1:3000
```

---

## Common Issues & Solutions Quick Reference

| Error / Symptom | Root Cause | Solution |
| :--- | :--- | :--- |
| `permission denied ... docker.sock` | `jenkins` user lacks permission to access Docker socket | Run `sudo usermod -aG docker jenkins && sudo chmod 666 /var/run/docker.sock && sudo systemctl restart jenkins` on EC2 SSH terminal |
| `502 Bad Gateway` | Frontend container is stopped, crashed, or mapped to wrong host port | Verify container is running with `docker ps`, ensure host port is `5173`, and check container logs with `docker logs mern_frontend` |
| Code changes not reflecting | Local rebuild didn't update tags pushed to Docker Hub | Ensure local `docker tag` commands run before `docker push` or define explicit `image:` fields in local `docker-compose.yml` |
