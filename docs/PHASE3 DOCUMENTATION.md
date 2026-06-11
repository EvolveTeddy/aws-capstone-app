# Phase 3: The Application Tiers
## AWS Capstone Project Documentation
**Author:** Theodore Tsekpo  
**Date:** June 2026  
**Region:** us-east-1

---

## Overview
Phase 3 builds the core application infrastructure — the compute 
and database tiers that power a production-grade web application. 
This phase implements a three-layer security and traffic architecture:

1. **Internet → Application Load Balancer** — public entry point
2. **Load Balancer → Web Servers** — auto-scaling EC2 instances
3. **Web Servers → RDS Database** — private backend data store

Each layer has its own dedicated security group enforcing the 
principle of least privilege — only the minimum required 
traffic is permitted between layers.

---

## 1. Security Group — Application Load Balancer

### alb-sg
The first security group created controls traffic into the 
Application Load Balancer. Since the load balancer is the 
public entry point for the application, it accepts traffic 
from the entire internet.

**Configuration Details:**
- Security Group Name: alb-sg
- Security Group ID: sg-0b5e184602abc9650
- Description: Security group for Application Load Balancer
- VPC: main-vpc (vpc-0b96de4dd7f689bb7)
- Inbound Rules: 2 Permission entries
  - HTTP port 80 from 0.0.0.0/0 (all internet traffic)
  - HTTPS port 443 from 0.0.0.0/0 (all internet traffic)
- Outbound Rules: 1 Permission entry

**Security Design:**
The load balancer is the only resource exposed to the public 
internet. Web servers behind it are only accessible from 
the load balancer — not directly from the internet. This 
creates a controlled single entry point into the application.

The security groups list also shows all previously created 
security groups — onprem-sg and migrated-server-sg from 
Phase 2 — confirming the progressive build of the architecture.

<img width="637" height="353" alt="1  First Security group created - application load balancer" src="https://github.com/user-attachments/assets/dc2a0d0d-07fa-4ea1-bf8e-1fc1a9535136" />

---

## 2. Security Group — Web Servers

### webserver-sg
A dedicated security group was created for the EC2 web servers 
launched by the Auto Scaling Group. This security group only 
allows traffic from the load balancer — not from the internet directly.

**Configuration Details:**
- Security Group Name: webserver-sg
- Security Group ID: sg-0e40be27c18b12450
- Description: security group for web servers
- VPC: main-vpc (vpc-0b96de4dd7f689bb7)
- Inbound Rules: 2 Permission entries
  - HTTP — source: alb-sg (load balancer only)
  - SSH — source: IPv4 (for administrative access)
- Outbound Rules: 1 Permission entry

**Security Design — Defence in Depth:**
The HTTP rule source is set to alb-sg rather than 0.0.0.0/0. 
This means only traffic originating from the Application Load 
Balancer can reach the web servers on port 80. A user attempting 
to access a web server's public IP directly would be blocked.

This is a critical security pattern used in production 
environments — the load balancer acts as a shield protecting 
the compute layer from direct internet exposure.


<img width="634" height="346" alt="2  Security group for the web servers" src="https://github.com/user-attachments/assets/9e6952fb-fabe-4101-bf14-c374a2d8587e" />

---
## 3. Application Load Balancer

### main-alb
An Application Load Balancer was deployed to distribute incoming 
traffic across multiple EC2 instances in different availability 
zones. The load balancer provides high availability — if one 
EC2 instance fails, traffic is automatically routed to healthy instances.

**Configuration Details:**
- Name: main-alb
- Type: Application Load Balancer
- Status: Active
- Scheme: Internet-facing
- VPC: main-vpc (vpc-0b96de4dd7f689bb7)
- Availability Zones:
  - us-east-1b (use1-az2) — subnet-0356f1a92aa962915
  - us-east-1a (use1-az1) — subnet-00e2925d98fef6260
- DNS Name: main-alb-1745470901.us-east-1.elb.amazonaws.com
- Date Created: June 4, 2026, 18:45 UTC

**Why Application Load Balancer over Network Load Balancer:**

| Feature | Application LB | Network LB |
|---|---|---|
| Layer | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) |
| Routing | Path-based, host-based | IP and port based |
| Use case | Web applications | High performance TCP |
| Cost | Moderate | Moderate |

Application Load Balancer was chosen because it operates at 
Layer 7 and understands HTTP — allowing future path-based 
routing (e.g. /api routes to one service, /web to another). 
This is essential for microservices architectures.

**Multi-AZ Deployment:**
The load balancer spans two availability zones (us-east-1a and 
us-east-1b). If an entire availability zone experiences an 
outage, the load balancer automatically routes all traffic 
to instances in the remaining zone with zero configuration change.


<img width="638" height="374" alt="3  Main Load balancer created" src="https://github.com/user-attachments/assets/14d313da-66d1-4509-b8b9-bd4ec70b4be1" />

---

## 4. Launch Template

### main-launch-template
A Launch Template defines the exact configuration blueprint 
for every EC2 instance the Auto Scaling Group creates. This 
ensures all instances are identical and consistently configured.

**Configuration Details:**
- Template Name: main-launch-template
- Template ID: lt-020a2ee12bb431d5e
- Default Version: 1
- Latest Version: 1
- Created: 2026-06-04

**Template Configuration Includes:**
- AMI: Amazon Linux 2023
- Instance Type: t2.micro
- Key Pair: my-keypair
- Security Group: webserver-sg
- User Data Script (runs on every new instance):

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Web Server - $(hostname -f)</h1>" > /var/www/html/index.html
```

**Why Launch Templates over Launch Configurations:**
Launch Templates are the modern replacement for Launch 
Configurations. They support versioning — allowing you to 
update the template and roll back if needed — and support 
both On-Demand and Spot instances in the same group.

The User Data script automatically installs and starts Apache 
on every new instance. The $(hostname -f) command inserts 
the instance's internal hostname into the webpage — this is 
how the load balancer test confirms traffic is hitting 
different servers on each refresh.


<img width="637" height="368" alt="4  Created launch template" src="https://github.com/user-attachments/assets/f1ea3c7c-5346-4777-872b-418b93919db3" />

---

## 5. Auto Scaling Group

### main-asg
An Auto Scaling Group was created using the launch template 
to automatically manage the number of EC2 instances based 
on demand. The ASG ensures the application always has 
enough capacity and never wastes money on idle servers.

**ASG Configuration:**
- Name: main-asg
- Launch Template: main-launch-template
- Desired Capacity: 2
- Minimum Capacity: 1
- Maximum Capacity: 4
- Subnets: public-subnet-1a and public-subnet-1b
- Load Balancer: attached to main-tg (target group)
- Health Check: ELB health checks enabled
- Scaling Policy: Target tracking — CPU utilization at 50%

**How Auto Scaling Works:**
- If average CPU across all instances exceeds 50%, 
  AWS automatically adds instances up to the maximum of 4
- If CPU drops below 50%, AWS removes instances down 
  to the minimum of 1
- New instances are configured identically using the 
  launch template
- The load balancer automatically registers new instances 
  and starts sending traffic to them

**EC2 Instances View:**
The instances page confirms 4 total instances running:
- 2 unnamed ASG instances (Running, 2/2 checks passed) 
  in us-east-1b and us-east-1a
- migrated-server (Running, 3/3 checks passed) 
  from Phase 2
- server-onprem (Stopped) from Phase 2

The monitoring panel shows active CPU utilization of 66.9% 
and network traffic confirming the ASG instances are 
actively serving traffic.

<img width="628" height="368" alt="5  auto scaling groups created" src="https://github.com/user-attachments/assets/55c60b26-43e7-48e1-9fab-0a54a3042f6e" />

---

## 6. Load Balancer Test — Verification

### End-to-End Traffic Test
The Application Load Balancer was tested by accessing its 
DNS name in a browser to confirm traffic is correctly 
routed through the load balancer to the web servers.

**Test Result:**
- URL: http://main-alb-1745470901.us-east-1.elb.amazonaws.com
- Response: "Web Server - ip-10-0-1-161.ec2.internal"
- Status: Working correctly

**What This Proves:**
The response shows the internal hostname of the EC2 instance 
that served the request (ip-10-0-1-161.ec2.internal). This 
hostname format (10-0-1-161) corresponds to the private IP 
address 10.0.1.161 — confirming the instance is running 
in public-subnet-1a (10.0.1.0/24) of main-vpc as intended.

**Load Balancing in Action:**
Refreshing the browser multiple times would show different 
hostnames as the load balancer routes each request to 
different instances using the round-robin algorithm. This 
confirms traffic distribution is working correctly.

**Production Note:**
In production, a custom domain name (e.g. app.company.com) 
would be configured in Route 53 pointing to the load 
balancer DNS name. An SSL certificate from AWS Certificate 
Manager would enable HTTPS, removing the "Not secure" 
browser warning.

<img width="632" height="456" alt="6  Testing Load Balancer" src="https://github.com/user-attachments/assets/17886969-6ae0-4929-8f17-150f1d0e1dbf" />

---

## 7. Security Group — RDS Database

### rds-sg
A dedicated security group was created to protect the RDS 
MySQL database. This security group only allows MySQL 
connections (port 3306) from the web servers — not from 
the internet or load balancer.

**Configuration Details:**
- Security Group Name: rds-sg
- Security Group ID: sg-0e42043787e7daa62
- Description: Security group for RDS database
- VPC: main-vpc (vpc-0b96de4dd7f689bb7)
- Inbound Rules: 1 Permission entry
  - Type: MYSQL/Aurora
  - Protocol: TCP
  - Port: 3306
  - Source: webserver-sg
- Outbound Rules: 1 Permission entry

**Critical Security Design:**
The inbound rule source is set to webserver-sg rather than 
an IP address. This means only EC2 instances that belong 
to webserver-sg can connect to the database on port 3306. 

This implements a strict three-tier security model:
- Internet → can only reach ALB
- ALB → can only reach web servers
- Web servers → can only reach the database
- Database → not accessible from anywhere else

Even if an attacker compromised the load balancer, they 
could not directly access the database without also 
compromising a web server instance.


<img width="634" height="370" alt="7  Security group for datatbase created" src="https://github.com/user-attachments/assets/4d047bd2-2e01-4621-b809-a6d071daab71" />

---

## 8. DB Subnet Group

### main-db-subnet-group
A DB Subnet Group was created to specify which subnets 
RDS can use when deploying the database instance. Placing 
the database in private subnets ensures it has no direct 
internet exposure.

**Configuration Details:**
- Name: main-db-subnet-group
- Description: subnet group for main database
- Status: Complete
- VPC: vpc-0b96de4dd7f689bb7 (main-vpc)
- Subnets included:
  - private-subnet-1a (us-east-1a)
  - private-subnet-1b (us-east-1b)

**Why Private Subnets for the Database:**
Private subnets have no route to the internet gateway. 
This means even if someone obtained the database endpoint 
URL and credentials, they could not connect from outside 
the VPC. The database is completely isolated from the 
public internet — only resources inside main-vpc 
with the correct security group can connect.

This is a fundamental security requirement for any 
production database containing customer or business data.


<img width="638" height="369" alt="8  DB subnet group created" src="https://github.com/user-attachments/assets/ea783baa-e0bd-4972-93c9-76811dfba216" />

---

## 9. RDS MySQL Database

### main-database
A MySQL database was created using Amazon RDS providing a 
fully managed relational database with automated backups, 
software patching, and monitoring — all handled by AWS.

**Configuration Details:**
- DB Identifier: main-database
- Status: Available
- Role: Instance
- Engine: MySQL Community
- Upgrade Rollout Order: SECOND
- Subnet Group: main-db-subnet-group
- Security Group: rds-sg
- Public Access: No
- Initial Database Name: appdb
- Storage: 20GB gp2

**Why RDS over Self-Managed MySQL:**

| Feature | Amazon RDS | Self-Managed MySQL on EC2 |
|---|---|---|
| Backups | Automated | Manual |
| Patching | AWS managed | Manual |
| Failover | Automatic (Multi-AZ) | Manual |
| Monitoring | CloudWatch integrated | Manual setup |
| Cost | Higher | Lower |
| Complexity | Low | High |

RDS was chosen to demonstrate managed database services 
and allow focus on architecture rather than database 
administration.

**Production Recommendation:**
In production, Aurora Global Database would replace RDS 
MySQL providing:
- Near-zero RPO with sub-second cross-region replication
- Automatic failover across regions in under 1 minute
- Better read scaling with up to 15 read replicas
- Higher performance with Aurora's distributed storage engine

For this learning project, RDS MySQL achieves the same 
architectural goals at a fraction of the cost.


<img width="634" height="369" alt="9  Database created" src="https://github.com/user-attachments/assets/76af96c5-9a0b-4506-b8dd-9ebbcf37e60d" />

---



## Complete Security Architecture After Phase 3
