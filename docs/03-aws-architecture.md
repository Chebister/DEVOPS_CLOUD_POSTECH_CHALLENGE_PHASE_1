# Proposed AWS Architecture

> Challenge goal: draw a simple architecture diagram to host the application on AWS.

## Diagram

- **Diagram source (draw.io, editable):** [`03-aws-architecture.drawio`](03-aws-architecture.drawio)
- **Diagram link (published, Miro / diagrams.net):** _TODO_
- Exported image: _TODO (`evidence/architecture-diagram.png`)_

## Components

| Component | Detail | Rationale |
| :--- | :--- | :--- |
| **VPC** | _TODO: CIDR (e.g. 10.0.0.0/16)_ | Logical network isolation |
| **Public subnet** | _TODO: CIDR + AZ_ | Hosts the EC2 (needs external access) |
| **Private subnet** | _TODO: CIDR + AZ_ | Hosts the RDS, not exposed to the internet |
| **Internet Gateway** | 1 per VPC | Allows inbound/outbound traffic for the public subnet |
| **EC2 instance** | _TODO: type (e.g. t2.micro/t3.micro), Amazon Linux 2023_ | Runs the Flask/Gunicorn application on port 5000 |
| **RDS PostgreSQL** | _TODO: type (e.g. db.t3.micro), Multi-AZ?_ | Managed database |
| **Security Group (EC2)** | Inbound: 5000 (0.0.0.0/0), 22 (specific IP only _TODO_) | Exposes the app and allows controlled administrative SSH |
| **Security Group (RDS)** | Inbound: 5432 only from the EC2 SG | Database reachable only by the application |

## Traffic flow

```text
Internet --> IGW --> Public subnet --> EC2 :5000 (Flask/Gunicorn app)
                                        |
                                        +--> EC2-SG -> RDS-SG :5432 --> RDS PostgreSQL (private subnet)
```

_TODO: detail decisions (instance size, PostgreSQL version, single AZ vs Multi-AZ, EIP vs dynamic IP)._

## Security

_TODO: describe_ — least privilege in Security Groups, SSH restricted to a specific IP, credentials via environment variables, no secrets in code.
