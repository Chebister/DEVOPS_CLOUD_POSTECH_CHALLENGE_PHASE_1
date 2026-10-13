# AWS Cost Estimate

> Challenge goal: estimate the monthly cost of the architecture using the AWS Pricing Calculator.

- **Estimate link:** https://calculator.aws/#/estimate?id=500593d865d04d8e20dc3759c3c0b74dd6780dc2
- **Estimate screenshot:** [`evidence/cost-estimate.png`](../evidence/cost-estimate.png)

## Assumptions

- Region: us-east-1 (N. Virginia)
- 730 hours/month (24/7), on-demand pricing (no Reserved/Savings Plans)
- Architecture as built ([docs/03](03-aws-architecture.md)): 1 EC2 + 1 Elastic IP + 1 RDS PostgreSQL Single-AZ + 1 Secrets Manager secret
- Data transfer not selected (demo/test workload, below the free tier)

## Estimate (AWS Pricing Calculator)

| Service | Configuration | Monthly cost (USD) |
| :--- | :--- | :--- |
| Amazon EC2 (on-demand) | t3.micro · Linux · 730 h · incl. 20 GB EBS | 9.19 |
| Elastic IP | attached to the EC2 ENI 24/7 | 0.00 |
| Amazon RDS for PostgreSQL (on-demand) | db.t4g.micro · Single-AZ · 730 h · incl. 20 GB gp2 storage | 13.98 |
| AWS Secrets Manager | 1 secret · 30 days | 0.40 |
| **Monthly total** | | **23.57** |
| Total 12 months | | 282.84 |

Upfront cost: 0.00 USD.

## Free Tier considerations

- EC2 t3.micro: 750 h/month free during the first 12 months (Free Tier accounts).
- RDS: 750 h/month of db.t2/db.t3.micro + 20 GB storage free during the first 12 months.
- The account used for this challenge (AWS Academy learner lab) is billed against the lab budget instead of a personal payment method.
- An **idle (detached) Elastic IP is charged** (~US$ 0.005/h) — keep it attached or release it.
- **Challenge reminder:** shut down or delete resources after grading to avoid charges.
