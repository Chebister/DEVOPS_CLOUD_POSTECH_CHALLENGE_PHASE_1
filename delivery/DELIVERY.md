# Delivery Report — Phase 1

## Participants

- Yuri _TODO: last name / student ID_

## Links

- **Repository:** https://github.com/Chebister/DEVOPS_CLOUD_POSTECH_CHALLENGE_PHASE_1
- **Application running (local validation environment, Docker + TLS):** https://toggle-local.cheb.com.br
- **Architecture diagram:** _TODO (Miro / diagrams.net)_
- **Demo video (up to 15 min):** _TODO_
- **Cost estimate (AWS Pricing Calculator):** _TODO_
- **Cost estimate screenshot:** [`evidence/cost-estimate.png`](../evidence/)

## Summary of challenges found and decisions made

### Local environment (Docker on Linux VPS)

- **Empty Docker repository for Rocky Linux 9:** the `docker-ce` repo for the `rocky` distro (added by `get.docker.com`) had no binary packages. Decision: pointed the repo to Docker's `centos` channel, which is compatible with RHEL-family distributions (Rocky/Alma/RHEL). Docker CE 29.8.0 + Compose v5.5.1 installed successfully.
- **VPS behind provider NAT:** the VPS has a private IP (`10.15.1.223`) and only the SSH port was forwarded externally (`22223` -> `22`), so the app's port 5000 was not reachable from the internet. Decision: while NAT rules for ports 80/443 were being set up, used an SSH tunnel (`ssh -L 5000:localhost:5000`) to access the app from a workstation.
- **International traffic filtering on ports 80/443:** the Let's Encrypt HTTP-01 and TLS-ALPN-01 validations kept timing out even though ports 80/443 were reachable from Brazil — the hosting provider was filtering international traffic on those ports. Decision: the filter was removed by the provider, after which the ACME challenge succeeded.
- **Staging certificate fallback:** because the first issuance attempts failed (filtering), Caddy fell back to the Let's Encrypt staging CA, whose certificate is not trusted by browsers. Decision: cleared Caddy's data volume and re-obtained the certificate from the production CA, which succeeded.
- **Public exposure with TLS (bonus beyond the challenge's local scope):** chose [Caddy](https://caddyserver.com/) as a reverse proxy for automatic Let's Encrypt issuance/renewal and HTTP-to-HTTPS redirect, proxying to the app on port 5000. Final URL: https://toggle-local.cheb.com.br
- **Endpoint validation:** full CRUD tested with `curl` (health, create, list, get, update, 409 for duplicates, persistence across restarts). Results recorded in [docs/01-monolith-analysis.md](../docs/01-monolith-analysis.md).
- **PostgreSQL 13 → 18 upgrade:** the database image was upgraded to `postgres:18` (18.6). Because it is a major version change, the data volume was recreated (data was disposable) and the app re-created its schema via `flask init-db`. Notably, the 18+ images require the volume mount at `/var/lib/postgresql` instead of the legacy `/var/lib/postgresql/data` — the entrypoint refuses to start otherwise. Full procedure, validation and rollback plan in [docs/05-postgresql-13-to-18-upgrade.md](../docs/05-postgresql-13-to-18-upgrade.md).
- **Base image and dependency update (12-Factor, factor II):** the base image was moved from EOL `python:3.9-slim` to `python:slim` (Python 3.14.7), all Python dependencies were upgraded to versions with modern-Python wheels, and the PostgreSQL client is now pinned to major version 18 via the official PGDG apt repository. Full procedure and validation in [docs/06-python-and-dependency-update.md](../docs/06-python-and-dependency-update.md).

### AWS architecture and deployment

_TODO: fill in after the AWS phase (instance types, PostgreSQL, VPC/subnet structure, Security Groups with least privilege, RDS credentials via environment variables, manual provisioning and EC2 deployment notes)._

## Deliverables checklist

- [x] Application running locally (Docker) — https://toggle-local.cheb.com.br (video pending)
- [ ] Architecture diagram (VPC, subnets, EC2, RDS, Security Groups) — link
- [ ] 12-Factor App discussion — [docs/02-12-factor-app.md](../docs/02-12-factor-app.md)
- [ ] Application running on EC2 connected to RDS — video
- [ ] Security Group and credentials configuration — video + screenshots in `evidence/`
- [ ] AWS cost estimate — link + screenshot
- [ ] Demo video (up to 15 min) — link
