# Cost Optimization Report
## AWS Capstone Project
**Date:** June 2026  
**Author:** Theodore Teddy Tsekpo

---

## Current Monthly Cost Estimate (Learning Environment)

| Service | Configuration | Estimated Monthly Cost |
|---|---|---|
| EC2 (ASG) | 2x t2.micro | ~$17.00 |
| RDS MySQL | db.t3.micro Single-AZ | ~$13.00 |
| NAT Gateway | 1x us-east-1 | ~$32.00 |
| Application Load Balancer | 1x main-alb | ~$16.00 |
| ECS Fargate | 0.25 vCPU / 0.5GB | ~$5.00 |
| ECR | Storage + transfers | ~$1.00 |
| CodePipeline | 1 pipeline | Free |
| GuardDuty | 30-day trial | Free |
| AWS Config | 2 rules | ~$2.00 |
| VPC | Peering + data transfer | ~$2.00 |
| S3 | Artifacts + Config bucket | ~$1.00 |
| **Total** | | **~$89/month** |

---

## Production Cost Estimate (Full Scale)

| Service | Production Configuration | Estimated Monthly Cost |
|---|---|---|
| EC2 (ASG) | 4x t3.medium | ~$120.00 |
| Aurora Global | db.r6g.large Multi-AZ | ~$400.00 |
| NAT Gateway | 2x (one per region) | ~$64.00 |
| Application Load Balancer | 2x (one per region) | ~$32.00 |
| EKS | Control plane + nodes | ~$200.00 |
| ECR | Storage + transfers | ~$10.00 |
| CodePipeline | 3 pipelines | ~$3.00 |
| GuardDuty | Full monitoring | ~$50.00 |
| AWS Config | 10+ rules | ~$20.00 |
| CloudTrail | Full audit logging | ~$10.00 |
| **Total** | | **~$909/month** |

---

## Cost Optimization Strategies

### Strategy 1 — Reserved Instances
- Purchase 1-year Reserved Instances for EC2 and RDS
- Savings: up to 40% compared to On-Demand pricing
- Best for: stable workloads running 24/7
- Example: t3.medium On-Demand = $30/month, 
  Reserved = $18/month

### Strategy 2 — Graviton Instances
- Switch from x86 (t3) to ARM-based Graviton (t4g) instances
- Savings: up to 20% better price/performance
- Works with: EC2, RDS, ECS Fargate
- Example: db.t4g.micro costs less than db.t3.micro 
  with better performance

### Strategy 3 — Spot Instances for Non-Critical Workloads
- Use Spot Instances for development and testing environments
- Savings: up to 90% compared to On-Demand
- Risk: instances can be interrupted with 2-minute notice
- Best for: batch jobs, dev/test, fault-tolerant workloads

### Strategy 4 — NAT Gateway Optimization
- NAT Gateway is the biggest cost driver at ~$32/month
- Alternative: NAT Instance using t3.nano (~$3.50/month)
- Savings: ~$28/month per NAT Gateway
- Trade-off: NAT Instance requires more management

### Strategy 5 — S3 Lifecycle Policies
- Move objects older than 30 days to S3 Standard-IA
- Move objects older than 90 days to S3 Glacier
- Savings: up to 90% on storage costs for old data
- Best for: log files, backups, archives

### Strategy 6 — Auto Scaling Optimization
- Set minimum capacity to 1 during off-peak hours
- Use scheduled scaling for predictable traffic patterns
- Example: scale down to 1 instance at night, 
  scale up to 4 during business hours

### Strategy 7 — Aurora Serverless v2
- Use Aurora Serverless v2 instead of provisioned Aurora
- Scales automatically from 0.5 to 128 ACUs
- Cost: only pay for what you use
- Best for: variable or unpredictable workloads

---

## AWS Well-Architected Cost Optimization Principles Applied

1. **Right-sizing** — Used t2.micro and t3.micro instead of 
   larger instances
2. **Elasticity** — Auto Scaling Group adds/removes instances 
   based on demand
3. **Pricing model selection** — Free tier and credits used 
   for learning environment
4. **Resource scheduling** — Instances stopped between 
   working sessions
5. **Tagging** — All resources tagged for cost tracking

---

## Tools Used for Cost Analysis
- AWS Pricing Calculator: calculator.aws
- AWS Cost Explorer: Billing → Cost Explorer
- AWS Budgets: Alert set at $10/month
