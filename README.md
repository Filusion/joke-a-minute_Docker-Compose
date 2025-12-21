# Joke-a-Minute - Docker Compose Deployment

A production-ready Flask web application that delivers jokes every minute, deployed using Docker Compose with a complete microservices architecture. This setup includes MySQL for data persistence, Redis for caching, Nginx as a reverse proxy, and automated SSL/TLS certificate management with Let's Encrypt.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Step 1: Install Docker](#step-1-install-docker)
  - [Step 2: Clone the Repository](#step-2-clone-the-repository)
- [Configuration Files Explained](#configuration-files-explained)
  - [Dockerfile](#dockerfile)
  - [docker-compose.test.yml](#docker-composetestyml)
  - [docker-compose.yml](#docker-composeyml)
  - [Nginx Configuration](#nginx-configuration)
- [Deployment](#deployment)
- [SSL/TLS Setup](#ssltls-setup)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

This application uses a multi-container Docker architecture with the following components:

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ HTTPS/HTTP
       ▼
┌─────────────┐
│    Nginx    │ ◄── SSL/TLS Termination
└──────┬──────┘
       │
       │ HTTP (Internal)
       ▼
┌─────────────┐
│  Flask App  │ ◄── Application Logic (Gunicorn)
└──┬────────┬─┘
   │        │
   │        └────────┐
   ▼                 ▼
┌────────┐      ┌────────┐
│  MySQL │      │  Redis │
└────────┘      └────────┘
 Database        Cache
```

**Component Breakdown:**

- **Flask Application**: The core application logic serving jokes, built with Python 3.11 and served by Gunicorn
- **MySQL 8.0**: Relational database storing jokes and user data
- **Redis 7**: In-memory cache for high-performance data access and session management
- **Nginx**: Reverse proxy handling SSL termination, load balancing, and static file serving
- **Certbot**: Automated SSL certificate provisioning and renewal from Let's Encrypt

## Prerequisites

Before you begin, ensure you have:

- A Linux-based server (Ubuntu 20.04+ recommended)
- Root or sudo access
- A domain name pointing to your server's IP address (for SSL setup)
- At least 2GB of RAM
- 10GB of available disk space

## Installation

### Step 1: Install Docker

Docker is the containerization platform that runs our entire application stack. Follow these steps to install it:

#### 1.1 Ensure curl is installed

```bash
sudo apt update
sudo apt install -y curl
```

Curl is necessary for downloading the Docker installation script.

#### 1.2 Install Docker using the official convenience script

```bash
curl -fsSL https://get.docker.com/ | sh
```

This script automatically detects your Linux distribution and installs the appropriate Docker version. While this method is sometimes called "dirty" because it's less customizable, it's the fastest and most reliable way to get Docker up and running.

#### 1.3 Enable Docker without sudo (REQUIRED)

By default, Docker requires root privileges. To use Docker as a regular user, add yourself to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

**IMPORTANT:** Group membership changes require a new login session. You must either:

- Log out and log back in, OR
- Reboot the system, OR
- Run the following command to activate the group in your current session:

```bash
newgrp docker
```

The `newgrp` command starts a new shell where your primary group is temporarily set to `docker`. This is necessary because group membership is loaded at login time, not dynamically updated.

#### 1.4 Verify Docker installation

Run the following command to check if Docker is properly installed:

```bash
docker version
```

You should see both **Client** and **Server** version information. If both appear, Docker is correctly installed and running.

Test that Docker works without sudo:

```bash
docker ps
```

If this command returns a list (even if empty) without permission errors, you're all set. If you see "Permission denied", run `newgrp docker` as mentioned in step 1.3.

### Step 2: Clone the Repository

Now that Docker is installed, let's get the application code.

#### 2.1 Install Git (if not already installed)

```bash
sudo apt install -y git
```

Git is essential for version control and cloning repositories from GitHub.

#### 2.2 Clone the repository

```bash
git clone https://github.com/Filusion/joke-a-minute_Docker-Compose.git
cd joke-a-minute_Docker-Compose
```

This creates a directory with all the necessary configuration files and application code.

#### 2.3 Verify the directory structure

After cloning, your project structure should look like this:

```
joke-a-minute_Docker-Compose/
├── app/
│   ├── Dockerfile
│   ├── app.py
│   ├── init_db.py
│   └── requirements.txt
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── app.conf
├── docker-compose.yml
├── docker-compose.test.yml
└── README.md
```

## Configuration Files Explained

### Dockerfile

**Location:** `app/Dockerfile`

The Dockerfile defines how the Flask application container is built. It uses a multi-stage build process to create a lean, secure production image.

#### Stage 1: Builder Stage

```dockerfile
FROM python:3.11-slim AS builder
```

This stage uses Python 3.11 slim image as a base and installs all build dependencies:

- **gcc** and **g++**: C/C++ compilers needed for building Python packages with native extensions
- **python3-dev**: Python development headers
- **libmariadb-dev**: Development libraries for MySQL/MariaDB connectivity
- **pkg-config**: Helper tool for compiling applications and libraries

The builder stage installs all Python dependencies from `requirements.txt` into a temporary `/install` directory. This includes Flask, Gunicorn (production WSGI server), MySQL connector, Redis client, and other application dependencies.

**Key advantage:** By separating build dependencies from runtime, we significantly reduce the final image size. Build tools like compilers aren't needed in production, so they're discarded after compilation.

#### Stage 2: Runtime Stage

```dockerfile
FROM python:3.11-slim
```

This stage creates the final minimal image with only runtime dependencies:

**Security hardening:**
- Creates a non-root user (`appuser`) with UID 1000
- All application files are owned by this non-root user
- The container runs as `appuser`, not root (best security practice)

**Runtime dependencies:**
- **libmariadb3**: Runtime library for MySQL connections (no development headers)
- **curl**: Used by the health check endpoint

**Key files copied:**
- Python packages from the builder stage (`/install/lib/python3.11/site-packages`)
- Binary executables like `gunicorn` from the builder stage (`/install/bin/*`)
- Application code (`app.py`, `init_db.py`)

**Health Check:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1
```

Docker periodically checks if the application is healthy by hitting the `/health` endpoint:
- **interval**: Check every 30 seconds
- **timeout**: Wait 10 seconds for a response
- **start-period**: Give the app 40 seconds to start before beginning health checks
- **retries**: Mark as unhealthy after 3 consecutive failures

**Production server:**
```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--timeout", "60", "app:app"]
```

Gunicorn is a production-grade WSGI HTTP server that:
- Binds to all network interfaces on port 5000
- Spawns 2 worker processes to handle concurrent requests
- Sets a 60-second timeout for long-running requests
- Runs the Flask application defined in `app.py`

### docker-compose.test.yml

**Purpose:** This is a simplified testing configuration that allows you to quickly spin up the application with its dependencies **without** the Nginx reverse proxy and SSL complexity.

**Use case:** Use this file during development or when testing changes to the application, database schema, or Redis caching logic.

#### Service: MySQL

```yaml
mysql:
  image: mysql:8.0
  environment:
    MYSQL_ROOT_PASSWORD: rootpass123
    MYSQL_DATABASE: jokes_db
    MYSQL_USER: joke_user
    MYSQL_PASSWORD: joke_pass123
  ports:
    - "3306:3306"
```

- Uses the official MySQL 8.0 image from Docker Hub
- Automatically creates a database named `jokes_db`
- Creates a user `joke_user` with password `joke_pass123`
- **Port mapping:** Exposes MySQL on the host's port 3306 (useful for direct database access with tools like MySQL Workbench)

**Health check:**
```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s
  timeout: 5s
  retries: 5
```

Docker waits for MySQL to be fully ready before starting dependent services. The health check pings MySQL every 10 seconds until it responds.

**Volume:** `mysql_data:/var/lib/mysql` ensures data persists even if the container is removed.

#### Service: Redis

```yaml
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
```

- Uses Redis 7 based on Alpine Linux (very small image)
- Exposes Redis on port 6379 for direct access during testing
- Health check ensures Redis is responsive before the app starts

**Volume:** `redis_data:/data` persists cached data across container restarts.

#### Service: App

```yaml
app:
  build:
    context: ./app
    dockerfile: Dockerfile
  environment:
    DB_HOST: mysql
    DB_USER: joke_user
    DB_PASSWORD: joke_pass123
    # ... other environment variables
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
```

- Builds the Flask application from the Dockerfile in `./app`
- Environment variables configure database and Redis connections
- **depends_on with condition:** Ensures MySQL and Redis are healthy before starting the app
- Port 5000 is exposed directly to the host for testing

**Why use this file?**
- Faster startup (no Nginx or SSL overhead)
- Direct access to all ports for debugging
- Simpler configuration for development

**How to use:**
```bash
docker compose -f docker-compose.test.yml up --build
```

Visit `http://localhost:5000` to access the application directly.

### docker-compose.yml

**Purpose:** This is the production-ready configuration with all four services, including the Nginx reverse proxy and SSL/TLS support via Let's Encrypt.

#### Key Differences from Test Configuration

1. **Network isolation:** All services communicate on a private `backend` network, not exposed to the host
2. **No direct port exposure:** Only Nginx exposes ports 80 and 443 to the internet
3. **Nginx reverse proxy:** Handles all incoming traffic, SSL termination, and proxies requests to the Flask app
4. **Certbot integration:** Automatically obtains and renews SSL certificates

#### Service: MySQL

```yaml
mysql:
  image: mysql:8.0
  networks:
    - backend
  restart: unless-stopped
```

- Runs on the isolated `backend` network (not directly accessible from the internet)
- **restart: unless-stopped** ensures the container automatically restarts if it crashes or the server reboots

The database is no longer exposed on port 3306 to the host, improving security. Only containers on the `backend` network can access it.

#### Service: Redis

```yaml
redis:
  command: redis-server --appendonly yes
  networks:
    - backend
```

- **--appendonly yes:** Enables Redis persistence mode (AOF - Append Only File), ensuring cached data survives restarts
- Also runs on the isolated `backend` network

#### Service: App

```yaml
app:
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
  networks:
    - backend
```

- No ports exposed to the host
- Only accessible via Nginx on the `backend` network
- Automatically restarts on failure

#### Service: Nginx

```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    - ./nginx/conf.d:/etc/nginx/conf.d:ro
    - certbot_www:/var/www/certbot:ro
    - certbot_conf:/etc/letsencrypt:ro
```

- Uses the lightweight Nginx Alpine image
- **Port 80:** HTTP traffic (redirects to HTTPS or serves Let's Encrypt challenges)
- **Port 443:** HTTPS traffic (SSL/TLS encrypted)
- Mounts configuration files as **read-only** (`:ro`) for security
- Shares volumes with Certbot for SSL certificate management

**Volume mounts explained:**
- `nginx.conf`: Main Nginx configuration
- `conf.d/`: Directory containing site-specific configurations
- `certbot_www`: Let's Encrypt challenge files (for domain verification)
- `certbot_conf`: SSL certificates and keys

#### Service: Certbot

```yaml
certbot:
  image: certbot/certbot:latest
  entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

Certbot is responsible for obtaining and automatically renewing SSL certificates from Let's Encrypt.

**How it works:**
- Runs in a continuous loop, checking for certificate renewal every 12 hours
- Let's Encrypt certificates are valid for 90 days
- Certbot automatically renews them when they have 30 days or less remaining
- Shares volumes with Nginx so renewed certificates are immediately available

**The entrypoint command explained:**
- `trap exit TERM`: Gracefully handle shutdown signals
- `while :; do ... done`: Infinite loop
- `certbot renew`: Check and renew certificates if needed
- `sleep 12h & wait $${!}`: Sleep for 12 hours (background process that can be interrupted)

#### Networks and Volumes

```yaml
networks:
  backend:
    driver: bridge
```

The `backend` network creates an isolated network for inter-container communication. Services on this network can reach each other by container name (e.g., `http://joke-app:5000`), but they're not directly accessible from the host or internet.

```yaml
volumes:
  mysql_data:
  redis_data:
  certbot_www:
  certbot_conf:
```

Named volumes persist data across container restarts and removals. Docker manages these volumes in `/var/lib/docker/volumes/`.

### Nginx Configuration

Nginx acts as a reverse proxy, sitting between the internet and your Flask application. It handles SSL termination, load balancing, and security headers.

#### Main Configuration: nginx.conf

**Location:** `nginx/nginx.conf`

```nginx
user nginx;
worker_processes auto;
```

- **user nginx:** Nginx runs as the `nginx` user (non-root) for security
- **worker_processes auto:** Automatically sets the number of worker processes based on CPU cores

```nginx
events {
    worker_connections 1024;
}
```

Each worker process can handle up to 1,024 simultaneous connections. With `auto` workers, this scales based on your server's CPU count.

```nginx
http {
    client_max_body_size 10M;
    
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
```

- **client_max_body_size 10M:** Limits request body size (uploads) to 10MB
- **gzip compression:** Compresses responses for faster transfer, reducing bandwidth usage by 60-80% for text-based content

```nginx
include /etc/nginx/conf.d/*.conf;
```

This line includes all `.conf` files from the `conf.d/` directory, allowing modular configuration.

#### Site Configuration: app.conf

**Location:** `nginx/conf.d/app.conf`

```nginx
server {
    listen 80;
    server_name _;
    
    resolver 127.0.0.11 valid=30s;
```

- **listen 80:** Accept HTTP connections on port 80
- **server_name _:** Accept requests for any domain (wildcard)
- **resolver 127.0.0.11:** Docker's internal DNS server, allowing Nginx to resolve container names like `joke-app`

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

This location block serves Let's Encrypt challenge files. When you request an SSL certificate, Let's Encrypt verifies domain ownership by accessing a file at `http://yourdomain.com/.well-known/acme-challenge/random-token`. Nginx serves these files from the shared `certbot_www` volume.

```nginx
location / {
    set $upstream http://joke-app:5000;
    proxy_pass $upstream;
    
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
```

This is the reverse proxy configuration that forwards all other requests to the Flask application:

- **set $upstream:** Uses a variable to prevent Nginx from crashing if the app container isn't available at startup
- **proxy_pass:** Forwards the request to the Flask app running on `joke-app:5000`
- **Headers:**
  - `Host`: Preserves the original hostname
  - `X-Real-IP`: Passes the client's real IP address (important for logging)
  - `X-Forwarded-For`: Chain of proxies the request passed through
  - `X-Forwarded-Proto`: Indicates whether the original request was HTTP or HTTPS

```nginx
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

Timeout settings prevent connections from hanging indefinitely:
- **connect_timeout:** How long to wait for a connection to the Flask app
- **send_timeout:** How long to wait when sending data to the Flask app
- **read_timeout:** How long to wait for a response from the Flask app

## Deployment

### Testing Deployment (without SSL)

First, test the application with the simplified configuration:

```bash
docker compose -f docker-compose.test.yml up --build -d
```

- `-f docker-compose.test.yml`: Use the test configuration file
- `--build`: Rebuild images if Dockerfile or code has changed
- `-d`: Run in detached mode (background)

**Verify the application:**
```bash
docker compose -f docker-compose.test.yml ps
```

All services should show status as "Up" or "Up (healthy)".

**Access the application:**
Open your browser and navigate to `http://localhost:5000` or `http://your-server-ip:5000`.

**View logs:**
```bash
docker compose -f docker-compose.test.yml logs -f app
```

The `-f` flag follows the logs in real-time (like `tail -f`).

**Stop the test environment:**
```bash
docker compose -f docker-compose.test.yml down
```

This stops and removes all containers but preserves data in volumes.

### Production Deployment (with Nginx and SSL)

Before deploying to production, ensure:
- Your domain's DNS A record points to your server's IP address
- Ports 80 and 443 are open in your firewall

#### Step 1: Update Nginx configuration with your domain

Edit `nginx/conf.d/app.conf` and replace `server_name _;` with your actual domain:

```nginx
server_name yourdomain.com www.yourdomain.com;
```

#### Step 2: Start the stack

```bash
docker compose up --build -d
```

This starts all services including Nginx and Certbot.

**Verify all services are running:**
```bash
docker compose ps
```

You should see:
- `joke-mysql` - Up (healthy)
- `joke-redis` - Up (healthy)
- `joke-app` - Up
- `joke-nginx` - Up
- `joke-certbot` - Up

#### Step 3: Check logs for errors

```bash
docker compose logs -f
```

Look for any error messages. Common issues:
- Database connection failures (check MySQL is healthy)
- Redis connection issues (check Redis is healthy)
- Nginx configuration syntax errors

## SSL/TLS Setup

### Initial Certificate Obtainment

To obtain your first SSL certificate from Let's Encrypt:

```bash
docker compose exec certbot certbot certonly --webroot \
  --webroot-path=/var/www/certbot \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email \
  -d yourdomain.com -d www.yourdomain.com
```

**Parameter explanation:**
- `certonly`: Only obtain the certificate, don't install it
- `--webroot`: Use the webroot method (places challenge files in `/var/www/certbot`)
- `--email`: Your email for renewal notifications
- `--agree-tos`: Agree to Let's Encrypt Terms of Service
- `--no-eff-email`: Don't share your email with EFF
- `-d`: Domains to include in the certificate (you can list multiple)

**Expected output:**
```
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/yourdomain.com/fullchain.pem
Your key file has been saved at:
/etc/letsencrypt/live/yourdomain.com/privkey.pem
```

### Enable HTTPS in Nginx

After obtaining certificates, update `nginx/conf.d/app.conf` to enable HTTPS:

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    resolver 127.0.0.11 valid=30s;
    
    location / {
        set $upstream http://joke-app:5000;
        proxy_pass $upstream;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**Reload Nginx:**
```bash
docker compose exec nginx nginx -s reload
```

Your application is now accessible via HTTPS at `https://yourdomain.com`!

### Automatic Renewal

The Certbot container automatically renews certificates every 12 hours. Let's Encrypt certificates expire after 90 days, but Certbot renews them when they have 30 days or less remaining.

**Verify renewal process:**
```bash
docker compose logs certbot
```

You should see entries like:
```
Cert not yet due for renewal
```

**Force a test renewal (dry run):**
```bash
docker compose exec certbot certbot renew --dry-run
```

This simulates renewal without actually requesting a new certificate.

## Maintenance

### Viewing Logs

**All services:**
```bash
docker compose logs -f
```

**Specific service:**
```bash
docker compose logs -f app
docker compose logs -f nginx
docker compose logs -f mysql
```

**Last 100 lines:**
```bash
docker compose logs --tail=100 app
```

### Restarting Services

**Restart a single service:**
```bash
docker compose restart app
```

**Restart all services:**
```bash
docker compose restart
```

### Updating the Application

When you update application code:

```bash
git pull
docker compose up --build -d
```

This rebuilds the app image with new code and restarts the container.

### Database Backups

**Backup MySQL database:**
```bash
docker compose exec mysql mysqldump -u joke_user -p joke_pass123 jokes_db > backup_$(date +%Y%m%d).sql
```

**Restore from backup:**
```bash
docker compose exec -T mysql mysql -u joke_user -p joke_pass123 jokes_db < backup_20250101.sql
```

### Stopping the Application

**Stop all services (keeps containers):**
```bash
docker compose stop
```

**Stop and remove containers (preserves volumes):**
```bash
docker compose down
```

**Stop and remove everything including volumes:**
```bash
docker compose down -v
```

⚠️ **Warning:** The `-v` flag permanently deletes all data!

## Troubleshooting

### Application won't start

**Check service status:**
```bash
docker compose ps
```

**Check service health:**
```bash
docker inspect joke-mysql | grep -A 10 Health
```

**Check if ports are in use:**
```bash
sudo netstat -tlnp | grep -E '80|443|3306|6379|5000'
```

### Database connection errors

**Ensure MySQL is healthy:**
```bash
docker compose exec mysql mysqladmin ping -h localhost
```

**Check environment variables:**
```bash
docker compose exec app env | grep DB_
```

**Test database connection manually:**
```bash
docker compose exec mysql mysql -u joke_user -p joke_pass123 jokes_db -e "SELECT 1;"
```

### Redis connection errors

**Test Redis connectivity:**
```bash
docker compose exec redis redis-cli ping
```

Expected output: `PONG`

### Nginx configuration errors

**Test configuration syntax:**
```bash
docker compose exec nginx nginx -t
```

**View Nginx error log:**
```bash
docker compose logs nginx | grep error
```

### SSL certificate issues

**Check certificate expiration:**
```bash
docker compose exec certbot certbot certificates
```

**Force certificate renewal:**
```bash
docker compose exec certbot certbot renew --force-renewal
```

### Permission denied errors

If you see Docker permission errors:
```bash
newgrp docker
```

Or restart your terminal session after adding yourself to the Docker group.

### Containers keep restarting

**Check resource usage:**
```bash
docker stats
```

**Check for OOM (Out of Memory) kills:**
```bash
dmesg | grep -i kill
```

**Increase container memory limits in docker-compose.yml:**
```yaml
app:
  deploy:
    resources:
      limits:
        memory: 512M
```

### Need more help?

1. Check container logs: `docker compose logs -f [service_name]`
2. Inspect container: `docker inspect joke-app`
3. Access container shell: `docker compose exec app /bin/bash`
4. Review Docker documentation: https://docs.docker.com/

---

## Quick Reference Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f

# Rebuild and restart
docker compose up --build -d

# Check service status
docker compose ps

# Execute command in container
docker compose exec app python init_db.py

# Access container shell
docker compose exec app /bin/bash

# Remove everything (including volumes)
docker compose down -v
```

# CI/CD Pipeline Documentation

## Overview

This document describes the Continuous Integration (CI) pipeline implemented for the Joke-a-Minute application using GitHub Actions. The pipeline automatically builds, tags, and publishes Docker images to GitHub Container Registry (GHCR) whenever code is pushed to the repository.

## Architecture

The CI pipeline consists of:
- **GitHub Actions**: Automated workflow engine
- **GitHub Container Registry (GHCR)**: Docker image storage
- **Multi-stage Dockerfile**: Optimized Python application container
- **Docker Compose**: Production deployment orchestration

## Implementation Steps

### 1. GitHub Actions Workflow Setup

**File Location**: `.github/workflows/docker-build-push.yml`

The workflow file was created directly in GitHub:
1. Navigate to repository → **Actions** tab
2. Select **"Set up a workflow yourself"**
3. Created `docker-build-push.yml` with the workflow configuration

### 2. Workflow Configuration

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main
      - develop
    tags:
      - 'v*'
  pull_request:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
```

**Trigger Conditions:**
- Pushes to `main` or `develop` branches
- Git tags starting with `v` (e.g., `v1.0.0`, `v2.1.3`)
- Pull requests targeting `main` branch (build only, no push)

### 3. Workflow Jobs

#### Job: `build-and-push`

**Step 1: Checkout Repository**
```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```
Clones the repository code into the GitHub Actions runner.

**Step 2: Set up Docker Buildx**
```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
```
Configures Docker Buildx for advanced build features including multi-platform builds and build caching.

**Step 3: Authenticate to GHCR**
```yaml
- name: Log in to GitHub Container Registry
  if: github.event_name != 'pull_request'
  uses: docker/login-action@v3
  with:
    registry: ${{ env.REGISTRY }}
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```
Authenticates to GitHub Container Registry using the automatically provided `GITHUB_TOKEN`. Authentication is skipped for pull requests to prevent unauthorized image pushes.

**Step 4: Generate Image Metadata**
```yaml
- name: Extract metadata (tags, labels)
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
    tags: |
      type=ref,event=branch
      type=ref,event=pr
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}}
      type=semver,pattern={{major}}
      type=sha,prefix={{branch}}-
      type=raw,value=latest,enable={{is_default_branch}}
```

**Generated Tags:**
- `latest` - Latest build from the default branch (main)
- `main`, `develop` - Branch-specific tags
- `v1.0.0`, `v1.0`, `v1` - Semantic versioning tags
- `main-abc1234` - Branch name + short commit SHA
- `pr-123` - Pull request number (for PR builds)

**Step 5: Build and Push Image**
```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: ./app
    file: ./app/Dockerfile
    push: ${{ github.event_name != 'pull_request' }}
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Key Features:**
- **Context**: `./app` directory contains the application code
- **Build Cache**: Uses GitHub Actions cache to speed up builds
- **Conditional Push**: Only pushes images for non-PR events
- **Multi-tagging**: Applies all generated tags to the built image

## Host Configuration Changes

### 1. Updated `docker-compose.yml`

**Before (Building Locally):**
```yaml
app:
  build:
    context: ./app
    dockerfile: Dockerfile
  container_name: joke-app
  # ... rest of configuration
```

**After (Using Pre-built Image):**
```yaml
app:
  image: ghcr.io/filusion/joke-a-minute_docker-compose:latest
  container_name: joke-app
  # ... rest of configuration
```

**Changes Made:**
- Removed `build` section
- Added `image` directive pointing to GHCR
- Application now pulls pre-built images instead of building locally

### 2. Initial Deployment

The following commands were executed on the production VM to deploy the containerized application:

```bash
# Stop and remove all existing containers and volumes
docker-compose down -v

# Pull the latest image from GitHub Container Registry
docker pull ghcr.io/filusion/joke-a-minute_docker-compose:latest

# Start all services using Docker Compose
docker-compose up -d

# Verify container status
docker-compose ps
```

**Note**: The `-v` flag removes named volumes. During initial setup, this was necessary to clean up conflicting container states. For routine updates, omit this flag to preserve data.

## Deployment Workflow

### Manual Deployment Process

When a new version needs to be deployed to production:

```bash
# 1. Navigate to project directory
cd ~/joke-app

# 2. Pull the latest image
docker pull ghcr.io/filusion/joke-a-minute_docker-compose:latest

# 3. Stop the application container
docker-compose stop app

# 4. Remove the old container
docker-compose rm -f app

# 5. Start the new container
docker-compose up -d app

# 6. Verify deployment
docker-compose ps app
docker-compose logs --tail=50 app
```