[![Total downloads](https://img.shields.io/badge/dynamic/json?label=Total%20downloads&url=https://ipitio.github.io/backage/warreth/super-sync-docker.json&query=$.package.downloads&color=blue)](https://github.com/warreth/super-sync-docker/packages)

# SuperSync Docker Deployment

This repository provides:

1. A GitHub Actions workflow to build and publish Super Sync images to GHCR.
2. A production-ready Docker Compose setup for running Super Sync with PostgreSQL.

## CI/CD Automation

The workflow tracks official upstream releases from:

https://github.com/super-productivity/super-productivity/releases

Behavior:

1. Checks for upstream updates on schedule.
2. Builds from official upstream source/tag.
3. Publishes container images to GHCR.

## Production Deployment

### 1. Configure .env

Use the root `.env` file:

```env
# Required secrets
# Generate with: openssl rand -base64 32
JWT_SECRET=replace-with-a-random-secret
POSTGRES_PASSWORD=replace-with-a-strong-password

# Database
POSTGRES_USER=postgres
POSTGRES_DB=supersync

# Server
PORT=1900
PUBLIC_URL=https://sync.your-domain.com
CORS_ORIGINS=https://app.super-productivity.com

# WebAuthn/Passkeys Configuration. MUST BE SAME AS WEB-URL!
WEBAUTHN_RP_ID=sync.your-domain.com

# SMTP / Email Configuration (Required in production)
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=your-smtp-username
SMTP_PASS=your-smtp-password
SMTP_FROM="Super Productivity Sync"
```

> [!NOTE]
> 1. `JWT_SECRET` should be a strong random value (minimum 32 chars).
> 2. `PUBLIC_URL` is the external URL users access.
> 3. **Self-hosting / Fake SMTP for testing:** If you don't have a real SMTP server and just want to test or self-host without a valid email sender, you can use [SMTP Bucket](https://www.smtpbucket.com/). To do this, configure: `SMTP_HOST=mail.smtpbucket.com`, `SMTP_PORT=8025`, and **leave `SMTP_USER` and `SMTP_PASS` entirely blank**. Then, simply open the SMTP Bucket website to read the verification emails sent to your users/yourself using the search feature.

### 2. Production compose.yml

Use the root `compose.yml`:

```yaml
services:
  supersync:
    image: ghcr.io/warreth/super-sync-server:latest
    container_name: supersync
    command: sh -c "npx prisma db push --accept-data-loss && npm start"
    environment:
      - NODE_ENV=production
      - PORT=${PORT:-1900}
      - DATABASE_URL=postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD}@supersync_postgres:5432/${POSTGRES_DB:-supersync}
      - JWT_SECRET=${JWT_SECRET}
      - PUBLIC_URL=${PUBLIC_URL}
      - CORS_ORIGINS=${CORS_ORIGINS:-https://app.super-productivity.com}
      - WEBAUTHN_RP_ID=${WEBAUTHN_RP_ID}
      - WEBAUTHN_RP_NAME="Super Productivity Sync"
      - WEBAUTHN_ORIGIN=${PUBLIC_URL}
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=${SMTP_PORT}
      - SMTP_SECURE=${SMTP_SECURE}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASS=${SMTP_PASS}
      - SMTP_FROM=${SMTP_FROM}
    ports:
      - "${PORT:-1900}:${PORT:-1900}"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - default

  postgres:
    image: postgres:15-alpine
    container_name: supersync_postgres
    environment:
      - POSTGRES_USER=${POSTGRES_USER:-postgres}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB:-supersync}
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-supersync}"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true

networks:
  default:
```

> [!IMPORTANT]
> - PostgreSQL data persists in `./postgres_data`.
> - Set tag to :master to try out beta versions (untested)

### 3. Start

From repo root:

```bash
mkdir -p ./postgres_data
docker compose -f compose.yml up -d
```

Check logs:

```bash
docker compose -f compose.yml logs -f
```
