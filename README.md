# 🐳 OpenClaw — Isolated Docker Deployment

A guide to deploying **OpenClaw** in a fully isolated Docker environment with enhanced security hardening, resource limits, and network isolation.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Install ClawHub Skills](#install-clawhub-skills)
- [Configuration](#configuration)
  - [Environment Variables](#environment-variables)
  - [Volumes](#volumes)
  - [Ports](#ports)
  - [Resource Limits](#resource-limits)
- [Security Hardening](#security-hardening)
- [Networking](#networking)
- [Managing the Deployment](#managing-the-deployment)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Overview

> **⚠️ Simplified Deployment:** This is a **simplified version** of the standard OpenClaw Docker Compose setup. Notably, it **does not include the `openclaw-cli` container** that the [official OpenClaw `docker-compose.yml`](https://github.com/openclaw/openclaw) ships with. Only the **gateway** service is deployed here.
>
> This setup is ideal for users who are **already running a server or home server** that may contain **sensitive data** and want to **try out OpenClaw** without giving it broad access to the host environment. The isolated configuration ensures OpenClaw runs in a tightly sandboxed container with minimal privileges.

This project provides a `docker-compose.yml` configuration to run the **OpenClaw Gateway** in an isolated Docker setup. The deployment is designed with the following goals:

- **Isolation** — Runs on a dedicated bridge network (`claw_isolated_net`) with no access to the host network or other containers.
- **Security** — Non-root user execution, all Linux capabilities dropped, privilege escalation disabled, and [Docker Enhanced Container Isolation (ECI)](https://docs.docker.com/security/for-admins/hardened-desktop/enhanced-container-isolation/) for additional runtime protection.
- **Resource Control** — CPU and memory limits prevent the container from consuming excessive host resources.
- **Persistence** — A host volume mount ensures OpenClaw data survives container restarts and upgrades.

---

## Architecture

```
┌──────────────────────────────────────────────┐
│                  Host Machine                │
│                                              │
│   ┌──────────────────────────────────────┐   │
│   │       claw_isolated_net (bridge)     │   │
│   │                                      │   │
│   │   ┌──────────────────────────────┐   │   │
│   │   │   openclaw_gateway_isolated  │   │   │
│   │   │                              │   │   │
│   │   │   Ports: 18789, 18790        │   │   │
│   │   │   CPU:   2 cores (max)       │   │   │
│   │   │   RAM:   4 GB (max)          │   │   │
│   │   │   User:  1000:1000           │   │   │
│   │   └──────────────────────────────┘   │   │
│   │                                      │   │
│   └──────────────────────────────────────┘   │
│                                              │
│   Volume: /home/pi/docker/openclaw            │
│           ↕ mounted to /home/node/.openclaw   │
└──────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement      | Minimum Version |
| ---------------- | --------------- |
| Docker Engine    | 20.10+          |
| Docker Compose   | 1.29+ (v1) or 2.0+ (v2) |
| Git              | 2.0+            |

---

## Quick Start

### 1. Clone This Deployment Repository

```bash
git clone https://github.com/rohitxsh/openclaw-docker-isolated
cd openclaw-docker-isolated
```

### 2. Clone and Build the OpenClaw Docker Image

The `docker-compose.yml` references the image `openclaw:latest`, which must be built locally.

```bash
# Clone the OpenClaw source code
git clone https://github.com/openclaw/openclaw.git

# Build the Docker image locally
cd openclaw
docker build -t openclaw:latest .

# Go back to the deployment directory
cd ..
```

> **Tip:** To rebuild the image later (e.g., after an update), pull the latest source and re-run the build:
> ```bash
> cd openclaw && git pull && docker build -t openclaw:latest . && cd ..
> ```

### 3. Generate a Gateway Token

Generate a secure random token for `OPENCLAW_GATEWAY_TOKEN`:

```bash
openssl rand -hex 32
```

Copy the output you'll use it in the next step.

### 4. Configure Environment Variables

Open `docker-compose.yml` and replace the placeholder values with your actual credentials (use the token generated above for `OPENCLAW_GATEWAY_TOKEN`):

```yaml
environment:
  OPENCLAW_GATEWAY_TOKEN: <token-from-step-3>
  GEMINI_API_KEY: <your-gemini-api-key>
  OPENAI_API_KEY: <your-openai-api-key>
  ANTHROPIC_API_KEY: <your-anthropic-api-key>
  HUGGINGFACE_API_KEY: <your-huggingface-api-key>
  TELEGRAM_BOT_TOKEN: <your-telegram-bot-token>
```

### 5. Prepare the Data Volume

Create the host directory for persistent data and set the correct ownership:

```bash
mkdir -p /home/pi/docker/openclaw
chown -R 1000:1000 /home/pi/docker/openclaw
```

> **Note:** If your host user is different, update the volume path in `docker-compose.yml` accordingly.

### 6. Start the Container

```bash
# Using Docker Compose v2 (recommended)
docker compose up -d

# Using Docker Compose v1
docker-compose up -d
```

> **💡 Tip: Set up an `openclaw` alias:**
>
> Instead of typing the full Docker exec command every time:
> ```bash
> docker exec -it openclaw_gateway_isolated node dist/index.js <cmd>
> ```
> You can create a convenient alias.
>
> **Temporarily** (current terminal session only):
> ```bash
> alias openclaw="docker exec -it openclaw_gateway_isolated node dist/index.js"
> ```
>
> **Permanently** (persists across sessions on Linux):
> ```bash
> echo 'alias openclaw="docker exec -it openclaw_gateway_isolated node dist/index.js"' >> ~/.bashrc
> source ~/.bashrc
> ```
> If you use Zsh, replace `~/.bashrc` with `~/.zshrc`.

### 7. Set Gateway Mode to Local

```bash
openclaw config set gateway.mode local
```

### 8. Configure the Model

Use the interactive `configure` command to select your preferred LLM provider and model:

```bash
openclaw configure
```

> This will walk you through selecting the AI model that OpenClaw should use (e.g., Gemini, OpenAI, Anthropic, etc.). Make sure you have the corresponding API key set in your `docker-compose.yml` environment variables.

### 9. Run OpenClaw Doctor

Verify that the gateway is configured correctly:

```bash
openclaw doctor
```

### 10. Link the Web UI

Approve your browser as a trusted device to access the Web UI:

```bash
# List pending device requests
openclaw devices list

# Approve a device by its request ID
openclaw devices approve <requestId>
```

> For more details, refer to the [Approve a Node Device](https://docs.openclaw.ai/channels/pairing#approve-a-node-device) documentation.

### 11. Link Telegram

Connect your Telegram bot to the gateway:

```bash
# List pending Telegram pairing requests
openclaw pairing list telegram

# Approve a Telegram pairing by code
openclaw pairing approve telegram <CODE>
```

> For more details, refer to the [Telegram Channel](https://docs.openclaw.ai/channels/telegram) documentation.

### 12. Verify the Deployment

```bash
# Check container status
docker compose ps

# View logs
docker compose logs -f openclaw-gateway
```

The OpenClaw Web UI should now be accessible at:

- **Web UI:** `http://127.0.0.1:18789`

> **⚠️ Localhost Only:** The control UI is only accessible from `127.0.0.1` (localhost). To access it remotely or serve it over HTTPS, you will need to set up a reverse proxy. Refer to the [Control UI over HTTP](https://docs.openclaw.ai/gateway/security/index#control-ui-over-http) documentation for details.

---

## Install ClawHub Skills

[ClawHub](https://clawhub.ai/) is a marketplace for OpenClaw skills. Follow these steps to install the `clawhub` CLI and add skills to your gateway.

### 13. Install ClawHub CLI

Install the `clawhub` CLI globally inside the container. The `--cache /tmp/npm-cache` flag avoids permission issues with root-owned cache files from older npm versions:

```bash
docker exec -u root -it openclaw_gateway_isolated npm install -g clawhub --cache /tmp/npm-cache
```

### 14. Log in to ClawHub

Obtain an authentication token from [https://clawhub.ai/](https://clawhub.ai/), then log in:

```bash
docker exec -it openclaw_gateway_isolated clawhub login --token <your-clawhub-token>
```

### 15. Install Skills

Browse skills on [ClawHub](https://clawhub.ai/) and install them by slug:

```bash
docker exec -it openclaw_gateway_isolated clawhub install <slug>
```

> **💡 Tip: Set up a `clawhub` alias:**
>
> Instead of typing the full Docker exec command every time, create a convenient alias.
>
> **Temporarily** (current terminal session only):
> ```bash
> alias clawhub="docker exec -it openclaw_gateway_isolated clawhub"
> ```
>
> **Permanently** (persists across sessions on Linux):
> ```bash
> echo 'alias clawhub="docker exec -it openclaw_gateway_isolated clawhub"' >> ~/.bashrc
> source ~/.bashrc
> ```
> If you use Zsh, replace `~/.bashrc` with `~/.zshrc`.
>
> After setting the alias, you can simply run:
> ```bash
> clawhub install <slug>
> ```

---

## Configuration

### Environment Variables

| Variable                 | Description                                | Required    |
| ------------------------ | ------------------------------------------ | ----------- |
| `HOME`                   | Home directory inside the container        | Yes         |
| `TERM`                   | Terminal type for proper display            | Yes         |
| `OPENCLAW_GATEWAY_TOKEN` | Authentication token for the gateway       | Yes         |
| `GEMINI_API_KEY`         | Google Gemini API key                      | Optional *  |
| `OPENAI_API_KEY`         | OpenAI API key                             | Optional *  |
| `ANTHROPIC_API_KEY`      | Anthropic (Claude) API key                 | Optional *  |
| `HUGGINGFACE_API_KEY`    | Hugging Face API key                       | Optional *  |
| `TELEGRAM_BOT_TOKEN`     | Telegram Bot token for notifications       | Optional    |

> **\*** At least one LLM provider API key is required for the gateway to function.


### Volumes

| Host Path                                    | Container Path           | Purpose                        |
| -------------------------------------------- | ------------------------ | ------------------------------ |
| `/home/pi/docker/openclaw`                   | `/home/node/.openclaw`   | Persistent OpenClaw data store |

> **Important:** Ensure the host directory exists and has the correct ownership (`1000:1000`) before starting the container:
>
> ```bash
> mkdir -p /home/pi/docker/openclaw
> chown -R 1000:1000 /home/pi/docker/openclaw
> ```

### Ports

| Host Port | Container Port | Description                |
| --------- | -------------- | -------------------------- |
| `18789`   | `18789`        | OpenClaw Gateway Web UI    |
| `18790`   | `18790`        | OpenClaw Gateway API port  |

To change the host-side port (e.g., to avoid conflicts), modify the `ports` section:

```yaml
ports:
  - "9000:18789"    # Maps host port 9000 → container port 18789
  - "9001:18790"    # Maps host port 9001 → container port 18790
```

### Resource Limits

The deployment enforces resource constraints to prevent the container from overwhelming the host:

| Resource | Limit  | Description                 |
| -------- | ------ | --------------------------- |
| CPU      | `2.0`  | Maximum of 2 CPU cores      |
| Memory   | `4096M`| Maximum of 4 GB RAM         |

Adjust these in the `deploy.resources.limits` section based on your workload:

```yaml
deploy:
  resources:
    limits:
      cpus: '4.0'      # Allow up to 4 CPU cores
      memory: 8192M     # Allow up to 8 GB RAM
```

---

## Security Hardening

This deployment includes several security best practices:

| Feature                        | Configuration              | Purpose                                                |
| ------------------------------ | -------------------------- | ------------------------------------------------------ |
| **Non-root user**              | `user: "1000:1000"`        | Container runs as a non-root user (UID/GID 1000)       |
| **No new privileges**          | `no-new-privileges: true`  | Prevents child processes from gaining extra privileges  |
| **Drop all capabilities**      | `cap_drop: ALL`            | Removes all Linux kernel capabilities                  |
| **Isolated network**           | `claw_isolated_net`        | Dedicated bridge network, no access to other containers |
| **Init process**               | `init: true`               | Uses `tini` as PID 1 for proper signal handling         |
| **Restart policy**             | `unless-stopped`           | Auto-restarts on failure, stays stopped when manually stopped |
| **Enhanced Container Isolation** | Docker ECI                 | Provides additional runtime isolation between containers and the host via [Docker ECI](https://docs.docker.com/security/for-admins/hardened-desktop/enhanced-container-isolation/) |

---

## Networking

The container runs on an isolated bridge network:

```yaml
networks:
  claw_isolated_net:
    driver: bridge
```

### Key Characteristics

- **Isolation:** Only containers explicitly attached to `claw_isolated_net` can communicate with the OpenClaw gateway.
- **No default bridge:** The container does **not** join Docker's default bridge network, preventing unintended lateral access.
- **DNS resolution:** Containers on the same network can reach each other by service name (e.g., `openclaw-gateway`).

### Adding Additional Services

To add another service that needs to communicate with OpenClaw, attach it to the same network:

```yaml
services:
  my-service:
    image: my-service:latest
    networks:
      - claw_isolated_net
```

---

## Managing the Deployment

### Start

```bash
docker compose up -d
```

### Stop

```bash
docker compose down
```

### Restart

```bash
docker compose restart openclaw-gateway
```

### View Logs

```bash
# Follow logs in real-time
docker compose logs -f openclaw-gateway

# View last 100 lines
docker compose logs --tail 100 openclaw-gateway
```

### Update the Image

```bash
# Pull latest source and rebuild the image
cd openclaw
git pull
docker build -t openclaw:latest .
cd ..

# Recreate the container with the new image
docker compose up -d --force-recreate
```

### Shell Access (Debugging)

```bash
docker compose exec openclaw-gateway /bin/sh
```

---

## Troubleshooting

### Container Fails to Start

1. **Check logs:**
   ```bash
   docker compose logs openclaw-gateway
   ```

2. **Verify the image exists:**
   ```bash
   docker images | grep openclaw
   ```
   If the image is missing, build or pull it first.

3. **Volume permissions:**
   Ensure the mounted volume directory has the correct ownership:
   ```bash
   ls -la /home/pi/docker/openclaw
   # Should be owned by 1000:1000
   ```

### Port Conflicts

If ports `18789` or `18790` are already in use:

```bash
# Find what's using the port
sudo lsof -i :18789

# Change the host-side port mapping in docker-compose.yml
ports:
  - "19000:18789"
  - "19001:18790"
```

### Out of Memory

If the container is being killed (OOMKilled):

```bash
# Check if the container was OOM-killed
docker inspect openclaw_gateway_isolated | grep OOMKilled

# Increase the memory limit in docker-compose.yml
deploy:
  resources:
    limits:
      memory: 8192M
```

### Unauthorized Errors After Running `openclaw onboard`

> **⚠️ Do not run `openclaw onboard` inside the container.** Running it regenerates the `OPENCLAW_GATEWAY_TOKEN`, overwriting the value baked into the container's environment. After this, any requests from the host machine using the original token will fail with **unauthorized** errors.

If you accidentally ran `openclaw onboard`:

1. **Open the OpenClaw config file** on the host:
   ```bash
   nano /home/pi/docker/openclaw/openclaw.json
   ```
   > **Note:** The host path above corresponds to the volume mapping in your `docker-compose.yml`. If you've customized the volume path, use your configured host path instead.
2. **Revert the `gateway.auth.token`** back to original `OPENCLAW_GATEWAY_TOKEN` value (from your `docker-compose.yml`).
3. **Restart the container** to apply the change:
   ```bash
   docker restart openclaw_gateway_isolated
   ```

### Network Issues

```bash
# Inspect the isolated network
docker network inspect openclaw-docker-isolated_claw_isolated_net

# Verify the container is attached
docker inspect openclaw_gateway_isolated --format '{{json .NetworkSettings.Networks}}'
```

---

## Gateway Command Reference

The container starts with the following command:

```bash
node dist/index.js gateway --bind lan --port 18789 --allow-unconfigured
```

| Flag                   | Description                                              |
| ---------------------- | -------------------------------------------------------- |
| `gateway`              | Starts OpenClaw in gateway mode                          |
| `--bind lan`           | Binds the gateway to the LAN interface                   |
| `--port 18789`         | Sets the gateway listening port                          |
| `--allow-unconfigured` | Allows the gateway to start without full configuration   |

---

## License

This deployment configuration is licensed under the [MIT License](LICENSE).

For the OpenClaw project itself, please refer to the [OpenClaw repository](https://github.com/openclaw/openclaw) for licensing information.
