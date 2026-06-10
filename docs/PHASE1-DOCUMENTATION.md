# Phase 1: Foundation & Governance
## AWS Capstone Project Documentation
**Author:** Theodore Tsekpo  
**Date:** June 2026  
**Region:** us-east-1 (Primary) | us-west-2 (DR)

---

## Overview
Phase 1 establishes the core networking foundation and governance 
framework for the entire project. This includes building production-grade 
VPCs, configuring subnets across multiple availability zones, setting up 
internet connectivity, and implementing centralized identity management 
through IAM Identity Center.

---

## 1. VPC Architecture

### Main VPC (main-vpc)
The primary VPC was created in us-east-1 with a CIDR block of 
10.0.0.0/16 providing 65,536 available IP addresses. This VPC serves 
as the production environment hosting all application workloads.

The resource map shows the complete network topology:
- 4 subnets across 2 availability zones (us-east-1a and us-east-1b)

- 3 route tables controlling traffic flow
- 1 internet gateway providing internet connectivity

**Key Design Decisions:**
- /16 CIDR chosen to provide sufficient IP space for future growth
- Dual availability zone deployment ensures high availability
- Separate public and private subnets enforce security boundaries

<img width="1908" height="814" alt="vpc main" src="https://github.com/user-attachments/assets/b7a4ff6c-d723-426c-baad-59ba19bf4824" />

## 2. Subnet Configuration

### Public and Private Subnets
Four subnets were created within main-vpc across two availability zones:

| Subnet Name | Availability Zone | CIDR Block | Type |
|---|---|---|---|
| public-subnet-1a | us-east-1a | 10.0.1.0/24 | Public |
| public-subnet-1b | us-east-1b | 10.0.2.0/24 | Public |
| private-subnet-1a | us-east-1a | 10.0.3.0/24 | Private |
| private-subnet-1b | us-east-1b | 10.0.4.0/24 | Private |

**Public Subnets** host internet-facing resources such as the 
Application Load Balancer and bastion hosts. Auto-assign public 
IPv4 is enabled on these subnets.

**Private Subnets** host backend resources such as EC2 web servers 
and RDS databases. These subnets have no direct internet access, 
providing an additional security layer.

<img width="635" height="352" alt="2  Public and Private Subnets Created" src="https://github.com/user-attachments/assets/b82ef8db-e7e0-4566-bc5a-54b7e80bfe39" />

---

## 3. Internet Gateway

### main-igw
An Internet Gateway was created and attached to main-vpc to enable 
internet connectivity for resources in public subnets.

The Internet Gateway serves as the entry and exit point for all 
internet-bound traffic from the VPC. Without this component, 
no resources in the VPC can communicate with the public internet.

**State:** Attached  
**Attached VPC:** main-vpc (vpc-0722e5adf23c45a8f)

<img width="633" height="347" alt="3  Internet Gateway created" src="https://github.com/user-attachments/assets/42cb582f-0251-45a5-8967-802700a18b37" />


## 4. Route Table Configuration

### Public Route Table
A dedicated route table was created for public subnets with two routes:

| Destination | Target | Purpose |
|---|---|---|
| 10.0.0.0/16 | local | Internal VPC traffic |
| 0.0.0.0/0 | main-igw | Internet-bound traffic |

The route 0.0.0.0/0 pointing to the Internet Gateway is what makes 
a subnet "public" — it allows resources to send traffic to and 
receive traffic from the internet.

Both public-subnet-1a and public-subnet-1b are associated with 
this route table ensuring consistent routing across availability zones.

<img width="634" height="347" alt="4  Route Table" src="https://github.com/user-attachments/assets/f4c06d0e-abee-4f14-aaaf-23465158d3a4" />


## 5. NAT Gateway

### main-nat-gw
A NAT (Network Address Translation) Gateway was deployed in 
public-subnet-1a to enable private subnet resources to access 
the internet for updates and package installations without 
exposing them to inbound internet traffic.

**Configuration:**
- Connectivity type: Public
- Subnet: public-subnet-1a (public subnet)
- Primary public IPv4: 16.192.59.99
- State: Available

**Why NAT Gateway over NAT Instance:**
NAT Gateway is AWS-managed, highly available within an AZ, 
and requires no maintenance. NAT Instance requires manual 
management but costs less — NAT Gateway was chosen for 
reliability in this architecture.

**Cost Note:** NAT Gateway costs approximately $0.045/hour 
plus data processing charges. It is deleted between working 
sessions to minimize costs.


<img width="635" height="335" alt="5  Nat gateway created" src="https://github.com/user-attachments/assets/22da7d3e-a54e-47d4-8c97-e588a8dc8f55" />


## 6. Backup/DR VPC Configuration

### Backup VPC (bu-vpc)
A disaster recovery VPC was created in us-west-2 with a separate 
CIDR block of 10.1.0.0/16 to avoid IP conflicts with main-vpc 
during VPC peering.

Four subnets were created following the same pattern as main-vpc:

| Subnet Name | Availability Zone | CIDR Block | Type |
|---|---|---|---|
| bu-public-subnet-1a | us-west-2a | 10.1.1.0/24 | Public |
| bu-public-subnet-1b | us-west-2b | 10.1.2.0/24 | Public |
| bu-private-subnet-1a | us-west-2a | 10.1.3.0/24 | Private |
| bu-private-subnet-1b | us-west-2b | 10.1.4.0/24 | Private |

[INSERT SCREENSHOT: 6__Backup_private_subnet_created.png]
*Screenshot: All 4 DR subnets successfully created in bu-vpc 
showing Available status across both availability zones*

<img width="623" height="355" alt="6  Backup private subnet created" src="https://github.com/user-attachments/assets/5782583d-03fd-4dd2-a711-5864d1e0bb5a" />


## 7. DR Internet Gateway

### bu-igw
A dedicated Internet Gateway was created and attached to bu-vpc 
to provide internet connectivity for DR resources when activated 
during a failover event.

Both Internet Gateways are shown in an Attached state — one for 
main-vpc and one for bu-vpc — confirming both environments have 
independent internet connectivity.

<img width="629" height="352" alt="7 backup Internet Gateway created" src="https://github.com/user-attachments/assets/218d70bc-686d-4bee-978f-cff0c52b38da" />


## 8. DR Route Tables

### bu-public-route-table and bu-private-route-table
Two route tables were created for the DR VPC following the same 
pattern as the main VPC:

- **bu-public-route-table** — routes internet traffic through bu-igw, 
  associated with both DR public subnets
- **bu-private-route-table** — associated with both DR private subnets, 
  no NAT Gateway attached (added only during active DR failover)

Both route tables show 2 subnet associations each confirming 
correct configuration.
<img width="632" height="367" alt="9  Permission set created" src="https://github.com/user-attachments/assets/ab7ceb3a-ca89-4e3f-a1ee-c01dd6cd961f" />


<img width="634" height="352" alt="8  backup route table created" src="https://github.com/user-attachments/assets/32f39cdc-b350-4608-820e-1ac644382730" />


## 9. VPC Overview — All Three VPCs

### Complete VPC Architecture
Three VPCs make up the complete network architecture:

| VPC Name | CIDR Block | Purpose |
|---|---|---|
| main-vpc | 10.0.0.0/16 | Primary production environment |
| bu-vpc | 10.1.0.0/16 | Disaster recovery environment |
| vpc-onprem | 192.168.0.0/16 | Simulated on-premise network |

The 192.168.0.0/16 CIDR for vpc-onprem was chosen to simulate 
a real on-premise network address space which typically uses 
private RFC 1918 addresses different from cloud VPCs.


<img width="1908" height="814" alt="vpc main" src="https://github.com/user-attachments/assets/bac1f2a7-17fe-4945-b8b9-3ca7cef458f6" />
<img width="1917" height="859" alt="3  vpc" src="https://github.com/user-attachments/assets/b596baef-333f-4307-9077-f60ccb42da5f" />
<img width="1919" height="818" alt="2  vpc-onprem" src="https://github.com/user-attachments/assets/f78ac97b-4613-4238-bae9-0fd530c00102" />


## 10. IAM Identity Center — User Management

### cloud-admin User
As part of the governance framework, a dedicated admin user was 
created in IAM Identity Center instead of using the root account 
for day-to-day operations.

**User Details:**
- Username: cloud-admin
- Display name: Theodore Tsekpo
- Status: Enabled
- MFA devices: None (to be configured for production)

**Why IAM Identity Center over IAM Users:**
IAM Identity Center provides centralized access management across 
multiple AWS accounts, supports single sign-on, and integrates 
with Service Control Policies at the organization level. This 
approach scales better than managing individual IAM users per account.


<img width="632" height="367" alt="9  Permission set created" src="https://github.com/user-attachments/assets/245e738f-24fd-4597-9824-09503ca92efa" />

