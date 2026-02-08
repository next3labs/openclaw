---
title: Docker Setup (Fork Changes)
description: Documentation for Docker setup changes in this fork
---

# Docker Setup Changes

This fork includes modifications to the Docker setup for improved permission handling and non-root container compatibility.

## Changes Overview

| Change | Purpose |
|--------|---------|
| Host UID/GID matching | Container runs with host user's permissions for bind mounts |
| npm global prefix | Skill installations work in non-root container |

---

## Host UID/GID Matching

### What It Does

The container now runs with the same UID/GID as your host user, allowing:

- Read/write access to bind-mounted directories (`~/.openclaw`)
- Host files can be edited directly with normal permissions
- No permission denied errors when container writes to host

### How It Works

```bash
# docker-setup.sh detects your user
Container user: uid=1000, gid=1000

# Passes these to docker-compose.yml
user: "${PUID:-1000}:${PGID:-1000}"
```

### Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `PUID` | User ID for container | Auto-detected from host |
| `PGID` | Group ID for container | Auto-detected from host |

### Usage

```bash
# Default (auto-detect from current user)
./docker-setup.sh

# Explicit override
PUID=1000 PGID=1000 ./docker-setup.sh

# Via sudo (automatically detects sudoing user)
sudo ./docker-setup.sh
```

### Migrating Existing Config

If you have existing root-owned config directories:

```bash
# Transfer ownership to your user before running setup
sudo chown -R $(id -u):$(id -g) ~/.openclaw
```

---

## npm Global Prefix for Non-Root Container

### Problem

The original Docker image runs as a non-root user (`node`, uid 1000). When OpenClaw tries to install skill dependencies via npm globally, it fails:

```
npm error EACCES: permission denied, mkdir '/usr/local/lib/node_modules/clawhub'
```

### Solution

The Dockerfile now configures npm to use a user-writable directory:

```dockerfile
# Create user-writable npm global directory
RUN mkdir -p /home/node/.npm-global && \
    npm config set prefix '/home/node/.npm-global' && \
    chown -R node:node /home/node/.npm-global

# Add to PATH
ENV PATH="/home/node/.npm-global/bin:${PATH}"
```

### What This Enables

- Skill dependencies install successfully in non-root container
- npm global packages go to `~/.npm-global/lib/node_modules/`
- Skill CLIs are available in PATH
- No sudo or root access required

---

## Setup Instructions

### Prerequisites

```bash
# Ensure user is in docker group
sudo usermod -aG docker $USER
# Log out and back in for group changes to apply
```

### Running Setup

```bash
# Clone and enter directory
git clone <your-fork-url>
cd openclaw

# Run setup (UID/GID auto-detected)
./docker-setup.sh
```

### Onboarding Prompts

When prompted during onboarding:

| Prompt | Recommended Answer |
|--------|-------------------|
| Gateway bind | `lan` |
| Gateway auth | `token` |
| Gateway token | (auto-generated) |
| Tailscale exposure | `Off` |
| Install Gateway daemon | `No` |

### Verifying Setup

```bash
# Check container runs as your user
docker compose exec openclaw-gateway id
# Output: uid=1000(hongchuan) gid=1000(hongchuan) ...

# Verify bind mounts work
ls -la ~/.openclaw/

# Check gateway is running
docker compose ps
```

---

## Troubleshooting

### Permission Denied on Bind Mounts

```bash
# Fix existing directories
sudo chown -R $(id -u):$(id -g) ~/.openclaw
```

### Skill Installation Fails

```bash
# Rebuild image with npm fix
docker compose down
docker build --no-cache -t openclaw:local -f Dockerfile .
docker compose up -d openclaw-gateway
```

### Container Won't Start

```bash
# Check logs
docker compose logs openclaw-gateway

# Verify .env has correct PUID/PGID
cat .env | grep -E "PUID|PGID"
```

---

## Files Modified

| File | Change |
|------|--------|
| `docker-compose.yml` | Added `user:` directive with PUID/PGID |
| `docker-setup.sh` | Added UID/GID detection and export |
| `Dockerfile` | Added npm global prefix configuration |
| `scripts/test-live-gateway-models-docker.sh` | Added `--user` flag |
| `scripts/test-live-models-docker.sh` | Added `--user` flag |

---

## Comparison with Upstream

| Feature | Upstream | This Fork |
|---------|----------|-----------|
| Container user | `node` (uid 1000) | Host UID/GID |
| Bind mount permissions | May fail | Works correctly |
| npm global installs | Fails in container | Works via ~/.npm-global |
| Skill installation | Requires root | Works non-root |
