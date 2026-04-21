# SuperSync Docker Deployment

This repository contains the GitHub Actions workflow and Docker Compose configuration to automate the deployment of the `super-sync-server` for Super Productivity.

## CI/CD GitHub Action Automation

The custom GitHub Action in this repository stays in sync with the upstream code by:
1. **Daily Checks**: Checking the [official repository releases](https://github.com/super-productivity/super-productivity/releases) once a day at 2 AM UTC.
2. **Publishing**: If it's a new tag, it pulls down the official source code, builds the container, and publishes it seamlessly to GitHub Container Registry. 

## Setup Instructions

### 1. Environment Variables

We use an `.env` file to manage configuration for both the database and the backend. Create an `.env` file and populate it with the following:

```env
POSTGRES_USER=syncuser
POSTGRES_PASSWORD=syncpassword
POSTGRES_DB=sync_db

# Required for server
PORT=1900
DATABASE_URL=postgresql://syncuser:syncpassword@postgres:5432/sync_db
JWT_SECRET=change_this_to_a_minimum_32_character_secret_key!
PUBLIC_URL=http://localhost:1900
CORS_ORIGINS=https://app.super-productivity.com
```

**Important Configuration Steps:**

* **`JWT_SECRET`**: Must be a secure random string (minimum 32 characters). You can generate one quickly in your terminal using:

  ```bash
  openssl rand -base64 32
  ```
* **`PUBLIC_URL`**: Update this to match the actual public deployment URL of your sync server (e.g., `https://sync.yourdomain.com`).
* **`CORS_ORIGINS`**: Define which web domains are allowed to access your sync server. Use `https://app.super-productivity.com` if using the official web app, your own domain if self-hosting the web client, or you can remove it if you are exclusively using the desktop/mobile apps.


### 2. Docker Compose Configuration

Ensure you have your `compose.yml` file matching this setup so it pulls the correct image and connects to the `.env` file:

```yaml
services:
  supersync:
    image: ghcr.io/warreth/super-sync-server:latest
    container_name: supersync
    ports:
      - "${PORT:-1900}:${PORT:-1900}"
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    container_name: supersync_postgres
    env_file:
      - .env
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-syncuser} -d ${POSTGRES_DB:-sync_db}"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

### 3. Starting the Server

Run the following command to download the latest container image and start the setup:
```bash
docker compose up -d
```
