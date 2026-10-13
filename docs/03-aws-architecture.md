# As-Built AWS Architecture — Phase 1

> Challenge goal: draw a simple architecture diagram to host the application on AWS. This document records the architecture **as built and validated** on 2026-10-12 (all endpoint tests green through the public URL).

## Diagram

- **Diagram source (draw.io, editable):** [`03-aws-architecture.drawio`](03-aws-architecture.drawio)
- **Diagram link (published, Miro / diagrams.net):** _TODO_
- Exported image: _TODO (`evidence/architecture-diagram.png`)_

## Region and network

- **Region:** us-east-1
- **VPC:** `prod-vpc` (`vpc-08be8772fc7f33a50`) — `10.25.0.0/16` — DNS support: enabled · DNS hostnames: disabled
- **Internet Gateway:** `prod-igw` (`igw-0c99d89950b3f7ee2`), attached to the VPC

## Subnets

| Subnet | ID | CIDR | AZ | Route table | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `prod-public-a` | `subnet-076513532c0bf3dbe` | 10.25.0.0/24 | us-east-1a | `prod-public-route` (explicit) | EC2 — public entry point |
| `prod-db-a` | `subnet-0880de6a95d4e41c3` | 10.25.10.0/24 | us-east-1a | Main (implicit) | RDS — active placement |
| `prod-db-b` | `subnet-0feff9e5bf86a917b` | 10.25.11.0/24 | us-east-1b | Main (implicit) | DB subnet group member (no instance — Single-AZ) |

Auto-assign public IPv4 is **disabled on all subnets** — the public address comes from the Elastic IP, not from auto-assign.

## Route tables

| Route table | ID | Association | Routes |
| :--- | :--- | :--- | :--- |
| `prod-public-route` | `rtb-0bc8f3f381c7c618c` | explicit: `prod-public-a` | `10.25.0.0/16 → local` · `0.0.0.0/0 → prod-igw` |
| Main | `rtb-0e6b8ed5be16703f4` | none explicit (`prod-db-a`/`prod-db-b` inherit) | `10.25.0.0/16 → local` — **no internet route** |

The main route table stays local-only: the database tier has **no internet route at all**.

## Security Groups (least privilege)

### `prod-app-sg` (`sg-094c00bc445979161`) — attached to the EC2 instance

| Direction | Rule |
| :--- | :--- |
| Inbound | `80/tcp ← 0.0.0.0/0` — ACME HTTP-01 + HTTP→HTTPS redirect |
| Inbound | `443/tcp ← 0.0.0.0/0` — application over TLS |
| Inbound | `22/tcp ← admin IP /32` — SSH, key-only (IP masked in this doc) |
| Outbound | default (allow all) |

### `prod-db-sg` (`sg-029efc45befb33906`) — attached to the RDS instance

| Direction | Rule |
| :--- | :--- |
| Inbound | `5432/tcp ← sg-094c00bc445979161` (`prod-app-sg`) — **SG reference, not CIDR** |
| Outbound | default (allow all) |

The application port `5000` is never exposed to the internet: Caddy proxies `127.0.0.1:5000` on the host, and `prod-app-sg` does not allow 5000 inbound.

## Compute (EC2)

- **Instance:** `srv-togglemaster-prod` (`i-0f19b69e143c302a3`) — `t3.micro` — `prod-public-a` (us-east-1a) — private IP `10.25.0.100`
- **AMI:** Ubuntu 24.04 LTS (noble), `ami-0f8a61b66d1accaee`
- **IMDSv2 required** (IMDSv1 requests receive 401)
- **SSH:** key pair only (user `ubuntu`), no password auth, restricted to the admin IP `/32`
- **IAM instance profile:** none
- **Elastic IP:** `44.223.184.20` (`eipalloc-072324b90775cf0ad`) attached to the instance ENI
- **Docker 29.9 + Compose v5.6:**
  - `caddy` (image `caddy:2`, host network) at `/opt/proxy` — TLS termination (Let's Encrypt, ACME HTTP-01 on :80), HTTP→HTTPS redirect, `reverse_proxy 127.0.0.1:5000`; volumes `caddy_data` + `caddy_config`
  - `app` (gunicorn, :5000) at `/opt/togglemaster` (repo clone) — **only the `app` service runs** (`docker compose up -d --no-deps app`): no local PostgreSQL on AWS
  - Database config injected via `docker-compose.override.yml` on the server (`chmod 600`, root-owned, outside git) — 12-Factor: config in the environment

## Database (RDS)

- **Instance:** `database-1` — PostgreSQL 18.3 — `db.t4g.micro` — Single-AZ — 20 GiB gp2 — us-east-1a (`prod-db-a`)
- **DB subnet group:** `prod-db-subnet` = `prod-db-a` + `prod-db-b` (RDS requires subnets in ≥ 2 AZs, even for Single-AZ)
- **Publicly accessible: No** — endpoint resolvable/reachable only inside the VPC
- **Security group:** `prod-db-sg`
- **Master credentials:** managed by RDS in AWS Secrets Manager (user `postgres`; password never stored in this repo or in compose files)
- **Database:** `togglemaster`, table `flags` (created by `flask init-db` on first boot)

## DNS and TLS

- **Cloudflare:** A record `togglemaster.cheb.com.br` → `44.223.184.20`
- **TLS:** Caddy + Let's Encrypt (ACME HTTP-01), automatic renewal, HTTP→HTTPS redirect
- **Public URL:** https://togglemaster.cheb.com.br/health

## Traffic flow

```text
Users --HTTPS--> Cloudflare DNS --> EIP 44.223.184.20 --> prod-igw --> prod-public-a
  --> prod-app-sg (80/443) --> Caddy :443 (TLS) --> 127.0.0.1:5000 --> app (gunicorn)
  --> prod-db-sg (5432) --> RDS PostgreSQL 18 (prod-db-a, isolated subnet)

Admin --SSH :22 (admin IP/32, key-only)--> EC2
Let's Encrypt --ACME HTTP-01 :80--> Caddy (certificate issuance/renewal)
```

## Validation (2026-10-12, via https://togglemaster.cheb.com.br)

| Test | Result |
| :--- | :--- |
| `GET /health` | 200 `{"status":"ok"}` |
| `POST /flags` (create) | 201 |
| `GET /flags` (list) | 200 |
| `GET /flags/<name>` | 200 |
| `PUT /flags/<name>` (update) | 200 |
| Duplicate `POST /flags` | 409 |
| App container restart → data persisted | OK (stored in RDS — no local database exists on the EC2) |
