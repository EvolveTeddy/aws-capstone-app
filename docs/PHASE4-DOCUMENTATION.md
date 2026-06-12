# Phase 4: Modernization & CI/CD
## AWS Capstone Project Documentation
**Author:** Theodore Tsekpo  
**Date:** June 2026  
**Region:** us-east-1

---

## Overview
Phase 4 modernizes the application architecture by introducing 
containerization, automated deployment pipelines, and security 
monitoring. This phase represents the shift from traditional 
EC2-based deployments to cloud-native, container-based workloads 
managed by AWS services.

Four major areas are covered:
1. **Container Infrastructure** — Docker, ECR, ECS Fargate
2. **Source Control** — GitHub repository for application code
3. **CI/CD Pipeline** — Automated build and deploy via CodePipeline
4. **Security & Compliance** — GuardDuty threat detection and 
   AWS Config compliance rules

---

## Part A: Container Infrastructure

---

## 1. ECR Repository & Docker Image Build

### payment-service ECR Repository
An Amazon Elastic Container Registry (ECR) private repository 
was created to store the Docker image for the payment microservice.

**Repository Details:**
- Repository Name: payment-service
- URI: 502976064775.dkr.ecr.us-east-1.amazonaws.com/payment-service
- Created: June 08, 2026, 13:04:56 UTC
- Tag Immutability: Mutable
- Encryption Type: AES-256

**Docker Build Process:**
The payment microservice was containerized using Docker in 
AWS CloudShell. The build process shows:

1. Initial attempt failed — the dot (.) at end of docker 
   build command was missing
2. Corrected command `docker build -t payment-service .` 
   succeeded with 8/8 build steps FINISHED
3. Build steps completed:
   - Loading build definition from Dockerfile
   - Loading metadata for python:3.11-slim base image
   - Pulling python:3.11-slim layers from Docker Hub
   - Setting WORKDIR /app
   - Copying app.py into the container
   - Exporting layers and writing final image

**Application Code:**
The containerized application is a Python HTTP server 
running on port 8080 that serves as the payment microservice. 
The Dockerfile uses python:3.11-slim as the base image 
keeping the container size minimal at 46.45 MB.

**Why Containerization:**
Containers package the application and all its dependencies 
together ensuring it runs identically in any environment — 
development, testing, or production. This eliminates the 
"it works on my machine" problem common in traditional 
server deployments.

<img width="1893" height="759" alt="1  Creating docker   ECR repository" src="https://github.com/user-attachments/assets/64dbff5c-fecb-4c9c-bad3-90d283d5e56a" />

---

## 2. Docker Image Pushed to ECR

### Image Successfully Stored
After building the Docker image locally in CloudShell, it 
was tagged and pushed to the ECR repository making it 
available for deployment to ECS.

**Image Details:**
- Image Tag: latest
- Image Type: Image
- Created: June 08, 2026, 13:29:35 UTC
- Image Size: 46.45 MB
- Image Digest: sha256:17f16ccbb2df312...

**Push Process:**
Three commands were executed to push the image:
1. ECR login — authenticated Docker to the ECR registry
2. Tag command — tagged the local image with the full 
   ECR repository URI
3. Push command — uploaded all image layers to ECR

The image is now centrally stored in AWS and can be 
pulled by any ECS task definition or other AWS service 
in the same account and region.

**Why ECR over Docker Hub:**
ECR integrates natively with ECS and IAM providing 
fine-grained access control. Images stored in ECR 
never leave the AWS network — no public internet 
transfer is required when ECS pulls the image. 
ECR also supports image scanning for vulnerabilities 
and lifecycle policies to manage storage costs.

<img width="1915" height="799" alt="2  image created" src="https://github.com/user-attachments/assets/fb950b60-c125-4f26-abc6-0052d31bf5ad" />

---

## 3. ECS Cluster

### payment-cluster
An Amazon ECS cluster was created to host and manage 
the containerized payment microservice. The cluster 
uses AWS Fargate as its compute engine — eliminating 
the need to provision or manage EC2 servers.

**Cluster Details:**
- Cluster Name: payment-cluster
- Status: Active
- CloudWatch Monitoring: Default
- Container Instances: 0 EC2 (Fargate — serverless)
- Services: 0
- Tasks: No tasks running (at time of screenshot)

**Why ECS Fargate over EKS:**
AWS App Runner stopped accepting new customers on 
April 30, 2026, requiring a pivot to ECS Fargate.

| Feature | ECS Fargate | EKS (Kubernetes) |
|---|---|---|
| Management overhead | Minimal | High |
| Cost | Pay per task | $73/month control plane |
| Learning curve | Moderate | Steep |
| Use case | Microservices | Large scale orchestration |
| AWS native | Fully integrated | Kubernetes ecosystem |

ECS Fargate was chosen because it provides serverless 
container execution — AWS manages all underlying 
infrastructure. EKS would be the production choice 
for large-scale deployments requiring advanced 
Kubernetes features.

**Cluster Creation Note:**
The ECS console UI failed to create the cluster due 
to a CloudFormation conflict with a failed previous 
attempt. The cluster was successfully created using 
AWS CloudShell:
```bash
aws ecs create-cluster --cluster-name payment-cluster --region us-east-1
```
This demonstrates the importance of knowing both 
console and CLI approaches — the CLI is a reliable 
fallback when the console UI encounters issues.

<img width="1916" height="799" alt="3  cluster created" src="https://github.com/user-attachments/assets/9dd933e6-813f-4da3-8345-9e2c76f60d42" />

---

## 4. ECS Task Definition

### payment-task:1
A Task Definition was created to specify exactly how 
the payment container should run — what image to use, 
how much CPU and memory to allocate, which port to 
expose, and what IAM permissions it needs.

**Task Definition Details:**
- Family: payment-task
- Revision: 1
- Status: ACTIVE
- App Environment: Fargate
- Operating System/Architecture: Linux/X86_64
- Network Mode: awsvpc
- Task Execution Role: ecsTaskExecutionRole
- Time Created: June 8, 2026, 14:50 UTC

**Resource Allocation:**
- Task CPU: 256 units (0.25 vCPU)
- Task Memory: 512 MiB (0.5 GiB)
- Container: payment-container
- Image: ECR payment-service:latest
- Port: 8080

**Why These Resource Sizes:**
0.25 vCPU and 0.5 GB memory is the minimum Fargate 
configuration. For a simple Python HTTP server serving 
demonstration traffic this is more than sufficient. 
In production, resources would be sized based on load 
testing results and traffic patterns.

**Network Mode — awsvpc:**
The awsvpc network mode gives each Fargate task its 
own elastic network interface and private IP address. 
This provides task-level network isolation and allows 
security groups to be applied directly to tasks — 
the same security model used for EC2 instances.


<img width="1898" height="805" alt="4  Task definition created" src="https://github.com/user-attachments/assets/d8433c5f-7646-4621-b89e-7daeac84c366" />

---

## 5. ECS Task Launch & Container Verification

### Task Successfully Launched
The payment-task was launched in the payment-cluster 
and the containerized microservice was verified working 
by accessing it in a browser.

**Task Launch Details:**
- Task ARN: arn:aws:ecs:us-east-1:502976064775:task/
  payment-cluster/0ea03c3731ce47dfba8db9195f0884c6
- Last Status: Provisioning → Running
- Desired Status: Running
- Task Definition: payment-task:1
- Health Status: Unknown (no health check configured)
- Created: June 8, 2026, 15:04 UTC

The cluster overview shows Tasks: Pending 1 confirming 
the task was successfully submitted and was in the 
process of being provisioned by Fargate.


<img width="1904" height="756" alt="5  Task launched successful" src="https://github.com/user-attachments/assets/9f0674d0-2741-4613-8633-7527d91bc15d" />

---

### Payment Microservice — End-to-End Verification

**Browser Test Result:**
- URL: http://3.235.250.57:8080
- Response: "Payment Microservice - Running on AWS ECS"
- Status: Successfully serving HTTP responses

This confirms the complete containerization pipeline 
is working end-to-end:
1. Code written in Python
2. Packaged into a Docker container
3. Image stored in ECR
4. Task definition references ECR image
5. ECS Fargate pulls and runs the container
6. Container accessible via public IP on port 8080

**Production Considerations:**
In production the container would sit behind an 
Application Load Balancer with HTTPS configured 
via AWS Certificate Manager. The public IP would 
not be exposed directly. An ECS Service (rather 
than a standalone task) would ensure the container 
automatically restarts if it fails.



<img width="1913" height="983" alt="6  Task Container running successfully" src="https://github.com/user-attachments/assets/2967436c-0c14-440f-a482-09be9c4f0c7c" />

---

## Part B: Source Control & CI/CD Pipeline

---

## 6. GitHub Repository

### aws-capstone-app
A GitHub repository was created to serve as the 
source of truth for application code. All changes 
pushed to this repository automatically trigger 
the CodePipeline CI/CD pipeline.

**Repository Details:**
- Owner: EvolveTeddy
- Repository Name: aws-capstone-app
- Visibility: Public
- Branch: main
- Contributors: 1 (EvolveTeddy)
- Commits: 2
- Files: README.md, index.html

**Repository Contents:**
- README.md — project overview and documentation links
- index.html — the web application served through 
  the pipeline

**Why GitHub as Source Control:**
GitHub provides free public repositories, excellent 
integration with AWS CodePipeline via GitHub App 
connections, and a familiar interface for developers. 
Every commit creates an immutable record of what 
changed, when, and by whom — essential for 
compliance and debugging.

**GitOps Principle:**
The repository represents the desired state of the 
application. When a developer pushes code to main, 
the pipeline automatically ensures the running 
application matches that desired state. This is 
the foundation of GitOps — treating infrastructure 
and application configuration as code.


<img width="1908" height="890" alt="7  Git Hub repository created" src="https://github.com/user-attachments/assets/299f706e-d003-46f8-9675-6573eaa38162" />

---

## 7. S3 Artifact Bucket

### codepipeline-artifacts-502976064775
An S3 bucket was created to store pipeline artifacts — 
the intermediate files passed between pipeline stages 
as code moves from source through build to deployment.

**Bucket Details:**
- Bucket Name: codepipeline-artifacts-502976064775
- Region: us-east-1
- Objects: 0 (empty at creation — pipeline populates it)
- Public Access: Blocked

**How Artifacts Work in CodePipeline:**
1. Source stage pulls code from GitHub and stores 
   it as a zip artifact in this bucket
2. Build stage retrieves the artifact, runs build 
   commands, and stores output artifacts back here
3. Deploy stage retrieves the built artifacts 
   and deploys them to the target (S3 in this case)

**Naming Convention:**
The bucket name includes the AWS account ID 
(502976064775) to ensure global uniqueness — 
S3 bucket names must be unique across all 
AWS accounts worldwide.

<img width="1914" height="843" alt="8  S3 bucket created" src="https://github.com/user-attachments/assets/bba8a837-1a56-41fa-af24-1423cea87fee" />

---

## 8. CodeBuild Project

### capstone-build
A CodeBuild project was created to handle the build 
stage of the pipeline — running build commands, 
tests, and producing deployable artifacts from 
the source code.

**Project Configuration:**
- Project Name: capstone-build
- Source Provider: GitHub
- Primary Repository: EvolveTeddy/aws-capstone-app 
  (tree/main branch)
- Artifacts Upload Location: None configured 
  (pipeline manages artifacts)
- Public Builds: Disabled
- Service Role: codebuild-capstone-build-service-role

**GitHub Integration:**
The source provider shows GitHub successfully connected 
to the EvolveTeddy/aws-capstone-app repository. 
This connection was established using a GitHub App 
connection — the modern, token-free authentication 
method for AWS-GitHub integration.

**Build Commands Configured:**
```yaml
version: 0.2
phases:
  build:
    commands:
      - echo "Build started"
      - echo "Build complete"
artifacts:
  files:
    - '**/*'
```

**Why CodeBuild over Jenkins:**
CodeBuild is fully managed — no server to provision 
or maintain. It scales automatically with each build 
running in an isolated environment. Jenkins requires 
a dedicated EC2 instance adding ongoing management 
overhead and cost.



<img width="1902" height="799" alt="9  codebuild created - linking aws to github" src="https://github.com/user-attachments/assets/1296d2a8-6a68-467b-96a8-f8f85bde97c3" />

---

## 9. CodePipeline — Full CI/CD Pipeline

### capstone-pipeline
A complete three-stage CI/CD pipeline was created 
connecting GitHub source code to automated build 
and deployment. Every code push to the main branch 
automatically triggers the full pipeline.

**Pipeline Details:**
- Pipeline Name: capstone-pipeline
- Status: Success — all stages passed
- Source: GitHub via GitHub App
- Build: AWS CodeBuild (Commands)
- Deploy: Amazon S3

**Pipeline Stages — All Green:**

**Stage 1 — Source:**
- Provider: GitHub (via GitHub App)
- Status: All actions succeeded 
- Trigger: Commit 8f66222 "Create index.html"
- Action: Pulls latest code from GitHub main branch

**Stage 2 — Build:**
- Provider: AWS CodeBuild (Commands)
- Status: All actions succeeded 
- Action: Runs build commands and produces artifacts

**Stage 3 — Deploy:**
- Provider: Amazon S3
- Status: All actions succeeded 
- Action: Deploys built artifacts to S3 bucket

**CI/CD Value Demonstrated:**
The pipeline ran automatically when index.html was 
committed to GitHub — no manual intervention required. 
In a full production setup this pipeline would:
1. Run automated tests in the build stage
2. Deploy to a staging environment first
3. Run integration tests
4. Deploy to production after approval
5. Roll back automatically if deployment fails


<img width="1896" height="800" alt="10  Pipeline created and running " src="https://github.com/user-attachments/assets/feb01c4e-13cc-4688-ba55-067f0f08ee0f" />

---

## Part C: Security & Compliance

---

## 10. Amazon GuardDuty

### Intelligent Threat Detection
Amazon GuardDuty was enabled to provide continuous 
intelligent threat detection across the AWS account. 
GuardDuty uses machine learning, anomaly detection, 
and integrated threat intelligence to identify 
potentially malicious activity.

**GuardDuty Dashboard:**
- Total Findings: 0
- Critical: 0
- High: 0
- Medium: 0
- Low: 0
- Resources with Findings: 0
- Accounts with Findings: 0

**What GuardDuty Monitors:**
- CloudTrail event logs — API calls and management events
- VPC Flow Logs — network traffic patterns
- DNS logs — DNS query activity
- EKS audit logs — Kubernetes API activity
- ECS runtime activity — container behaviour
- S3 data events — object access patterns

**Why Zero Findings is Expected:**
The account was just set up and contains only 
legitimate infrastructure. GuardDuty's value becomes 
apparent over time as it builds a baseline of normal 
behaviour and flags deviations. Common real-world 
findings include:
- Unusual API calls from unexpected locations
- EC2 instances communicating with known malicious IPs
- Credential access from Tor exit nodes
- Cryptocurrency mining activity

**30-Day Free Trial:**
GuardDuty provides a 30-day free trial for new 
enablement. After the trial, pricing is based on 
the volume of data analysed. For this project 
GuardDuty should be disabled after the 30-day 
trial to avoid unexpected charges.

**Runtime Monitoring:**
The dashboard shows an option to Enable Runtime 
Monitoring for ECS Fargate, EKS, and EC2 — this 
would provide deeper visibility into container 
and instance behaviour at runtime. This is an 
advanced feature for production environments.

[INSERT SCREENSHOT: 11__GuardDuty_created.png]
*Screenshot: GuardDuty Summary page showing 0 total 
findings across all severity levels confirming 
GuardDuty is active and the environment is clean*


<img width="1914" height="792" alt="11  GuardDuty created" src="https://github.com/user-attachments/assets/059907f2-87a6-4b35-b91f-483c5115c6cb" />

---

## 11. AWS Config Dashboard

### Continuous Compliance Monitoring
AWS Config was enabled to continuously record 
configuration changes across all AWS resources 
and evaluate them against compliance rules.

**Config Dashboard:**
- Conformance Packs: None deployed
- Noncompliant Rules: 0
- Compliant Rules: 0
- Noncompliant Resources: 0
- Compliant Resources: 0

**What AWS Config Does:**
AWS Config maintains a complete configuration 
history for every AWS resource. For each resource 
it records:
- What the configuration was at any point in time
- Who changed it and when
- What it changed from and to
- Whether it complies with defined rules

This creates an immutable audit trail that is 
invaluable for:
- Security investigations
- Compliance audits
- Troubleshooting configuration drift
- Understanding the blast radius of changes

**Configuration Recorder:**
The recording strategy was set to record all 
current and future resource types — ensuring 
no resource type is missed. Configuration items 
are stored in the dedicated S3 bucket 
(config-bucket-502976064775).


<img width="1919" height="802" alt="12  AWS Config done" src="https://github.com/user-attachments/assets/be9394c8-6032-4e89-9f02-f6df1d4d6bf2" />

---

## 12. AWS Config Rules

### Compliance Rules Configured
Two AWS managed compliance rules were added to 
enforce security best practices across the account. 
Both rules use DETECTIVE evaluation mode — 
continuously evaluating existing resources 
against the rule criteria.

**Rule 1 — cloudtrail-s3-bucket-public-access-prohibited**
- Type: AWS Managed
- Remediation Action: Not set
- Evaluation Mode: DETECTIVE
- Purpose: Detects any S3 bucket used by CloudTrail 
  that has public access enabled
- Why Critical: CloudTrail logs contain sensitive 
  audit data. Public access would expose API call 
  history to anyone on the internet

**Rule 2 — rds-storage-encrypted**
- Type: AWS Managed
- Remediation Action: Not set
- Evaluation Mode: DETECTIVE
- Purpose: Detects any RDS database instance where 
  storage encryption is not enabled
- Why Critical: Unencrypted databases expose data 
  at rest. If storage media is physically stolen 
  or improperly decommissioned, unencrypted data 
  can be read directly

**Detective vs Proactive Mode:**
DETECTIVE mode evaluates resources after they are 
created and flags non-compliant ones. PROACTIVE 
mode (available for some rules) evaluates resources 
before they are created using CloudFormation hooks. 
For this project DETECTIVE mode is appropriate — 
the SCPs created in Phase 1 already prevent 
creation of non-compliant resources.

**Relationship to SCPs:**
These Config rules work in conjunction with the 
Service Control Policies created in Phase 1:
- SCP: block-unencrypted-rds PREVENTS creation 
  of unencrypted databases
- Config rule: rds-storage-encrypted DETECTS 
  if any slip through or pre-existing ones exist

This two-layer approach — prevention and detection — 
is a security best practice.


<img width="1910" height="813" alt="13  Rules configured" src="https://github.com/user-attachments/assets/9a90e868-646e-444a-865e-c6158733b004" />

---

