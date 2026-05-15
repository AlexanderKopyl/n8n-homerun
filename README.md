# n8n Homerun

Local-only n8n setup with Docker Compose.

## Architecture

- `n8n` container exposed only on localhost.
- Dedicated `postgres` container for n8n data.
- PostgreSQL is not exposed to the host.
- Internal Docker network only for app/database traffic.
- Persistent named volumes:
  - `n8n_data`
  - `n8n_postgres_data`

Default local URL:

```text
http://n8n-homerun.local:15678
```

## Requirements

```bash
docker --version
docker compose version
```

Check occupied ports before start:

```bash
lsof -iTCP -sTCP:LISTEN -n -P
docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"
lsof -nP -iTCP:15678 -sTCP:LISTEN || echo "Port 15678 is free"
```

## Local domain

Add this line to `/etc/hosts`:

```text
127.0.0.1 n8n-homerun.local
```

macOS command:

```bash
sudo sh -c 'echo "127.0.0.1 n8n-homerun.local" >> /etc/hosts'
```

Verify:

```bash
ping -c 1 n8n-homerun.local
```

## First setup

Create local `.env` from example:

```bash
cp .env.example .env
```

Generate local secrets:

```bash
N8N_ENCRYPTION_KEY="$(openssl rand -hex 32)"
POSTGRES_PASSWORD="$(openssl rand -base64 32 | tr -d '\n' | tr '/+' '_-')"

perl -0777 -i.bak -pe "s/N8N_ENCRYPTION_KEY=.*/N8N_ENCRYPTION_KEY=$N8N_ENCRYPTION_KEY/; s/POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$POSTGRES_PASSWORD/" .env
rm -f .env.bak
```

Validate Compose config:

```bash
docker compose config
```

Start:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Open:

```bash
open http://n8n-homerun.local:15678
```

## Maintenance

Start:

```bash
docker compose up -d
```

Stop without deleting data:

```bash
docker compose stop
```

Logs:

```bash
docker compose logs -f n8n
```

Restart:

```bash
docker compose restart
```

## Backup

Backup PostgreSQL:

```bash
mkdir -p backups

docker compose exec -T postgres pg_dump \
  -U n8n \
  -d n8n \
  > "backups/n8n_postgres_$(date +%Y%m%d_%H%M%S).sql"
```

Backup n8n volume:

```bash
mkdir -p backups

docker run --rm \
  -v n8n_homerun_n8n_data:/data:ro \
  -v "$PWD/backups:/backup" \
  alpine \
  tar czf "/backup/n8n_data_$(date +%Y%m%d_%H%M%S).tar.gz" -C /data .
```

## Update

Backup first, then:

```bash
docker compose pull
docker compose up -d
docker compose logs -f n8n
```

Check version:

```bash
docker compose exec n8n n8n --version
```

For safer updates, replace `N8N_IMAGE=n8nio/n8n:latest` in `.env` with a fixed version tag after the first successful setup.

## Rollback / cleanup

Stop and remove containers/network, keep volumes:

```bash
docker compose down
```

Danger: remove all local n8n data as well:

```bash
docker compose down -v
```

Use `down -v` only if you intentionally want to delete local n8n and PostgreSQL data.
