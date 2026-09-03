# AWS Cost Estimate

> Challenge goal: estimate the monthly cost of the architecture using the AWS Pricing Calculator.

- **Estimate link:** _TODO (calculator.aws → Share)_
- **Estimate screenshot:** _TODO (`evidence/cost-estimate.png`)_

## Assumptions

_TODO: region (e.g. us-east-1), hours per month (e.g. 730h), instance types._

## Estimated items

| Service | Configuration | Estimated monthly cost (USD) |
| :--- | :--- | :--- |
| EC2 | _TODO (e.g. t3.micro, Amazon Linux)_ | _TODO_ |
| EBS (EC2 volume) | _TODO (e.g. 8 GB gp3)_ | _TODO_ |
| RDS PostgreSQL | _TODO (e.g. db.t3.micro Single-AZ, 20 GB)_ | _TODO_ |
| Total | | **_TODO_** |

## Free Tier considerations

- EC2 t2.micro/t3.micro: 750h/month on the Free Tier (12 months).
- RDS: 750h of db.t2.micro/db.t3.micro + 20 GB storage on the Free Tier (12 months).
- Watch out for idle Elastic IP charges, data transfer, and snapshots.
- **Challenge reminder:** shut down or remove resources after grading to avoid costs.
