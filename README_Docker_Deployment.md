# Frappe Docker Deployment

This repository contains a Docker Compose based deployment setup for a Frappe application.  
The application image is built on a remote VM through GitLab CI/CD and deployed on the same VM using Docker Compose.

## Overview

The deployment flow is:

```text
GitLab CI/CD
    ↓
Connects to deployment VM using SSH
    ↓
Copies CI deployment files to VM
    ↓
Builds Docker image on VM
    ↓
Runs deploy.sh on VM
    ↓
deploy.sh updates docker-compose.yml with the commit SHA
    ↓
Docker Compose starts Frappe services
```

## Repository Structure

```text
.
├── ci/
│   ├── build.env
│   ├── deploy.sh
│   ├── docker-compose.yml
│   ├── nginx-template.conf
│   └── nginx-entrypoint.sh
├── Dockerfile
├── .gitlab-ci.yml
└── README.md
```

## Components

### 1. `.gitlab-ci.yml`

The GitLab pipeline has two main jobs:

```text
build_on_vm
deploy_to_vm
```

### 2. `Dockerfile`

The Dockerfile builds a Frappe image using a multi-stage build.

It includes:

- Python `3.12.3-slim-bookworm`
- Frappe Bench
- Node.js installed using NVM
- Yarn
- wkhtmltopdf
- Nginx
- PostgreSQL and MariaDB clients
- Redis tools
- Required system packages for Frappe, WeasyPrint, backups, and Python packages
- Frappe bench initialized using an `apps.json` file
- Runtime image for backend services

### 3. `ci/docker-compose.yml`

The Docker Compose file runs the full Frappe stack.

Main services:

```text
volume-init
configurator
create-site
backend
frontend
queue-long
queue-short
scheduler
websocket
redis-cache
redis-queue
```

### 4. `ci/deploy.sh`

The deployment script runs on the VM.

It performs:

1. Loads environment variables from `build.env`
2. Moves to the application deployment directory
3. Replaces `$CI_COMMIT_SHA` placeholder in `docker-compose.yml`
4. Stops the existing Docker Compose stack
5. Removes unwanted Docker volumes
6. Recreates required persistent volumes
7. Starts the Docker Compose stack using the newly built image

## Deployment Architecture

```text
User / Browser
    ↓
frontend container - Nginx - port 8080
    ↓
backend container - Gunicorn - port 8000
    ↓
Frappe application
    ↓
External PostgreSQL database

Additional services:
- redis-cache
- redis-queue
- websocket
- scheduler
- queue workers
```

## GitLab CI/CD Pipeline

### Build Job: `build_on_vm`

This job connects to the target VM and builds the Docker image on the VM itself.

The job uses:

```text
docker.io/library/ubuntu:24.04
```

Build process:

1. Loads variables from `ci/build.env`
2. Installs SSH client
3. Configures SSH known hosts
4. Decodes the private SSH key
5. Creates the deployment directory on the VM
6. Copies all files from `ci/` to the VM
7. Runs `docker build` on the VM

Image tag format:

```text
<DEPLOYMENT_NAME>:<CI_COMMIT_SHA>
```

Example:

```bash
docker build \
  --build-arg FRAPPE_PATH=${FRAPPE_PATH:-${FRAPPE_REPO}} \
  --build-arg APPS=${APPS} \
  --build-arg FRAPPE_BRANCH=${FRAPPE_BRANCH:-${FRAPPE_VERSION}} \
  --build-arg PYTHON_VERSION=${PYTHON_VERSION:-${PY_VERSION}} \
  --build-arg NODE_VERSION=${NODE_VERSION:-${NODEJS_VERSION}} \
  --build-arg VM_USER=${VM_USER} \
  -t $DEPLOYMENT_NAME:$CI_COMMIT_SHA \
  .
```

### Deploy Job: `deploy_to_vm`

This job connects to the VM and runs `deploy.sh`.

Deployment process:

1. Installs SSH client
2. Configures SSH private key
3. Verifies SSH connection to the VM
4. Exports required deployment variables
5. Runs:

```bash
/usr/bin/bash /home/$VM_USER/$DEPLOYMENT_NAME/deploy.sh
```

The deployment job uses the image built in the previous job:

```text
$DEPLOYMENT_NAME:$CI_COMMIT_SHA
```

## Dockerfile Details

The Dockerfile has three stages:

```text
base
builder
backend
```

### Stage 1: `base`

The `base` stage installs common runtime dependencies.

Included tools and packages:

- Python
- Frappe Bench
- Node.js using NVM
- Yarn
- Nginx
- wkhtmltopdf
- PostgreSQL client
- MariaDB client
- Redis tools
- Restic
- GPG
- jq
- wait-for-it
- WeasyPrint dependencies

It also copies:

```text
ci/nginx-template.conf
ci/nginx-entrypoint.sh
```

### Stage 2: `builder`

The `builder` stage installs build dependencies and initializes the Frappe bench.

It creates:

```text
/opt/frappe/apps.json
```

from the base64 encoded `APPS` build argument.

Then it runs:

```bash
bench init \
  --apps_path=/opt/frappe/apps.json \
  --frappe-branch=${FRAPPE_BRANCH} \
  --frappe-path=${FRAPPE_PATH} \
  --no-procfile \
  --no-backups \
  --skip-redis-config-generation \
  /home/frappe/frappe-bench
```

Additional Python packages installed:

```text
captcha==0.7.1
python-dotenv
```

### Stage 3: `backend`

The `backend` stage copies the generated bench from the builder image.

It exposes the runtime Frappe bench with Docker volumes:

```text
/home/frappe/frappe-bench/sites
/home/frappe/frappe-bench/apps
/home/frappe/frappe-bench/env
/home/frappe/frappe-bench/config
/home/frappe/frappe-bench/sites/assets
/home/frappe/frappe-bench/logs
```

Default command:

```bash
env PYTHONPATH=/home/frappe/frappe-bench/apps \
/home/frappe/frappe-bench/env/bin/gunicorn \
  --chdir=/home/frappe/frappe-bench/sites \
  --bind=0.0.0.0:8000 \
  --threads=4 \
  --workers=5 \
  --worker-class=gthread \
  --worker-tmp-dir=/dev/shm \
  --timeout=120 \
  --preload \
  frappe.app:application
```

## Docker Compose Services

### `volume-init`

Initializes Docker volumes required by the Frappe stack.

Mounted volumes:

```text
sites
logs
assets
apps
env
config
```

### `configurator`

Configures Frappe common site configuration.

It sets:

```text
db_host
db_port
redis_cache
redis_queue
redis_socketio
socketio_port
developer_mode
server_script_enabled
```

### `create-site`

Creates the Frappe site and installs required apps.

It runs:

```bash
bench new-site $SITE_NAME \
  --db-host="$DB_HOST" \
  --db-type="postgres" \
  --admin-password="$ADMIN_PASSWORD" \
  --db-root-password="$DB_ROOT_PASSWORD" \
  --db-user="$DB_USER" \
  --db-name="$DB_NAME" \
  --db-password="$DB_PASSWORD"
```

It also sets application config values:

```text
gis_user
gis_password
gis_end_url
```

Apps installed by this service:

```text
survey_v2
commit
dfp_external_storage
frappe_s3_attachment
india_compliance
erpnext
```

### `backend`

Runs the Frappe backend using Gunicorn.

Before starting Gunicorn, it:

1. Selects the site using `bench use`
2. Pulls latest code for apps
3. Runs migration
4. Runs production build
5. Starts Gunicorn

Backend port:

```text
8000
```

### `frontend`

Runs Nginx as the frontend reverse proxy.

It forwards requests to:

```text
backend:8000
```

It also proxies websocket traffic to:

```text
websocket:9000
```

Frontend port:

```text
8080
```

### `queue-long`

Runs Frappe worker for:

```text
long,default,short
```

### `queue-short`

Runs Frappe worker for:

```text
short,default
```

### `scheduler`

Runs Frappe scheduler:

```bash
bench schedule
```

### `websocket`

Runs Frappe Socket.IO:

```bash
node /home/frappe/frappe-bench/apps/frappe/socketio.js
```

Websocket port:

```text
9000
```

### `redis-cache`

Runs Redis cache service.

Host port:

```text
13000
```

Container port:

```text
6379
```

### `redis-queue`

Runs Redis queue service.

Host port:

```text
11000
```

Container port:

```text
6379
```

## Required GitLab CI/CD Variables

Configure these variables in GitLab:

### SSH Variables

```text
SSH_PRIVATE_KEY
SSH_KNOWN_HOSTS
VM_USER
SERVER_NAME
```

`SSH_PRIVATE_KEY` should be base64 encoded.

Example:

```bash
base64 -w 0 ~/.ssh/deploy_key
```

### Deployment Variables

```text
DEPLOYMENT_NAME
DEPLOY_ENVIRONMENT
SITE_NAME
```

### Frappe Build Variables

```text
FRAPPE_REPO
FRAPPE_PATH
FRAPPE_VERSION
FRAPPE_BRANCH
PY_VERSION
PYTHON_VERSION
NODEJS_VERSION
NODE_VERSION
APPS
```

`APPS` should be a base64 encoded `apps.json`.

Example:

```bash
base64 -w 0 apps.json
```

### Database Variables

```text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
DB_ROOT_USERNAME
DB_ROOT_PASSWORD
ADMIN_PASSWORD
```

### Application Variables

```text
GIS_USER
GIS_PASSWORD
GIS_BASE_URL
```

## Example `apps.json`

Create an `apps.json` file that contains all Frappe apps to be installed during `bench init`.

Example:

```json
[
  {
    "url": "https://gitlab.example.com/group/frappe.git",
    "branch": "main"
  },
  {
    "url": "https://gitlab.example.com/group/erpnext.git",
    "branch": "main"
  },
  {
    "url": "https://gitlab.example.com/group/survey-v2.git",
    "branch": "main"
  },
  {
    "url": "https://gitlab.example.com/group/commit.git",
    "branch": "main"
  },
  {
    "url": "https://gitlab.example.com/group/frappe_dfp_minio.git",
    "branch": "main"
  }
]
```

Encode it:

```bash
base64 -w 0 apps.json
```

Store the encoded output in GitLab variable:

```text
APPS
```

## Example `ci/build.env`

Create `ci/build.env` with non-secret defaults.

```bash
DEPLOYMENT_NAME=frappe-docker-deployment
FRAPPE_VERSION=main
PY_VERSION=3.12.3
NODEJS_VERSION=20.19.0
SITE_NAME=example.com
```

Sensitive values should be stored in GitLab CI/CD variables, not committed to Git.

## VM Prerequisites

The target VM should have:

```text
Docker
Docker Compose plugin
Git
Bash
SSH access from GitLab runner
Network access to database
Network access to Git repositories
```

Check Docker:

```bash
docker --version
docker compose version
```

Check VM connectivity:

```bash
ssh <VM_USER>@<SERVER_NAME>
```

## Manual Build on VM

Go to deployment directory:

```bash
cd /home/<VM_USER>/<DEPLOYMENT_NAME>
```

Build image:

```bash
docker build \
  --build-arg FRAPPE_PATH=<FRAPPE_REPO_URL> \
  --build-arg APPS=<BASE64_APPS_JSON> \
  --build-arg FRAPPE_BRANCH=main \
  --build-arg NODE_VERSION=20.19.0 \
  --build-arg VM_USER=<VM_USER> \
  -t <DEPLOYMENT_NAME>:manual \
  .
```

## Manual Deployment on VM

Export required variables:

```bash
export DEPLOYMENT_NAME=frappe-docker-deployment
export CI_COMMIT_SHA=manual
export VM_USER=ubuntu
export SITE_NAME=example.com
export DB_HOST=<DB_HOST>
export DB_PORT=5432
export DB_NAME=<DB_NAME>
export DB_USER=<DB_USER>
export DB_PASSWORD=<DB_PASSWORD>
export DB_ROOT_USERNAME=<DB_ROOT_USERNAME>
export DB_ROOT_PASSWORD=<DB_ROOT_PASSWORD>
export ADMIN_PASSWORD=<ADMIN_PASSWORD>
export GIS_USER=<GIS_USER>
export GIS_PASSWORD=<GIS_PASSWORD>
export GIS_BASE_URL=<GIS_BASE_URL>
```

Run deployment:

```bash
bash /home/$VM_USER/$DEPLOYMENT_NAME/deploy.sh
```

## Useful Docker Commands

Check running containers:

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

View logs:

```bash
docker compose logs -f
```

View backend logs:

```bash
docker compose logs -f backend
```

View frontend logs:

```bash
docker compose logs -f frontend
```

Enter backend container:

```bash
docker compose exec backend bash
```

Run bench command:

```bash
docker compose exec backend bench --site $SITE_NAME migrate
```

Restart stack:

```bash
docker compose restart
```

Stop stack:

```bash
docker compose down
```

Start stack:

```bash
docker compose up -d --no-build
```

## Backup and Restore

### Backup

```bash
docker compose exec backend bench --site $SITE_NAME backup --with-files
```

Backup files will be created inside:

```text
/home/frappe/frappe-bench/sites/<SITE_NAME>/private/backups
```

### Restore

Copy backup files into the backend container or mounted site volume, then run:

```bash
docker compose exec backend bench --site $SITE_NAME restore <database-backup.sql.gz> \
  --with-public-files <public-files.tar> \
  --with-private-files <private-files.tar>
```

## Troubleshooting

### 1. SSH connection failed

Check:

```bash
ssh -i ~/.ssh/deploy_key <VM_USER>@<SERVER_NAME>
```

Verify:

- `SSH_PRIVATE_KEY` is valid
- `SSH_PRIVATE_KEY` is base64 encoded correctly
- `SSH_KNOWN_HOSTS` contains the VM host key
- VM security group/firewall allows SSH

### 2. Docker build failed

Check build logs in GitLab.

On the VM, check Docker status:

```bash
sudo systemctl status docker
```

Run manual build to reproduce:

```bash
docker build --progress=plain .
```

### 3. `APPS` decode failed

The Dockerfile expects `APPS` to be base64 encoded.

Validate locally:

```bash
echo "$APPS" | base64 -d
```

It should output valid JSON.

### 4. Site creation failed

Check create-site logs:

```bash
docker compose logs create-site
```

Common causes:

- Database unreachable
- Incorrect DB credentials
- Database user does not have required permissions
- Site already exists
- PostgreSQL extensions missing

### 5. Backend migration failed

Check backend logs:

```bash
docker compose logs backend
```

Enter backend container:

```bash
docker compose exec backend bash
cd /home/frappe/frappe-bench
bench --site $SITE_NAME migrate
```

### 6. Frontend not loading

Check frontend logs:

```bash
docker compose logs frontend
```

Check if backend is reachable from frontend:

```bash
docker compose exec frontend curl http://backend:8000
```

### 7. Redis issue

Check Redis health:

```bash
docker compose ps
docker compose logs redis-cache
docker compose logs redis-queue
```

Test Redis:

```bash
docker compose exec redis-cache redis-cli ping
docker compose exec redis-queue redis-cli ping
```

### 8. Websocket issue

Check websocket logs:

```bash
docker compose logs websocket
```

Verify port:

```bash
docker compose exec frontend curl http://websocket:9000
```

## Important Notes

### Persistent Volumes

The Docker Compose setup uses named volumes:

```text
sites
config
assets
logs
apps
env
```

These volumes persist Frappe site data, logs, app code, environment, and assets.

### Volume Cleanup

The deployment script removes Docker volumes except selected persistent volumes.

Review this command carefully before using in production:

```bash
sudo ls /var/lib/docker/volumes/ | grep -v '^cicd_sites' | xargs -r docker volume rm || true
```

This can remove unrelated Docker volumes if the VM is shared with other applications.

### Runtime Git Pull

The backend command performs `git pull` inside app directories at container startup.

This means the running code may change after image build.

Recommended production improvement:

- Build all app code into the Docker image
- Avoid `git pull` during container startup
- Use immutable image tags
- Deploy only tested images

### Duplicate Volume Name

Check the `volumes` section in `docker-compose.yml`.

If `assets` appears twice, keep only one entry:

```yaml
volumes:
  sites:
  config:
  assets:
  logs:
  ssh:
  apps:
  env:
```

## Security Notes

Do not commit or hardcode:

- GitLab tokens
- GitHub tokens
- Database passwords
- Admin passwords
- SSH private keys
- Superset passwords
- GIS credentials
- Any `.env` file containing secrets

Use GitLab CI/CD masked variables for secrets.

Recommended:

- Rotate any token/password that was pasted in logs or committed
- Use deploy tokens or SSH deploy keys for private repositories
- Store credentials in GitLab CI/CD variables
- Mark sensitive variables as masked and protected
- Avoid passing secrets as Docker build arguments
- Avoid storing real credentials inside `apps.json`
- Restrict SSH key access to only the deployment VM

## Production Recommendations

- Use immutable Docker images
- Avoid runtime `git pull`
- Use separate database user with minimum required permissions
- Keep database outside Docker Compose for production
- Add health checks for backend and frontend
- Configure log rotation
- Backup database and files regularly
- Use HTTPS reverse proxy in front of frontend
- Do not remove all Docker volumes on a shared VM
- Keep separate environments for development, staging, and production
- Pin dependency versions wherever possible
