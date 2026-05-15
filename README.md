# n8n Homerun

Local-only n8n installation with Docker Compose.

## What this setup does

- Runs n8n locally in Docker.
- Uses a dedicated PostgreSQL container.
- Keeps PostgreSQL internal to Docker, without host port binding.
- Exposes n8n only on localhost: `127.0.0.1:15678`.
- Stores all persistent data in named Docker volumes.
- Keeps real local configuration in `.env`, which is ignored by Git.

## Default URL

```text
http://n8n-homerun.local:15678
```

Optional local Nginx HTTPS proxy:

```text
https://n8n-homerun.local
```

## Requirements

```bash
docker --version
docker compose version
```

Check local ports before starting:

```bash
lsof -iTCP -sTCP:LISTEN -n -P
docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"
lsof -nP -iTCP:15678 -sTCP:LISTEN || echo "Port 15678 is free"
```

## Hosts

Add:

```text
127.0.0.1 n8n-homerun.local
```

macOS:

```bash
sudo sh -c 'echo "127.0.0.1 n8n-homerun.local" >> /etc/hosts'
```

## First install

```bash
cp .env.example .env

N8N_ENCRYPTION_KEY="$(openssl rand -hex 32)"
POSTGRES_PASSWORD="$(openssl rand -base64 32 | tr -d '\n' | tr '/+' '_-')"

perl -0777 -i.bak -pe "s|N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$N8N_ENCRYPTION_KEY|; s|POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$POSTGRES_PASSWORD|" .env
rm -f .env.bak

docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

Open:

```bash
open http://n8n-homerun.local:15678
```

## Important after first start

Do not regenerate these `.env` values after the first successful start:

```text
N8N_ENCRYPTION_KEY
POSTGRES_PASSWORD
```

They must stay stable while Docker volumes exist.

## Expected containers

```text
n8n_homerun_app        Up   127.0.0.1:15678->5678/tcp
n8n_homerun_postgres   Up   5432/tcp
```

PostgreSQL should not expose `5432` to the host.

## Optional Nginx HTTPS proxy

Use example config:

```text
nginx/n8n-homerun.local.ssl.conf.example
```

Copy it into your active local Nginx vhost directory and set your real local SSL certificate paths.

Validate and reload:

```bash
nginx -t && nginx -s reload
```

Check vhost:

```bash
curl -Ik --resolve n8n-homerun.local:443:127.0.0.1 https://n8n-homerun.local \
  | grep -Ei "HTTP|X-N8N|x-powered-by|server"
```

If you see `x-powered-by: PHP`, the request is still going to another local PHP/Symfony vhost.

## Maintenance

```bash
# start
docker compose up -d

# stop without deleting data
docker compose stop

# stop and remove containers/network, keep volumes
docker compose down

# logs
docker compose logs -f n8n

# status
docker compose ps
```

## Backup

PostgreSQL:

```bash
mkdir -p backups

docker compose exec -T postgres pg_dump \
  -U n8n \
  -d n8n \
  > "backups/n8n_postgres_$(date +%Y%m%d_%H%M%S).sql"
```

n8n data volume:

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

For safer updates, pin `N8N_IMAGE` in `.env` to a specific version instead of `latest`.

## Troubleshooting

Check logs:

```bash
docker compose logs --tail=120 n8n
```

If n8n reports mismatching encryption keys, restore the original `N8N_ENCRYPTION_KEY` in `.env` and restart n8n.

If PostgreSQL rejects login for user `n8n`, restore the original `POSTGRES_PASSWORD` in `.env` or update the PostgreSQL user password to match the current `.env` value.

If direct URL works but HTTPS domain does not, Docker is fine and the issue is local Nginx vhost routing.

## Cleanup

Safe cleanup:

```bash
docker compose down
```

Danger: delete all local n8n/PostgreSQL data:

```bash
docker compose down -v
```

Use `down -v` only when you intentionally want to delete all local project data.

## Git safety

Do not commit:

```text
.env
backups/
*.sql
*.tar.gz
```
