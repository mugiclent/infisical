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
                  ├─ create infisical postgres user + database (idempotent)
                  ├─ docker compose up -d --pull always
                  │      ├─ infisical-db-migration  →  runs migrations, exits
                  │      └─ infisical               →  starts after migrations
                  └─ waits for healthy status
```

No dedicated postgres or redis containers — both are shared with the rest of
the Katisha platform on `katisha-net`.

---

## Repository layout

```
infisical/
├── docker-compose.yml           # infisical + migration runner
├── .env.example                 # template — copy to .env for local dev
├── .github/
│   └── workflows/
│       └── deploy.yml           # CI/CD pipeline
└── README.md
```

---

## Critical secrets — read before first deploy

| Secret | Warning |
|---|---|
| `INFISICAL_ENCRYPTION_KEY` | Encrypts all secrets at rest. **Never change after first run** — existing secrets become unrecoverable. |
| `INFISICAL_AUTH_SECRET` | Signs JWTs. Changing it invalidates all active sessions. |

Generate fresh values before setting the secrets:

```bash
# ENCRYPTION_KEY — 32 hex characters
openssl rand -hex 16

# AUTH_SECRET — base64 string
openssl rand -base64 32
```

Set them as GitHub Actions secrets, then never change them.

---

## GitHub Actions secrets

| Secret | Description |
|---|---|
| `SERVER_HOST` | Server IP or hostname |
| `SERVER_USER` | SSH username |
| `SERVER_SSH_KEY` | Private SSH key |
| `INFISICAL_ENCRYPTION_KEY` | Platform encryption key — generate once, never rotate |
| `INFISICAL_AUTH_SECRET` | JWT signing secret — generate once, never rotate |
| `INFISICAL_DB_PASSWORD` | Password for the `infisical` postgres user |
| `REDIS_PASSWORD` | Must match the `REDIS_PASSWORD` secret in the redis repo |
| `SMTP_HOST` | SMTP server hostname |
| `SMTP_PORT` | SMTP port (typically 587) |
| `SMTP_FROM_ADDRESS` | From address for emails |
| `SMTP_FROM_NAME` | From name for emails |
| `SMTP_USERNAME` | SMTP auth username |
| `SMTP_PASSWORD` | SMTP auth password |

---

## Database

The deploy pipeline automatically creates the `infisical` PostgreSQL user and
database on the shared `db` container if they don't exist. Nothing needs to be
done manually.

Database migrations run automatically on every deploy via the
`infisical-db-migration` container, which exits after completing.

---

## Nginx config

Infisical listens on port `8080` inside `katisha-net`. It has no public port.
Add this to the nginx config when deploying the nginx container:

```nginx
server {
    listen 80;
    server_name vault.katisha.online;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name vault.katisha.online;

    ssl_certificate     /etc/letsencrypt/live/vault.katisha.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vault.katisha.online/privkey.pem;

    ssl_protocols             TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_session_cache         shared:SSL:10m;

    location / {
        proxy_pass         http://infisical:8080;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

---

## Local development

```bash
cp .env.example .env
# Fill in ENCRYPTION_KEY, AUTH_SECRET, DB_CONNECTION_URI, REDIS_URL
docker compose up -d
```

Local dev requires the shared `db` and `redis` containers to be running on
`katisha-net`, or override `DB_CONNECTION_URI` and `REDIS_URL` to point at
local instances.
