# n8n on AWS (EC2 + Docker Compose)

Single-node [n8n](https://n8n.io) running on an EC2 instance with Docker +
Docker Compose, fronted by Caddy for automatic HTTPS. Infrastructure is managed
with CloudFormation and deployed from GitHub Actions using OIDC (no long-lived
AWS keys).

> The server runs Docker because Amazon Linux 2023 ships Docker (not Podman) in
> its base repos. The same `compose.yml` works with `podman compose` locally.

- **Compute:** 1x EC2 (Amazon Linux 2023), Docker + Docker Compose v2
- **Stack:** n8n + Postgres 16 + Caddy (reverse proxy + Let's Encrypt TLS)
- **Domain:** `n8n.londoso.com` (Route53 A record → Elastic IP)
- **Region:** `us-east-1`
- **CI/CD:** GitHub Actions → OIDC role `github-oidc-provider-aws` → CloudFormation + SSM

## Repository layout

```
infra/cloudformation.yaml     # VPC, subnet, SG, EC2, EIP, IAM (SSM), Route53
compose.yml                   # n8n + postgres + caddy
Caddyfile                     # reverse proxy + auto TLS
.env.example                  # env template (real values come from GitHub Secrets)
.github/workflows/deploy.yml  # provision infra + deploy compose via SSM
```

## Architecture

```
Internet ──HTTPS──▶ Caddy :443 ──▶ n8n :5678
                     (Let's Encrypt)      │
                                          ▼
                                   Postgres :5432
   (all containers run via docker compose on one EC2 instance)
```

Only ports 80/443 are open to the internet. Postgres and n8n are reachable only
inside the compose network. Administrative access uses **SSM Session Manager**
(no SSH, no key pairs, no open port 22).

## Prerequisites

1. The Route53 hosted zone for `londoso.com` already exists in this account.
2. The OIDC role `arn:aws:iam::862807499233:role/github-oidc-provider-aws` trusts
   this GitHub repo and has permissions for CloudFormation, EC2, IAM (create the
   instance role + instance profile, PassRole to EC2), Route53, and SSM
   `SendCommand`. (These are bundled in the `n8n-deploy` managed policy.)
3. GitHub repository **Secrets** are configured (see below).

## Required GitHub Secrets

Set these under **Settings → Secrets and variables → Actions**:

| Secret | Example | Notes |
|---|---|---|
| `HOSTED_ZONE_ID` | `Z09788721KJRYG10EK3E2` | Route53 zone id for `londoso.com` |
| `ACME_EMAIL` | `you@londoso.com` | Let's Encrypt expiry notices |
| `POSTGRES_USER` | `n8n` | |
| `POSTGRES_PASSWORD` | strong random | |
| `POSTGRES_DB` | `n8n` | |
| `N8N_ENCRYPTION_KEY` | `openssl rand -hex 32` | **keep stable forever** |
| `GENERIC_TIMEZONE` | `America/Bogota` | |

The editor is protected by n8n's built-in user management: the first time you
open `https://n8n.londoso.com` it prompts you to create the owner account.

> The `N8N_ENCRYPTION_KEY` encrypts stored credentials. If it changes, existing
> credentials become unreadable. Generate it once and never rotate it casually.

## Deploy

Run the workflow manually via **Actions → Deploy n8n → Run workflow**
(or `gh workflow run deploy.yml`). The pipeline:

1. **infra** — `aws cloudformation deploy` creates/updates the stack and outputs
   the instance id. The Route53 A record for `n8n.londoso.com` is created here.
2. **deploy** — waits for the instance to register with SSM, builds `.env` from
   secrets, tars `compose.yml` + `Caddyfile` + `.env`, ships them to `/opt/n8n`
   on the instance, and restarts the `n8n-stack` systemd service (which runs
   `docker compose up -d`).

First run takes a few minutes: EC2 boot + Docker install + image pulls + Caddy
requesting the TLS cert. Once DNS resolves to the EIP, open:

```
https://n8n.londoso.com
```

## Local testing (podman compose)

You can run the exact same stack locally before deploying:

```bash
cp .env.example .env
# edit .env: set DOMAIN_NAME=localhost and generate N8N_ENCRYPTION_KEY
podman compose up -d
podman compose ps
podman compose logs -f n8n
```

For pure-local use, TLS via Let's Encrypt won't work on `localhost`; either use
Caddy's `local_certs` or hit n8n directly. To reach n8n without Caddy locally,
you can temporarily add `ports: ["5678:5678"]` to the `n8n` service.

## Operations

Connect to the instance without SSH:

```bash
aws ssm start-session --target <INSTANCE_ID> --region us-east-1
```

Common commands on the host:

```bash
cd /opt/n8n
docker compose ps
docker compose logs -f n8n
systemctl restart n8n-stack.service   # restart whole stack
```

## Data & backups

Persistent data lives in named volumes on the instance's encrypted gp3 root
volume:

- `postgres_data` — n8n database
- `n8n_data` — n8n home (`.n8n`)
- `caddy_data` — issued TLS certificates

For a quick backup, dump Postgres:

```bash
docker exec -t n8n-postgres-1 pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" > backup.sql
```

For durability, consider moving Postgres to RDS later (the compose env vars
already point at a `DB_POSTGRESDB_HOST`, so it's a small change).

## Cost note

Roughly a single `t3.small` + EBS + Elastic IP. Stop the instance when idle to
save on compute (the EIP stays attached, so DNS keeps working on next start).

## Tear down

```bash
aws cloudformation delete-stack --stack-name n8n --region us-east-1
```

This removes the EC2 instance, EIP, Route53 record, and VPC. Container volumes
are destroyed with the instance, so back up first if needed.
