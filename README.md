# katisha — infisical

Self-hosted [Infisical](https://infisical.com) secrets manager for the Katisha
platform, available at `https://vault.katisha.online`. Runs on `katisha-net`
and uses the shared `db` (PostgreSQL) and `redis` containers.

---

## How it works

```
push to main
    └─ GitHub Actions
           └─ SSH into server
                  ├─ git clone (first time) or git pull
                  ├─ write .env from secrets
                  ├─ docker compose up -d --pull always
                  │      ├─ infisical-db-migration  →  runs migrations, exits
                  │      └─ infisical               →  starts after migrations
                  └─ waits for healthy status
```

The `infisical` postgres user and database are provisioned by the **db service**
(`db/init/03-infisical.sql`), not by this pipeline. The db service must be
deployed before this service.

---

## Repository layout

```
infisical/
├── docker-compose.yml           # infisical + migration runner
├── .env.example                 # template — copy to .env for local dev
├── .github/workflows/deploy.yml
└── README.md
```

---

## Secrets — what lives where

### GitHub Actions secrets (this repo)

All infisical secrets stay in GitHub Actions because this service IS the
secrets manager — it cannot fetch its own bootstrap credentials from itself.

| Secret | Description | Notes |
|---|---|---|
| `SERVER_HOST` | Server IP or hostname | |
| `SERVER_USER` | SSH username | |
| `SERVER_SSH_KEY` | Private SSH key | |
| `INFISICAL_ENCRYPTION_KEY` | Encrypts all secrets at rest | **Never change after first run** |
| `INFISICAL_AUTH_SECRET` | Signs JWTs | Changing invalidates all sessions |
| `INFISICAL_DB_PASSWORD` | Password for the `infisical` postgres user | Must match `INFISICAL_DB_PASSWORD` in the **db repo** secrets |
| `REDIS_PASSWORD` | Must match `REDIS_PASSWORD` in the redis repo | |
| `SMTP_HOST` | SMTP server hostname | |
| `SMTP_PORT` | SMTP port (typically 587) | |
| `SMTP_FROM_ADDRESS` | From address for emails | |
| `SMTP_FROM_NAME` | From name for emails | |
| `SMTP_USERNAME` | SMTP auth username | |
| `SMTP_PASSWORD` | SMTP auth password | |

### Why not Infisical for these?

Infisical itself uses postgres and redis as its backend — it must be
bootstrapped before it can serve any secrets. All credentials for this service
therefore live in GitHub Actions secrets, which are available without any
running service.

Once Infisical is up, all other services (cdn, user-service, api-gw, etc.)
fetch their runtime credentials from Infisical.

---

## Critical secrets — read before first deploy

| Secret | Warning |
|---|---|
| `INFISICAL_ENCRYPTION_KEY` | Encrypts all secrets at rest. **Never change after first run** — existing secrets become unrecoverable. |
| `INFISICAL_AUTH_SECRET` | Signs JWTs. Changing it invalidates all active sessions. |

Generate fresh values before setting:

```bash
openssl rand -hex 16    # ENCRYPTION_KEY
openssl rand -base64 32 # AUTH_SECRET
```

---

## Database

The `infisical` postgres user and database are created by the **db service**
deploy pipeline (`db/init/03-infisical.sql`). This pipeline no longer touches
the database directly. The `INFISICAL_DB_PASSWORD` set here must match the
one set in the db repo's GitHub Actions secrets.

Database migrations run automatically on every deploy via the
`infisical-db-migration` container, which exits after completing.

---

## Local development

```bash
cp .env.example .env
# Fill in ENCRYPTION_KEY, AUTH_SECRET, DB_CONNECTION_URI, REDIS_URL
docker compose up -d
```

Requires the shared `db` and `redis` containers running on `katisha-net`.
