# Architecture Decision Record (ADR)
## AWS Capstone Project
**Date:** June 2026  
**Author:** Theodore Teddy Tsekpo

---

## ADR-001: VPC Design and Multi-Region Strategy

### Decision
Two VPCs were created across two AWS regions:
- main-vpc (10.0.0.0/16) in us-east-1 as the primary region
- dr-vpc (10.1.0.0/16) in us-west-2 as the disaster recovery region

### Reasoning
- Separate regions provide geographic redundancy
- Different CIDR blocks prevent IP conflicts during VPC peering
- Two availability zones per VPC ensure high availability

### Alternatives Considered
- Single region with multiple AZs — rejected because a full region 
  outage would cause complete downtime
- Three regions — rejected due to cost and complexity

---

## ADR-002: VPC Peering vs Transit Gateway

### Decision
VPC Peering was used to connect main-vpc, dr-vpc, and onprem-vpc.

### Reasoning
- VPC Peering is free for same-region connections
- Sufficient for connecting three VPCs in this project
- Simple to configure and troubleshoot

### Alternatives Considered
- Transit Gateway — rejected because it costs $0.05/hour per 
  attachment and is designed for connecting 10+ VPCs
- Site-to-Site VPN — costs ~$36/month, not justified for 
  a learning project

### Production Recommendation
In a real production environment with many accounts and VPCs, 
Transit Gateway would be the preferred solution for its 
hub-and-spoke architecture and centralized routing.

---

## ADR-003: RDS MySQL vs Aurora Global Database

### Decision
RDS MySQL with Dev/Test template and Single-AZ deployment was used 
instead of Aurora Global Database.

### Reasoning
- RDS MySQL db.t3.micro is within free tier limits
- Teaches the same relational database concepts
- Aurora Global Database costs significantly more (~$0.10/hour 
  per instance minimum)

### Aurora Global Advantages (Production)
- RPO near zero with cross-region replication under 1 second
- RTO under 1 minute for regional failover
- Better performance with Aurora's distributed storage engine
- Automatic failover across regions

### RDS MySQL Limitations
- Manual promotion of read replica during failover
- Higher RPO (up to 5 minutes depending on replication lag)
- No automatic cross-region failover

### Production Recommendation
Aurora Global Database with a read replica in us-west-2 would be 
used in production for minimal RPO/RTO during disaster recovery.

---

## ADR-004: App Runner Deprecation — ECS Fargate Decision

### Decision
Amazon ECS Fargate was used to run the containerized payment 
microservice instead of AWS App Runner.

### Reasoning
- AWS App Runner stopped accepting new customers on April 30, 2026
- ECS Fargate is AWS's recommended replacement
- Fargate provides serverless container execution without 
  managing EC2 instances
- More control over networking and security configuration

### Alternatives Considered
- EKS (Kubernetes) — rejected because the control plane costs 
  $73/month making it impractical for a learning project
- EC2 with Docker — rejected because it requires manual server 
  management defeating the purpose of modernization

### Production Recommendation
Amazon EKS would be used in production for:
- Advanced orchestration with Kubernetes
- Better support for microservices at scale
- More ecosystem tooling and community support

---

## ADR-005: CI/CD Pipeline Design

### Decision
AWS CodePipeline with GitHub source was used for the CI/CD pipeline.

### Reasoning
- CodePipeline first pipeline is free
- Native integration with AWS services
- Automatic deployment on every GitHub push
- No additional infrastructure to manage

### Alternatives Considered
- Jenkins — requires managing a dedicated EC2 instance adding cost
- GitHub Actions — excellent alternative but less integrated 
  with AWS services natively

### Production Recommendation
For a larger team, GitHub Actions with AWS OIDC authentication 
would provide more flexibility and better developer experience.

---

## ADR-006: Security Design Decisions

### Decision
Three layers of security were implemented:
1. Service Control Policies at the organization level
2. Security Groups at the resource level
3. GuardDuty and AWS Config for monitoring and compliance

### Reasoning
- Defense in depth — multiple layers prevent single points of failure
- SCPs enforce company-wide rules nobody can bypass
- Security Groups provide granular resource-level control
- GuardDuty provides intelligent threat detection without 
  manual rule configuration

### Key Security Rules Implemented
- No unencrypted RDS databases allowed (SCP)
- No public S3 buckets allowed (SCP + Config rule)
- Web servers only accessible through load balancer
- Database only accessible from web servers
- SSM Session Manager used instead of SSH for server access
