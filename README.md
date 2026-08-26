# n8n Homerun

Local-only n8n installation with Docker Compose.

## What this setup does

- Runs n8n locally in Docker.
- Uses a dedicated PostgreSQL container.
- Keeps PostgreSQL internal to Docker, without host port binding.
- Exposes n8n only on localhost: `127.0.0.1:15678`.
- Stores persistent data inside the local project directory under `./data/`.
- Keeps real local configuration in `.env`, which is ignored by Git.

## Default URL

```text
http://n8n-homerun.local:15678
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

mkdir -p data/n8n data/postgres

docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

Open:

```bash
open http://n8n-homerun.local:15678
```

## Data layout

Persistent data is stored in the project directory:

```text
data/
  n8n/       # n8n config and local files
  postgres/  # PostgreSQL database files
```

The `data/` directory is ignored by Git.

Workflow definitions are kept separately, in Git:

```text
workflows/   # workflow JSON, committed
```

## Workflows

Workflows are authored as JSON files in `workflows/` and imported into n8n:

```text
Workflows -> Import from File
```

After editing a workflow in the UI, download it and save it back over the file in `workflows/`.

Credentials are never stored in workflow JSON. Create them once in the UI under `Settings -> Credentials`.

Full conventions, JSON shape, and webhook URL rules: [docs/workflows/authoring.md](docs/workflows/authoring.md).

### Available workflows

```text
analytics-slack-report.json   Scheduled Athena query -> Slack Block Kit report
```

Configuration and testing: [docs/workflows/analytics-slack-report.md](docs/workflows/analytics-slack-report.md).

It is self-contained: all deployment settings live in a `Config` node inside the workflow, so
it imports and runs on any n8n instance without touching this project's `.env` or
`docker-compose.yml`. Athena is reached by assuming a cross-account role: the n8n `AWS (Assume Role)` credential
calls `sts:AssumeRole` with the IAM user's keys and signs Athena with the temporary
credentials. Created once per instance. The Slack webhook is filled in on the instance and never committed.

## Migrating from old Docker volumes

If this project was previously using Docker named volumes, copy existing n8n files into `./data/n8n` before relying on the local directory setup.

```bash
docker compose down

mkdir -p data/n8n data/postgres backups

docker run --rm \
  -v n8n_homerun_n8n_data:/old:ro \
  -v "$PWD/data/n8n:/new" \
  alpine sh -c 'cp -a /old/. /new/ && chown -R 1000:1000 /new'

docker compose up -d
docker compose ps
```

If the database also still exists only in the old PostgreSQL volume, create a dump before switching and restore it into the new `./data/postgres` database.

## Important after first start

Do not regenerate these `.env` values after the first successful start:

```text
N8N_ENCRYPTION_KEY
POSTGRES_PASSWORD
```

They must stay stable while local data exists.

## Expected containers

```text
n8n_homerun_app        Up   127.0.0.1:15678->5678/tcp
n8n_homerun_postgres   Up   5432/tcp
```

PostgreSQL should not expose `5432` to the host.

## Maintenance

```bash
# start
docker compose up -d

# stop without deleting data
docker compose stop

# stop and remove containers/network, keep local data
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

Local data directory:

```bash
mkdir -p backups

tar czf "backups/n8n_local_data_$(date +%Y%m%d_%H%M%S).tar.gz" data
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

If the browser shows `Cannot GET /` after moving to `./data/`, verify that `data/n8n` is not empty and copy the old `n8n_homerun_n8n_data` volume into it.

## Cleanup

Safe cleanup:

```bash
docker compose down
```

Danger: delete all local project data:

```bash
docker compose down
rm -rf data
```

Use `rm -rf data` only when you intentionally want to delete all local n8n and PostgreSQL data.

## Git safety

Do not commit:

```text
.env
backups/
data/
*.sql
*.tar.gz
```
