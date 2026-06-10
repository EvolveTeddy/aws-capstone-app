# Phase 2: Hybrid Migration & Connectivity
## AWS Capstone Project Documentation
**Author:** Theodore Tsekpo  
**Date:** June 2026  
**Region:** us-east-1

---

## Overview
Phase 2 simulates a real-world hybrid cloud migration scenario where 
an organisation moves workloads from an on-premise data centre into 
AWS. A dedicated VPC simulates the on-premise environment, VPC Peering 
simulates the Site-to-Site VPN tunnel, and EC2 instances represent 
the physical servers being migrated.

This phase demonstrates the architectural thinking behind hybrid 
connectivity and server migration — two of the most common 
real-world cloud engineering tasks.

---

## 1. On-Premise VPC Creation

### vpc-onprem
A dedicated VPC was created to simulate an on-premise data centre 
network. This VPC uses the 192.168.0.0/16 CIDR block — a private 
RFC 1918 address range commonly used in physical office networks — 
to accurately simulate what a real on-premise environment would look like.

**Configuration Details:**
- VPC ID: vpc-05daea563a818faef
- CIDR Block: 192.168.0.0/16
- State: Available
- Region: us-east-1
- DNS Resolution: Enabled
- Tenancy: Default

**Why 192.168.0.0/16:**
This address range was deliberately chosen to be different from 
main-vpc (10.0.0.0/16) and bu-vpc (10.1.0.0/16) to prevent 
IP address conflicts when peering all three networks together. 
In a real hybrid environment, the on-premise network would have 
its own distinct address space.

<img width="633" height="350" alt="1  Onprem vpc created" src="https://github.com/user-attachments/assets/bad0d0c4-f09a-4631-90d0-1fea191867f9" />


## 2. On-Premise Subnet Configuration

### subnet-onprem
A single subnet was created within vpc-onprem to host the 
simulated on-premise server.

**Configuration Details:**
- Subnet Name: subnet-onprem
- Subnet ID: subnet-00b5154ea9598b202
- CIDR Block: 192.168.1.0/24
- Availability Zone: us-east-1a
- Available IPv4 Addresses: 251
- State: Available

The subnet list also shows the four main-vpc subnets 
(public-subnet-1a/1b and private-subnet-1a/1b) confirming 
all networks are correctly configured in us-east-1.

**Subnet Design Decision:**
Only one subnet was created for the on-premise simulation since 
a real on-premise data centre typically exists in a single 
physical location unlike cloud environments that span 
multiple availability zones.


<img width="635" height="350" alt="2  onprem subnet created" src="https://github.com/user-attachments/assets/7a7c6bfe-10c8-4e3a-a6db-ae7574196ec2" />


## 3. On-Premise Route Table

### route-table-onprem
A route table was created for the on-premise VPC to direct 
traffic correctly between the local network and the internet.

**Routes Configured:**

| Destination | Target | Status | Purpose |
|---|---|---|---|
| 192.168.0.0/16 | local | Active | Internal VPC traffic |
| 0.0.0.0/0 | igw-0a6be548466cd7c... | Active | Internet access |

The route table is explicitly associated with subnet-onprem 
confirming correct subnet-to-route-table binding.

**Key Observation:**
The internet route (0.0.0.0/0) was added to allow the 
on-premise server to receive inbound connections for testing 
and to allow SSM Session Manager access for server 
management without SSH.


<img width="634" height="350" alt="3  route-table-onprem" src="https://github.com/user-attachments/assets/66dcb8b5-afbf-4c53-a671-40bb77099067" />


## 4. On-Premise Server (EC2 Instance)

### server-onprem
An EC2 t2.micro instance running Amazon Linux 2023 was launched 
inside vpc-onprem to simulate a physical server running in an 
on-premise data centre.

**Instance Configuration:**
- Name: server-onprem
- Instance ID: i-08113ad3b6617abdb
- Instance Type: t2.micro
- State: Running
- Status Checks: 2/2 checks passed
- Availability Zone: us-east-1a

**Access Method — SSM Session Manager:**
EC2 Instance Connect and standard SSH were unreliable for this 
instance due to network configuration. AWS Systems Manager 
Session Manager was used instead by:
1. Creating an IAM role (ec2-ssm-role) with 
   AmazonSSMManagedInstanceCore policy
2. Attaching the role to the EC2 instance
3. Using CloudShell to initiate SSM sessions

This is actually the production-recommended approach — SSM 
Session Manager eliminates the need to open port 22 and 
provides an auditable connection method.

<img width="632" height="368" alt="4  EC2 created" src="https://github.com/user-attachments/assets/b2d539eb-f632-4d1e-a60e-f2347fadf593" />

## 5. VPC Peering — Simulating Site-to-Site VPN

### main-to-onprem-peering
A VPC Peering connection was established between main-vpc and 
vpc-onprem to simulate the Site-to-Site VPN tunnel that would 
connect a real on-premise data centre to AWS in production.

**Peering Connection Details:**
- Connection ID: pcx-086fdb052ae7411d7
- Name: main-to-onprem-peering
- Status: Active
- Requester VPC: main-vpc (10.0.0.0/16) — us-east-1
- Accepter VPC: vpc-onprem (192.168.0.0/16) — us-east-1

**Why VPC Peering Instead of Site-to-Site VPN:**

| Feature | VPC Peering | Site-to-Site VPN |
|---|---|---|
| Cost | Free (same region) | ~$36/month |
| Setup complexity | Simple | Complex |
| Encryption | AWS backbone | IPSec encrypted tunnel |
| Use case | AWS-to-AWS | On-premise-to-AWS |
| Production use | Multi-VPC architecture | Hybrid connectivity |

VPC Peering was used as a cost-effective simulation. In a 
real production environment, AWS Site-to-Site VPN or AWS 
Direct Connect would be used to connect a physical data 
centre to AWS. Direct Connect provides dedicated bandwidth 
and lower latency compared to VPN.

Route tables in both main-vpc and vpc-onprem were updated 
after peering to enable bidirectional traffic flow:
- main-vpc routes: added 192.168.0.0/16 → peering connection
- vpc-onprem routes: added 10.0.0.0/16 → peering connection


<img width="631" height="350" alt="6  peering created" src="https://github.com/user-attachments/assets/9ee2067f-ffeb-4fed-9d01-c0c90b1f58bc" />


## 6. Migration Target Server

### migrated-server
A second EC2 instance was launched inside main-vpc to serve 
as the migration target — representing the new AWS environment 
the workload is being migrated to.

**Instance Configuration:**
- Name: migrated-server
- Instance ID: i-0283c5de0592769ef
- Instance Type: t3.micro
- State: Running
- Availability Zone: us-east-1a

Both instances are visible in the EC2 console showing the 
complete migration architecture:
- server-onprem — the source (old environment)
- migrated-server — the target (new AWS environment)

**IAM Role Applied:**
The same ec2-ssm-role was attached to migrated-server to 
enable SSM Session Manager access for server configuration 
and application installation.

<img width="633" height="370" alt="7  Migrated Server EC2 created" src="https://github.com/user-attachments/assets/48ad3e1b-d88b-4b67-bd60-3d37860ed95f" />


## 7. Web Server Installation — On-Premise Server

### Apache HTTP Server Installation
The Apache web server (httpd) was installed on server-onprem 
via SSM Session Manager using AWS CloudShell. This simulates 
the application running on the old on-premise server before migration.

**Installation Process:**
The SSM session was initiated from CloudShell using:
```bash
aws ssm start-session --target i-08113ad3b6617abdb --region us-east-1
```

Commands executed on the instance:
```bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "<h1>This is the On-Premise Server</h1>" | sudo tee /var/www/html/index.html
```

The terminal shows Amazon Linux 2023 successfully connected 
with IP 192.168.1.109 confirming the instance is running 
in the correct on-premise subnet (192.168.0.0/16 range).

<img width="632" height="349" alt="8  fake on-premise server 1" src="https://github.com/user-attachments/assets/faa81add-0073-4687-a456-6dd976cbf3c0" />


## 8. Web Server Installation — Completion

### Installation Successfully Completed
All 13 packages were downloaded and installed successfully 
including httpd and all dependencies.

**Packages Installed:**
- httpd 2.4.67
- apr, apr-util, apr-util-lmdb
- httpd-core, httpd-filesystem, httpd-tools
- mod_http2, mod_lua
- libbrotli, mailcap, generic-logos-httpd

Total installation size: 7.0 MB downloaded at 10 MB/s.

The installation output confirms:
- Transaction check succeeded
- Transaction test succeeded  
- All 13/13 packages installed and verified
- httpd service started successfully
- httpd enabled for automatic startup on reboot
- HTML page created at /var/www/html/index.html

<img width="638" height="366" alt="9  fake on-premise server 2" src="https://github.com/user-attachments/assets/e5295a58-bc6a-4f3d-94ad-7066f802c71d" />

<img width="637" height="362" alt="10  fake on-premise server 3" src="https://github.com/user-attachments/assets/2bc3cebd-9d7a-4f45-b228-b174f76db5e6" />

## 9. Phase 2 Verification — Migration Working

### End-to-End Test
The migrated server was accessed via its public IP address 
in a browser to confirm the web application is running 
successfully inside the AWS main-vpc environment.

**Test Result:**
- URL: http://44.193.3.80
- Response: "It works!"
- Status: HTTP 200 OK
- Connection: Not secure (HTTP only — HTTPS not configured 
  for this learning environment)

**What This Proves:**
The Apache web server installed on migrated-server is 
running and accessible from the public internet confirming:
1. The EC2 instance launched correctly in main-vpc
2. The security group allows HTTP traffic on port 80
3. The public route table correctly routes internet traffic
4. The Internet Gateway is functioning correctly
5. The migration simulation was successful

**Production Note:**
In a real migration, HTTPS would be configured using AWS 
Certificate Manager (ACM) with an SSL certificate attached 
to the Application Load Balancer. The "Not secure" warning 
would not appear in production.

<img width="638" height="430" alt="11  phase 2 completed- fake server works" src="https://github.com/user-attachments/assets/0ba81198-dae8-4f90-b238-79e02808c054" />
