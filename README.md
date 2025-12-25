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
├── app
│   ├── app.py
│   ├── Dockerfile
│   ├── init_db.py
│   └── requirements.txt
├── certbot
│   └── init-certs.sh
├── docker-compose.test.yml
├── docker-compose.yml
├── nginx
│   ├── conf.d
│   │   └── app.conf
│   └── nginx.conf
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

#### Service: Certbot-init
```yaml
certbot-init:
  image: certbot/certbot:latest
  container_name: joke-certbot-init
  volumes:
    - ./certbot/init-certs.sh:/init-certs.sh:ro
    - certbot_www:/var/www/certbot
    - certbot_conf:/etc/letsencrypt
  entrypoint: /init-certs.sh
  depends_on:
    - nginx
  networks:
    - backend
```

**Purpose:** This is a one-time initialization service that obtains the initial SSL certificate from Let's Encrypt before the main certbot renewal service starts.

**How it differs from the main certbot service:**
- Runs once at startup (not continuously)
- Executes the `init-certs.sh` script to request new certificates
- Exits after certificate generation (won't restart automatically)
- Must run after Nginx is up (ensured by `depends_on: nginx`)

**Volume mounts explained:**
- `./certbot/init-certs.sh:/init-certs.sh:ro`: Mounts the initialization script as read-only
- `certbot_www`: Shared with Nginx for Let's Encrypt domain verification challenges
- `certbot_conf`: Stores the generated SSL certificates

**Workflow:**
1. Docker Compose starts Nginx first (due to `depends_on`)
2. `certbot-init` container starts and runs the `init-certs.sh` script
3. Script requests SSL certificate from Let's Encrypt
4. Certificate is saved to the `certbot_conf` volume
5. Container exits after completion
6. Nginx can now use the generated certificates


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

This configuration file defines an **NGINX reverse proxy setup** for a web application running inside a Docker environment. It handles both **HTTP (port 80)** and **HTTPS (port 443)** traffic, integrates **Let’s Encrypt** for SSL certificates, and forwards requests to a backend Flask application.

---

### HTTP Server (Port 80)

The first `server` block listens on **port 80** and serves two main purposes:

- **Domain handling**  
  The server responds to requests for `devops-vm-43.lrk.si`.

- **Let’s Encrypt ACME challenge**  
  The path `/.well-known/acme-challenge/` is used by Certbot to verify domain ownership.  
  Files for this challenge are served from `/var/www/certbot`.

- **Reverse proxy to the application**  
  All other HTTP requests are forwarded to the backend service:
  - The application runs at `http://joke-app:5000`
  - `joke-app` is resolved via Docker’s internal DNS (`127.0.0.11`)

- **Forwarded headers**  
  Headers such as `Host`, `X-Real-IP`, and `X-Forwarded-*` ensure that the backend application receives correct client and protocol information.

- **Timeout configuration**  
  Connection, send, and read timeouts are set to 60 seconds to avoid premature connection drops.

---

### HTTPS Server (Port 443)

The second `server` block enables **secure HTTPS communication** using SSL/TLS:

- **SSL & HTTP/2 support**  
  - Listens on port 443 with SSL and HTTP/2 enabled
  - Uses TLS versions 1.2 and 1.3 for secure communication

- **SSL certificates**  
  Certificates are provided by Let’s Encrypt and loaded from:
  - `/etc/letsencrypt/live/devops-vm-43.lrk.si/fullchain.pem`
  - `/etc/letsencrypt/live/devops-vm-43.lrk.si/privkey.pem`

- **Security configuration**
  - Strong cipher suites are enforced
  - Weak or insecure algorithms are disabled
  - SSL sessions are cached for better performance

- **Reverse proxy to Flask app**  
  HTTPS requests are forwarded to the same backend service (`joke-app:5000`) as HTTP traffic.

- **Docker DNS resolver**  
  The internal Docker resolver is used again to dynamically resolve container IP addresses.

### Summary

This `app.conf` file:
- Acts as a **reverse proxy** between users and the Flask application
- Enables **automatic HTTPS** via Let’s Encrypt
- Supports **Docker-based service discovery**
- Ensures secure, reliable, and properly forwarded client connections

It is designed for a production-like environment where NGINX handles networking, security, and certificate management while the application runs independently in a container.


### init-certs.sh

**Purpose:** This shell script automates the initial SSL certificate generation from Let's Encrypt, avoiding the need for manual certificate requests.

```shellscript
#!/bin/sh

# Configuration
DOMAIN="devops-vm-43.lrk.si"
EMAIL="admin@devops-vm-43.lrk.si"  
STAGING="--staging"  # Remove this line for production certs

# Check if certificate already exists
if [ -d "/etc/letsencrypt/live/$DOMAIN" ]; then
    echo "Certificate for $DOMAIN already exists. Skipping generation."
    exit 0
fi

echo "Certificate not found. Generating new certificate for $DOMAIN..."

# Wait for Nginx to be ready
echo "Waiting for Nginx to be ready..."
sleep 15 

# Request certificate
certbot certonly --webroot \
    --webroot-path=/var/www/certbot \
    --email "$EMAIL" \
    --agree-tos \
    --no-eff-email \
    $STAGING \
    -d "$DOMAIN" \
    --non-interactive

if [ $? -eq 0 ]; then
    echo "Certificate successfully generated for $DOMAIN"
else
    echo "Failed to generate certificate for $DOMAIN"
    exit 1
fi
```

#### Configuration Variables

- **DOMAIN:** The domain name for which to generate the SSL certificate
- **EMAIL:** Contact email for Let's Encrypt notifications (certificate expiration warnings, security notices)
- **STAGING:** Uses Let's Encrypt's staging environment for testing
  - Staging certificates are not trusted by browsers (will show security warnings)
  - **Why use staging?** Let's Encrypt has rate limits (5 certificates per domain per week). Staging has higher limits and is perfect for testing
  - **For production:** Remove the `STAGING="--staging"` line or comment it out

#### Script Logic

1. **Certificate existence check:**
```bash
   if [ -d "/etc/letsencrypt/live/$DOMAIN" ]; then
```
   - Checks if certificates already exist to avoid unnecessary requests
   - Prevents hitting Let's Encrypt rate limits on repeated runs
   - Exits early if certificates are found

2. **Nginx readiness wait:**
```bash
   sleep 15
```
   - Gives Nginx time to fully start and begin serving HTTP traffic
   - Let's Encrypt needs to access `http://DOMAIN/.well-known/acme-challenge/` for domain verification
   - If Nginx isn't ready, certificate request will fail

3. **Certificate request (certbot certonly):**
```bash
   certbot certonly --webroot \
       --webroot-path=/var/www/certbot \
       --email "$EMAIL" \
       --agree-tos \
       --no-eff-email \
       $STAGING \
       -d "$DOMAIN" \
       --non-interactive
```

   **Flag explanations:**
   - `certonly`: Only obtain the certificate, don't install it (Nginx configuration handles installation)
   - `--webroot`: Use the webroot authentication method (places verification files in a directory that Nginx serves)
   - `--webroot-path=/var/www/certbot`: Directory where Let's Encrypt verification files are placed
   - `--email "$EMAIL"`: Contact email for urgent renewal and security notices
   - `--agree-tos`: Automatically agree to Let's Encrypt Terms of Service
   - `--no-eff-email`: Don't share email with the Electronic Frontier Foundation (EFF)
   - `$STAGING`: If set, uses staging environment (test certificates)
   - `-d "$DOMAIN"`: Domain name to generate certificate for
   - `--non-interactive`: Run without prompts (required for automation)

4. **Success/failure handling:**
```bash
   if [ $? -eq 0 ]; then
```
   - `$?` contains the exit code of the last command (0 = success, non-zero = error)
   - Exits with error code 1 if certificate generation fails, which Docker logs and can alert on

#### Let's Encrypt Validation Process

When this script runs, the following happens:

1. **Certbot sends a certificate request** to Let's Encrypt servers
2. **Let's Encrypt responds** with a challenge: "Prove you control this domain"
3. **Certbot creates a verification file** in `/var/www/certbot/.well-known/acme-challenge/`
4. **Let's Encrypt makes an HTTP request** to `http://devops-vm-43.lrk.si/.well-known/acme-challenge/[random-string]`
5. **Nginx serves the verification file** (configured in nginx config to serve files from `/var/www/certbot`)
6. **Let's Encrypt verifies** the file content matches what it expects
7. **Certificate is issued** and saved to `/etc/letsencrypt/live/$DOMAIN/`

**This is why Nginx must be running on port 80 and serving the `.well-known/acme-challenge/` location before certificates can be obtained.**

#### Switching from Staging to Production

Once you've tested and verified the setup works with staging certificates:

1. Edit `init-certs.sh` and remove or comment out:
```bash
   # STAGING="--staging"
```

2. Remove existing staging certificates:
```bash
   docker compose down
   sudo rm -rf certbot_conf/*
```

3. Restart and generate production certificates:
```bash
   docker compose up -d
```

Production certificates will be trusted by all browsers and valid for 90 days.

## Deployment 

This guide explains how to deploy the application using Docker, Nginx, and automatic SSL via Let’s Encrypt.

---

#### 1. Prepare Nginx

- Edit `nginx/conf.d/app.conf` and **comment out the HTTPS server block**, leaving only HTTP active.  
- This allows Certbot to generate the SSL certificate first.

---

#### 2. Start Docker services

```bash
docker compose up -d
docker compose ps nginx

```
* Verify that Nginx is running

#### 3. Generate SSL Certificate

``` bash 
  docker compose up certbot-init
  docker compose logs -f certbot-init
```
* Wait until the certificate is successfully generated.

#### 4. Enable HTTPS

* Uncomment the HTTPS server block in app.conf.

* Reload Nginx to apply changes:

``` bash
  docker compose exec nginx nginx -s reload
```
#### 5. Initialize the database

```bash
  docker exec -it joke-app python init_db.py
```

#### 6. Access the Application

* The app is now available at:

``` txt
  https://devops-sk-10.lrk.si/
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