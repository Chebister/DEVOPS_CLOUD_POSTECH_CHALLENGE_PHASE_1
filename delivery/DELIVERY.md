# Delivery Report — Phase 1

## Participants

- Yuri Vinicius Santin de Araújo (RM377533)

## Links

- **Repository:** https://github.com/Chebister/DEVOPS_CLOUD_POSTECH_CHALLENGE_PHASE_1
- **Application running (local validation environment, Docker + TLS):** https://toggle-local.cheb.com.br
- **Architecture diagram:** [docs/03-aws-architecture.drawio](../docs/03-aws-architecture.drawio) · exported: [evidence/architecture-diagram.png](../evidence/architecture-diagram.png)
- **Demo video (up to 15 min):** delivered separately with the final submission (external links kept out of this public repository)
- **Cost estimate (AWS Pricing Calculator):** https://calculator.aws/#/estimate?id=500593d865d04d8e20dc3759c3c0b74dd6780dc2
- **Cost estimate screenshot:** [`evidence/cost-estimate.png`](../evidence/)

## Summary of challenges found and decisions made

### Local environment (Docker on Linux VPS)

- **Empty Docker repository for Rocky Linux 9:** the `docker-ce` repo for the `rocky` distro (added by `get.docker.com`) had no binary packages. Decision: pointed the repo to Docker's `centos` channel, which is compatible with RHEL-family distributions (Rocky/Alma/RHEL). Docker CE 29.8.0 + Compose v5.5.1 installed successfully.
- **VPS behind provider NAT:** the VPS has a private IP (`10.15.1.223`) and only the SSH port was forwarded externally (a non-standard external port -> `22`), so the app's port 5000 was not reachable from the internet. Decision: while NAT rules for ports 80/443 were being set up, used an SSH tunnel (`ssh -L 5000:localhost:5000`) to access the app from a workstation.
- **International traffic filtering on ports 80/443:** the Let's Encrypt HTTP-01 and TLS-ALPN-01 validations kept timing out even though ports 80/443 were reachable from Brazil — the hosting provider was filtering international traffic on those ports. Decision: the filter was removed by the provider, after which the ACME challenge succeeded.
- **Staging certificate fallback:** because the first issuance attempts failed (filtering), Caddy fell back to the Let's Encrypt staging CA, whose certificate is not trusted by browsers. Decision: cleared Caddy's data volume and re-obtained the certificate from the production CA, which succeeded.
- **Public exposure with TLS (bonus beyond the challenge's local scope):** chose [Caddy](https://caddyserver.com/) as a reverse proxy for automatic Let's Encrypt issuance/renewal and HTTP-to-HTTPS redirect, proxying to the app on port 5000. Final URL: https://toggle-local.cheb.com.br
- **Endpoint validation:** full CRUD tested with `curl` (health, create, list, get, update, 409 for duplicates, persistence across restarts). Results recorded in [docs/01-monolith-analysis.md](../docs/01-monolith-analysis.md).
- **PostgreSQL 13 → 18 upgrade:** the database image was upgraded to `postgres:18` (18.6). Because it is a major version change, the data volume was recreated (data was disposable) and the app re-created its schema via `flask init-db`. Notably, the 18+ images require the volume mount at `/var/lib/postgresql` instead of the legacy `/var/lib/postgresql/data` — the entrypoint refuses to start otherwise. Full procedure, validation and rollback plan in [docs/05-postgresql-13-to-18-upgrade.md](../docs/05-postgresql-13-to-18-upgrade.md).
- **Base image and dependency update (12-Factor, factor II):** the base image was moved from EOL `python:3.9-slim` to `python:slim` (Python 3.14.7), all Python dependencies were upgraded to versions with modern-Python wheels, and the PostgreSQL client is now pinned to major version 18 via the official PGDG apt repository. Full procedure and validation in [docs/06-python-and-dependency-update.md](../docs/06-python-and-dependency-update.md).
- **Config moved to environment files (12-Factor, factor III):** hardcoded credentials in `docker-compose.yaml` were replaced with variable interpolation resolved from a gitignored `.env` file on the deployment machine (with a committed `.env.example` template). Details in [docs/07-config-via-env-files.md](../docs/07-config-via-env-files.md).

### AWS architecture and deployment

Provisioned manually via the AWS console (us-east-1), validated on 2026-10-12:

- **VPC** `prod-vpc` (10.25.0.0/16) with an attached Internet Gateway; a public subnet (10.25.0.0/24) for the application tier and isolated DB subnets in two AZs (10.25.10.0/24, 10.25.11.0/24) with a local-only route table — the database tier has no internet route. Full detail in [docs/03-aws-architecture.md](../docs/03-aws-architecture.md).
- **EC2** `srv-togglemaster-prod` (t3.micro, Ubuntu 24.04, IMDSv2 required): application container (Flask/Gunicorn :5000) behind a Caddy container (host network) terminating TLS on 80/443 with automatic Let's Encrypt issuance; only the `app` service runs (`docker compose up -d --no-deps app`) — no local PostgreSQL on AWS. Elastic IP attached; DNS on Cloudflare.
- **RDS** PostgreSQL 18 (`database-1`, db.t4g.micro, Single-AZ) in the isolated subnets (DB subnet group across two AZs), public access disabled, reachable only on 5432 from the application Security Group.
- **Security Groups (least privilege):** `prod-app-sg` — inbound 80/443 from 0.0.0.0/0 and 22 only from the administrator IP /32 (key-only SSH; the lab account has no IAM permissions for SSM); `prod-db-sg` — inbound 5432 only from `prod-app-sg` (SG-to-SG reference, not CIDR). Port 5000 is never exposed — Caddy proxies it internally.
- **Credentials:** the RDS master password is generated and managed by RDS in AWS Secrets Manager (managed rotation) and injected into the application via a server-side `docker-compose.override.yml` (chmod 600, root-owned, outside Git and covered by `.gitignore`). No secret is ever committed.

Challenges and decisions (AWS phase):

- **NAT gateway:** the first creation failed — a public NAT gateway requires the Internet Gateway to be attached to the VPC *before* creation. After analysis, the final architecture dropped the NAT gateway entirely: the web tier needs *inbound* 80/443 (Caddy/ACME), so the EC2 lives in a public subnet; RDS patching is AWS-managed, with no customer NAT required.
- **ALB:** evaluated and discarded — a single instance in the MVP and Caddy already terminates TLS (~US$ 16–20/month saved).
- **RDS in the wrong subnet:** the first instance was born in the default subnet group (which includes the public subnet) and RDS refuses in-place subnet-group moves within the same VPC (misleading error message) — the instance was recreated directly in the isolated group.
- **`$` in the generated password swallowed by Compose interpolation** (interpreted as an undefined variable) — fixed with the `$$` escape in the environment file.
- **Self-recovery:** after a VM reboot the application did not return (no restart policy) — `unless-stopped` applied; today the whole VM recovers by itself (systemd → Docker → containers → entrypoint waits for RDS).
- **VM "unreachable" while a VPN was active:** the SSH rule allows only the residential IP; a VPN changes the egress IP and the port appears blocked.

Validation (via https://togglemaster.cheb.com.br): health 200 · create 201 · list/get/update 200 · duplicate 409 · application restart keeps data (persisted in RDS). Cost estimate (~US$ 23.57/month) in [docs/04-cost-estimate.md](../docs/04-cost-estimate.md).

## Deliverables checklist

- [x] Application running locally (Docker) — https://toggle-local.cheb.com.br
- [x] Architecture diagram (VPC, subnets, EC2, RDS, Security Groups) — [docs/03-aws-architecture.drawio](../docs/03-aws-architecture.drawio) + [evidence/architecture-diagram.png](../evidence/architecture-diagram.png)
- [x] 12-Factor App discussion — [docs/02-12-factor-app.md](../docs/02-12-factor-app.md)
- [x] Application running on EC2 connected to RDS — [docs/03-aws-architecture.md](../docs/03-aws-architecture.md) · tests in [evidence/](../evidence/)
- [x] Security Group and credentials configuration — screenshots in [evidence/](../evidence/)
- [x] AWS cost estimate — [link](https://calculator.aws/#/estimate?id=500593d865d04d8e20dc3759c3c0b74dd6780dc2) + [screenshot](../evidence/cost-estimate.png)
- [x] Demo video (up to 15 min) — delivered separately with the final submission
