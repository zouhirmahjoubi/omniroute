---
title: "OmniRoute Dokploy Deployment Guide"
version: 3.8.46
lastUpdated: 2026-07-08
---

This document describes how to deploy OmniRoute on **Dokploy** (an open-source Heroku/Coolify alternative) directly from a GitHub repository.

Dokploy supports two primary deployment methods for OmniRoute:

- **Option A (Recommended - Docker Compose):** Deploys OmniRoute and a Redis rate-limiter sidecar together using `docker-compose.dokploy.yml`.
- **Option B (Single Application):** Deploys OmniRoute as a single Dokploy Application using the `Dockerfile` and links to a separately created Dokploy Redis database.

---

## 1. Prerequisites

Before starting, ensure you have:

1. A running instance of **Dokploy** (v0.12.0 or higher recommended).
2. A GitHub account with a fork or copy of the [OmniRoute](https://github.com/diegosouzapw/OmniRoute) repository.
3. Dokploy connected to your GitHub account (under **Settings -> Git Providers**).

---

## 2. Option A: Deploying via Docker Compose (Recommended)

Using Docker Compose allows you to deploy both OmniRoute and Redis in a single stack with pre-configured internal networking.

### Step 1: Create a Compose Service

1. Log in to your Dokploy Dashboard.
2. Go to **Projects** and select an existing project or click **Create Project**.
3. Inside the project, click **Create Service** and select **Compose**.
4. Give your service a name (e.g., `omniroute-stack`).

### Step 2: Configure Git Source

1. In the service settings, select **GitHub** as the provider.
2. Select your repository (e.g., `yourusername/OmniRoute`) and the target branch (e.g., `main`).
3. Set the **Compose Path** to:

   ```text
   docker-compose.dokploy.yml
   ```

4. Click **Save**.

### Step 3: Configure Environment Variables

1. Go to the **Environment** tab.
2. Add your required secrets (see [Section 4: Environment Variables](#4-environment-variables) below). Dokploy automatically saves these into a `.env` file in the service directory, which is read by `docker-compose.dokploy.yml`.
3. Click **Save**.

### Step 4: Configure Domains

1. Go to the **Domains** tab in Dokploy.
2. Add a new domain (e.g., `omniroute.example.com`).
3. Set the **Path** to `/` and select the target service `omniroute` on port `20128`.
4. Enable HTTPS/SSL.

### Step 5: Deploy

1. Go to the **Deployments** tab and click **Deploy**.
2. Dokploy will clone the repository, build the Docker images, and start the containers.

---

## 3. Option B: Deploying as a Standalone Application

If you prefer to manage the database/Redis services separately, you can deploy OmniRoute as a standalone Dokploy Application and connect it to a Dokploy-managed Redis database.

### Step 1: Create a Redis Database in Dokploy

1. In your Dokploy project, click **Create Service** and select **Redis**.
2. Name it (e.g., `omniroute-redis`).
3. Once deployed, note down the internal connection string or credentials under the **Connection** or **Environment** tab.

### Step 2: Create an Application Service

1. Click **Create Service** and select **Application**.
2. Name it (e.g., `omniroute-app`).

### Step 3: Configure Git Source and Build settings

1. Under **Source**, select **GitHub**, choose your repository and branch.
2. Under **Build**, set:
   - **Build Type:** `Dockerfile`
   - **Dockerfile Path:** `Dockerfile` (default)
   - **Target Stage:** `runner-base` (or `runner-web` if you require web-cookie/session providers)
3. Click **Save**.

### Step 4: Map Persistent Volume

OmniRoute uses SQLite for data persistence. You must mount a persistent volume to preserve the database across deployments:

1. Go to the **Volumes** tab.
2. Click **Add Volume**.
3. Configure the mount:
   - **Volume Name:** `omniroute-data`
   - **Mount Path:** `/app/data`
4. Click **Save**.

### Step 5: Configure Environment Variables

1. Go to the **Environment** tab.
2. Set the environment variables, including `REDIS_URL` pointing to the Redis database created in Step 1 (e.g., `redis://omniroute-redis:6379`). See [Section 4: Environment Variables](#4-environment-variables) for details.
3. Click **Save**.

### Step 6: Expose Port and Domain

1. Go to the **Port** or **Domains** tab.
2. Map your domain to the application container's internal port `20128`.
3. Enable HTTPS/SSL.

### Step 7: Deploy

1. Click **Deploy**.

---

## 4. Environment Variables

Configure the following environment variables in the Dokploy UI:

### 4.1 Required Production Variables

| Variable           | Description / Value                                                                   |
| :----------------- | :------------------------------------------------------------------------------------ |
| `JWT_SECRET`       | Required for dashboard session signing. Generate via `openssl rand -base64 48`.       |
| `API_KEY_SECRET`   | Encryption key for API keys stored in SQLite. Generate via `openssl rand -hex 32`.    |
| `INITIAL_PASSWORD` | The initial admin password on first boot. Change immediately after logging in.        |
| `DATA_DIR`         | Directory for the SQLite database. Must be set to `/app/data` (matches volume mount). |

### 4.2 Database & Redis Variables

| Variable                     | Description / Value                                                                                                       |
| :--------------------------- | :------------------------------------------------------------------------------------------------------------------------ |
| `REDIS_URL`                  | Redis connection URL. For Option A (Compose), use `redis://redis:6379`. For Option B, use your Dokploy Redis credentials. |
| `STORAGE_ENCRYPTION_KEY`     | Optional encryption key for SQLite database at rest. Generate via `openssl rand -hex 32`.                                 |
| `DISABLE_SQLITE_AUTO_BACKUP` | Set to `false` to enable automatic database backup on startup.                                                            |

### 4.3 Network & Ports (Defaults matched to Docker config)

| Variable                   | Description / Value       |
| :------------------------- | :------------------------ |
| `PORT`                     | Set to `20128`.           |
| `HOSTNAME`                 | Set to `0.0.0.0`.         |
| `OMNIROUTE_MIGRATIONS_DIR` | Set to `/app/migrations`. |

---

## 5. Automated Deployments (Webhooks)

To enable automatic deployment when you push code to GitHub:

1. Go to your Dokploy service (Compose or Application) and find the **Webhooks** or **Auto Deploy** tab.
2. Copy the webhook URL.
3. In your GitHub repository, go to **Settings -> Webhooks -> Add webhook**.
4. Paste the webhook URL into **Payload URL**, set **Content type** to `application/json`, and select the **Just the push event** trigger.
5. Click **Add webhook**.

---

## 6. Troubleshooting

### 6.1 Memory Exhaustion & Build Failures (OOM)

Compiling Next.js App Router projects can consume substantial memory (~1.5GB to 2GB RAM). If your VPS has 1GB or 2GB of RAM and no SWAP space configured, Docker builds will frequently crash or hang during the build stage.

> [!IMPORTANT]
> Always configure a swap file on your host VPS to prevent Out-Of-Memory (OOM) situations during Next.js builds.

To set up a 4GB swap space on your Ubuntu/Debian host VPS:

```bash
# Check existing swap
sudo swapon --show

# Create a 4GB swap file
sudo fallocate -l 4G /swapfile

# Set correct permissions
sudo chmod 600 /swapfile

# Set up swap area
sudo mkswap /swapfile

# Enable swap
sudo swapon /swapfile

# Persist across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify it works
free -h
```

### 6.2 Bypassing Build Tool Installation (g++ / make / python3)

By default, the Dockerfile builds the `better-sqlite3` native addon using prebuilt binaries. It skips installing compiler tools (`python3`, `make`, `g++`) to make builds faster and less resource-heavy.

If you are deploying on a non-standard platform or architecture where prebuilt binaries are unavailable, the build might fail during dependency installation.

To force compilation from source:

1. In Dokploy, go to your service's **Build** settings.
2. Add a new **Build Argument**:
   - **Key:** `OMNIROUTE_INSTALL_BUILD_TOOLS`
   - **Value:** `1`
3. Click **Save** and redeploy.
