# mini-gitops-compose

A lightweight, poll-based GitOps automation workflow built with **Bash**, **Git**, **Cron**, and **Docker Compose** for single-node Linux environments.

## Overview
Instead of relying on heavy external CI/CD runners for simple deployments, `mini-gitops-compose` monitors a target Git branch at scheduled intervals, compares local and remote commit hashes, and automatically redeploys containerized services when updates are detected. It also includes a built-in self-healing check to recover stopped containers automatically.

## Key Features
- **Automated Git Polling:** Periodically fetches `origin/main` and compares commit hashes (`git rev-parse`) to trigger deployments only when actual code changes occur.
- **Zero-Touch Redeployment:** Automatically pulls new commits and updates the stack via `docker compose up -d --build --remove-orphans`.
- **Automated Service Recovery (Self-Healing):** Inspects container runtime status (`docker inspect`) on every run and restarts the service if it has crashed or stopped.
- **Resource & Volume Management:** Enforces memory limits (`128M`) and mounts static application files as read-only (`:ro`) volumes.
- **Audit & Deployment Logging:** Records timestamps, commit hash transitions, and recovery actions in `deploy.log`.

## Project Structure
```text
mini-gitops-compose/
├── app/
│   └── index.html         # Sample static web application
├── docker-compose.yml     # Container definition, port mapping, and resource limits
├── sync.sh                # Core Bash automation & health-check script
├── .gitignore             # Excludes runtime logs (deploy.log) from version control
└── README.md              # Project documentation
```

## Prerequisites
- **OS:** Linux environment (Debian / Ubuntu recommended)
- **Packages:** `git`, `docker`, `docker-compose-plugin` (Docker Compose v2), and `cron`

## Quick Start

1. **Clone the repository and navigate to the project directory:**
   ```bash
   git clone [https://github.com/hassanzargar/mini-gitops-compose.git](https://github.com/hassanzargar/mini-gitops-compose.git)
   cd mini-gitops-compose
   ```

2. **Grant execution permissions to the automation script:**
   ```bash
   chmod +x sync.sh
   ```

3. **Start the initial Docker Compose stack:**
   ```bash
   docker compose up -d
   ```

4. **Verify that the service is running:**
   ```bash
   docker ps
   curl -i http://localhost:8085
   ```

## Cron Setup (Automated Polling)

To run `sync.sh` automatically every 2 minutes in the background, configure a user crontab job:

1. **Open the crontab editor:**
   ```bash
   crontab -e
   ```

2. **Add the following lines at the bottom of the file** (replace `/opt/mini-gitops-compose` with your actual project path):
   ```cron
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   */2 * * * * /opt/mini-gitops-compose/sync.sh >/dev/null 2>&1
   ```
   > **Note:** Explicitly defining `PATH` ensures that Cron can locate the `docker` and `git` binaries properly in non-interactive shells.

3. **Verify that the cron job is registered:**
   ```bash
   crontab -l
   ```

## Testing the Workflow

### 1. Test Automated Git Deployment
Make a change to `app/index.html` (e.g., update `Version 1.0` to `Version 2.0`), commit, and push it to GitHub. Within 2 minutes (or by running `./sync.sh` manually), check the deployment log:
```bash
cat deploy.log
```
Expected log output:
```text
[2026-11-02 12:30:01] Update detected (a1b2c3d -> e4f5g6h). Pulling changes and redeploying...
[2026-11-02 12:30:04] Deployment successful. Current commit: e4f5g6h
```

### 2. Test Automated Container Recovery
Simulate a service outage by manually stopping the container:
```bash
docker stop mini-gitops-web
```
Run `./sync.sh` (or wait for the 2-minute Cron cycle) and inspect `deploy.log`:
```bash
cat deploy.log
```
Expected log output:
```text
[2026-11-02 12:32:01] WARN: Container mini-gitops-web is down! Triggering automated recovery...
[2026-11-02 12:32:03] Recovery complete. Container mini-gitops-web is back online.
```
