# Production-Grade AWS ECS Fargate Platform — Infrastructure as Code

A complete, enterprise-level AWS infrastructure built entirely with **AWS CloudFormation** across **7 interconnected stacks**. This platform covers everything from network foundation to CI/CD pipeline — designed to run containerized applications in production with full observability, security, and automation.

---

## What This Platform Does

This project provisions a fully working cloud platform where you can:

- Deploy containerized applications using **ECS Fargate** (serverless containers — no EC2 to manage)
- Route traffic through an **Application Load Balancer** with optional HTTPS and WAF protection
- Store Docker images in **ECR** with automatic vulnerability scanning
- Connect to a managed **RDS database** with auto-rotating credentials
- Automatically build, scan, and deploy new code through a **CI/CD pipeline** connected to GitHub
- Monitor everything through **CloudWatch dashboards and alarms** with email notifications

All infrastructure is defined as code — no manual clicking in the AWS console. Deploy once, reuse across Dev, Staging, and Production environments.

---

## Stack Overview

The platform is split into 7 CloudFormation stacks. Each stack has a specific responsibility and they connect to each other through **cross-stack exports**.

| # | Stack | File | Purpose |
|---|-------|------|---------|
| 1 | VPC | `vpc-new.yaml` | Network foundation — subnets, routing, security |
| 2 | ALB | `alb.yaml` | Load balancer — receives internet traffic |
| 3 | ECR | `ecr.yaml` | Container image registry |
| 4 | ECS | `ecs.yaml` | Runs your containers on Fargate |
| 5 | RDS | `rds.yaml` | Managed relational database |
| 6 | Other ECS | `other-ecs-new.yaml` | Add-on services on the same cluster |
| 7 | Pipeline | `Pipeline.yaml` | CI/CD — build, scan, deploy automatically |

> Deploy in the order listed above. Each stack depends on the one before it.

---

## Deployment Order & Cross-Stack Dependencies

```
VPC  →  ALB  →  ECR  →  ECS  →  RDS  →  Other ECS  →  Pipeline
```

- ALB imports VPC subnet IDs and VPC ID
- ECS imports VPC subnets, ALB security group, target group ARN, and listener ARNs
- RDS imports VPC ID, subnets, and ECS task security group ID
- Other ECS imports the ECS cluster name and ALB listener ARNs
- Pipeline imports the ECR repository URI and ECS cluster/service names

---

## Naming Convention

Every resource across all 7 stacks follows the same pattern:

```
{ProjectName}-{Environment}-{ResourceType}
```

Examples:
- `Myproject-Dev-Vpc`
- `Myproject-Prod-Alb`
- `Myproject-Staging-EcsCluster`
- `Myproject-Dev-Rds`

**ProjectName** must be PascalCase (e.g. `Myproject`, `MyApp`) and must match exactly across all stacks for cross-stack imports to work.

**Environment** must be one of: `Dev`, `Staging`, `Prod`

---

## Stack 1 — VPC (Network Foundation)

**File:** `vpc-new.yaml` | **Version:** 4.1.0

The VPC is the network container for everything else. It creates a fully isolated private network inside AWS.

### What it creates

- 1 VPC with a configurable CIDR block (e.g. `10.0.0.0/16`)
- 3 Public Subnets across 3 Availability Zones — for the load balancer and NAT gateways
- 3 Private Subnets across 3 Availability Zones — for ECS tasks and RDS (never directly reachable from internet)
- Internet Gateway — allows public subnets to reach the internet
- NAT Gateway(s) — allows private subnets to reach the internet for outbound calls (e.g. pulling Docker images)
- Route Tables — controls where traffic flows
- Shared Security Group — a reusable security group exported for downstream stacks

### NAT Gateway Options

| Option | Cost | Use Case |
|--------|------|----------|
| Single NAT (`UseSingleNat=true`) | ~$35/month | Dev and Staging |
| One NAT per AZ (`UseSingleNat=false`) | ~$105/month | Production (high availability) |

### Optional Features (all off by default)

- **VPC Flow Logs** — captures all network traffic for security auditing. Can send to CloudWatch Logs or S3 (cheaper for long-term storage)
- **Gateway VPC Endpoints** — free private routes to S3 and DynamoDB without going through NAT
- **Interface VPC Endpoints** — private routes to ECR, SSM, Secrets Manager, and CloudWatch Logs (avoids NAT costs for these services, ~$7/endpoint/AZ/month)
- **Custom Network ACLs** — subnet-level firewall rules (in addition to security groups)

### Exports (used by other stacks)

`VpcId`, `VpcCidrBlock`, `SharedSgId`, `PublicSubnetOneId`, `PublicSubnetTwoId`, `PublicSubnetThreeId`, `PrivateSubnetOneId`, `PrivateSubnetTwoId`, `PrivateSubnetThreeId`, `AvailabilityZones`, `NatPublicIp`

---

## Stack 2 — ALB (Application Load Balancer)

**File:** `alb.yaml` | **Version:** 3.1.0

The ALB sits in front of your application and distributes incoming HTTP/HTTPS traffic to your ECS containers. It also handles health checking — if a container is unhealthy, the ALB stops sending it traffic.

### What it creates

- Application Load Balancer (internet-facing or internal)
- Target Group — the group of ECS tasks that receive traffic
- HTTP Listener on port 80
- Optional HTTPS Listener on port 443 (requires an ACM certificate)
- ALB Security Group — controls who can reach the load balancer
- Optional S3 bucket for access logs (stores every request for compliance/debugging)
- Optional WAFv2 Web ACL — protects against common attacks and rate limits per IP

### How HTTP and HTTPS work together

- If HTTPS is enabled: HTTP port 80 automatically redirects to HTTPS port 443
- If HTTPS is disabled: HTTP port 80 forwards directly to ECS containers

### CloudWatch Alarms (6 static + 3 anomaly detection)

| Alarm | What it detects |
|-------|----------------|
| High Request Count | Traffic spike or DDoS |
| High 5XX Error Rate | Backend application failures |
| High 4XX Error Rate | Client-side errors |
| High Response Time | Slow backend (DB bottleneck, CPU pressure) |
| Unhealthy Hosts | Containers failing health checks |
| Rejected Connections | ALB at capacity or no healthy targets |
| Request Count Anomaly (ML) | Unusual traffic patterns vs baseline |
| 5XX Anomaly (ML) | Unusual error spikes vs baseline |
| Response Time Anomaly (ML) | Unusual latency vs baseline |

All alarms can send email notifications via SNS. A CloudWatch dashboard shows all metrics in one view.

### Design Note — Dual ALB Resource Pattern

CloudFormation does not support conditional attributes on the same resource. Because access logs require a different set of `LoadBalancerAttributes`, this template creates **two ALB resources** — one with access logs enabled and one without. Only one is actually created based on your `EnableAccessLogs` parameter. This is a known CloudFormation workaround.

### Exports

`AlbArn`, `AlbDnsName`, `AlbSgId`, `TgArn`, `HttpListenerArn`, `HttpsListenerArn`

---

## Stack 3 — ECR (Container Image Registry)

**File:** `ecr.yaml`

ECR is where your Docker images are stored. Every time the pipeline builds a new version of your application, it pushes the image here. ECS then pulls from here when deploying.

### What it creates

- ECR Repository to store Docker images
- Image scanning on every push (finds security vulnerabilities in your container)
- Lifecycle policy — automatically cleans up old images to save storage costs
- Optional SNS alert when CRITICAL or HIGH vulnerabilities are found in a scan
- Audit log group — records every image push and delete event via EventBridge

### Image Scanning

| Type | Cost | How it works |
|------|------|-------------|
| BASIC | Free | Scans on push using Clair engine |
| ENHANCED | Paid | Amazon Inspector — continuous scanning, not just on push |

### Lifecycle Policy

Two rules run automatically:
1. Delete untagged images older than N days (default: 3 days) — these are old builds that got replaced
2. Keep only the newest N tagged images (default: 20) — your rollback history

### Design Note — Dual Repository Pattern

CloudFormation does not support conditional `DeletionPolicy` on a single resource. To allow choosing between `Retain` (keep images when stack is deleted) and `Delete` (remove everything), this template creates **two repository resources** with different deletion policies. Only one is created based on your `DeletionProtection` parameter.

### Exports

`RepoName`, `RepoArn`, `RepoUri`, `ScanFindingsTopicArn`

---

## Stack 4 — ECS (Fargate Compute)

**File:** `ecs.yaml` | **Version:** 12.0.0

This is where your application actually runs. ECS Fargate runs containers without you managing any servers — AWS handles the underlying compute.

### What it creates

- ECS Cluster (shared — named `ProjectName-Environment-EcsCluster`)
- Per service: ECS Task Definition, ECS Service, Security Group, IAM roles
- Auto Scaling — scales container count up/down based on CPU and memory
- CloudWatch Alarms per service — CPU, memory, task count, response time
- CloudWatch Dashboard per service — built dynamically by a Lambda function
- Optional XRay / ADOT sidecar container for distributed tracing

### How containers are named

```
Container name: {ProjectName}-{Environment}-{ServiceName}-Container
Task definition: {ProjectName}-{Environment}-{ServiceName}-TaskDef
Service: {ProjectName}-{Environment}-{ServiceName}-Service
```

### SSM Environment Variable Injection

Instead of hardcoding environment variables in the task definition, this stack supports loading them from **AWS Systems Manager Parameter Store**. A Lambda-backed custom resource reads parameters from a specified SSM path and injects them into the task definition at deploy time. This keeps secrets out of CloudFormation templates.

### Phantom-Free Dashboard Design

A common problem with CloudWatch dashboards is "phantom alarms" — widgets that reference alarms which don't exist because a feature was disabled. This stack uses a **Lambda custom resource** that dynamically builds the dashboard JSON, passing only the ARNs of alarms that were actually created. The result is a clean dashboard with no broken widgets.

### IAM Roles Created Per Service

- **Task Execution Role** — allows ECS to pull images from ECR and write logs to CloudWatch
- **Task Role** — the permissions your application code has at runtime (e.g. read from S3, write to DynamoDB)

### Exports

`EcsClusterName`, `EcsClusterArn`, `EcsTaskSgId`, per-service task definition ARN and service ARN

---

## Stack 5 — RDS (Relational Database)

**File:** `rds.yaml` | **Version:** 2.3.0

RDS provides a managed relational database. AWS handles backups, patching, failover, and password rotation automatically.

### What it creates

- RDS DB Instance (PostgreSQL, MySQL, or MariaDB)
- DB Subnet Group — places the database in private subnets
- DB Security Group — controls which resources can connect to the database
- Optional Enhanced Monitoring IAM Role

### Password Management

This stack uses `ManageMasterUserPassword: true` — AWS automatically generates a strong password and stores it in **Secrets Manager**. The password rotates automatically. Your application reads the password from Secrets Manager at runtime. You never see or manage the password manually.

The Secrets Manager ARN is exported as `RdsMasterSecretArn` so your ECS task can be granted permission to read it.

### Database Access Modes

| Mode | Who can connect | Use case |
|------|----------------|----------|
| `EcsTasks` | Only ECS task security group | Most secure — production |
| `VpcCidr` | Any resource in the VPC | Dev/testing |
| `CustomSecurityGroup` | A specific security group you provide | Flexible |

### CloudWatch Alarms (10 static + 4 anomaly detection)

| Alarm | Metric |
|-------|--------|
| High CPU | CPU % above threshold |
| Low Burst Balance | CPU credits depleted (t3/t4g only) |
| Low Freeable Memory | RAM running low |
| Low Free Storage | Disk space running low |
| High Connections | Too many active connections |
| High Read Latency | Slow reads from disk |
| High Write Latency | Slow writes to disk |
| High Replica Lag | Read replica falling behind |
| High Swap Usage | Instance swapping to disk |
| High Disk Queue Depth | I/O requests queuing up |
| CPU Anomaly (ML) | Unusual CPU pattern |
| Connections Anomaly (ML) | Unusual connection count |
| Read Latency Anomaly (ML) | Unusual read latency |
| Write Latency Anomaly (ML) | Unusual write latency |

### Exports

`RdsEndpoint`, `RdsPort`, `RdsSgId`, `RdsMasterSecretArn`

---

## Stack 6 — Other ECS (Add-On Services)

**File:** `other-ecs-new.yaml` | **Version:** 3.0.0

This stack adds more services to the existing ECS cluster created by Stack 4. Instead of creating a new cluster, it imports the existing one. This is how you add a second, third, or fourth microservice to the same cluster.

### What it creates

- Additional ECS Task Definition and Service on the existing cluster
- New Target Group for the service
- ALB Listener Rule — routes traffic based on host header (e.g. `api.example.com` goes to service A, `admin.example.com` goes to service B)
- Auto Scaling and CloudWatch Alarms (same pattern as Stack 4)
- Dynamic CloudWatch Dashboard (same phantom-free Lambda pattern as Stack 4)

### How routing works

The ALB has one listener. Multiple services share it using **host-header rules**:
- Request for `app.example.com` → forwards to main ECS service (Stack 4)
- Request for `api.example.com` → forwards to add-on service (Stack 6)

This avoids needing a separate load balancer per service, saving cost.

### Imports from other stacks

- ECS Cluster name from Stack 4
- ALB HTTP/HTTPS Listener ARNs from Stack 2
- VPC and subnet IDs from Stack 1

---

## Stack 7 — Pipeline (CI/CD)

**File:** `Pipeline.yaml` | **Version:** 2.0.0

The pipeline automates the entire software delivery process. When a developer pushes code to GitHub, the pipeline automatically builds the Docker image, scans it for vulnerabilities, runs tests, and deploys to ECS — with optional manual approval before production.

### Pipeline Stages

```
GitHub Push
    ↓
Source (CodeStar Connection to GitHub)
    ↓
Build (CodeBuild — Docker build + push to ECR)
    ↓
Scan Gate (optional — checks ECR scan results)
    ↓
Test (optional — run automated tests)
    ↓
Manual Approval (optional — human must approve before deploy)
    ↓
Deploy (ECS rolling deployment)
```

### Build Stage

CodeBuild runs a Docker build, tags the image with the Git commit SHA, and pushes it to ECR. The commit SHA tag means every deployment is traceable back to an exact code commit.

### Scan Gate (Security Checkpoint)

This is one of the most important features. After the image is pushed to ECR, a separate CodeBuild project runs a Python script that:

1. Calls the ECR API to get the vulnerability scan results
2. Counts CRITICAL, HIGH, and MEDIUM findings
3. Compares against configurable thresholds
4. **Fails the pipeline** if thresholds are exceeded — the deployment never reaches ECS

This means a container with known critical vulnerabilities cannot be deployed, even accidentally.

### Notifications

EventBridge rules send SNS notifications for:
- Pipeline succeeded
- Pipeline failed
- Manual approval is waiting (so someone knows to review)

### Exports

`PipelineName`, `BuildProject`, `ScanGateProject`

---

## Security Design

Security decisions made throughout this platform:

- **No hardcoded passwords** — RDS uses Secrets Manager with automatic rotation
- **Private subnets for compute** — ECS tasks and RDS are never directly reachable from the internet
- **Least-privilege IAM** — each ECS service has its own task role with only the permissions it needs
- **WAF protection** — optional WAFv2 with AWS managed rule sets and per-IP rate limiting
- **Image scanning** — every Docker image is scanned before deployment; pipeline blocks on critical findings
- **Encryption at rest** — RDS storage encrypted, ECR images encrypted, S3 buckets encrypted with AES-256
- **VPC endpoints** — optional private routes to AWS services that avoid internet traversal
- **Flow logs** — optional network traffic capture for security auditing and incident investigation
- **Deletion protection** — RDS and ECR have configurable deletion protection to prevent accidental data loss

---

## Observability Design

Every layer of the stack has monitoring built in:

| Layer | What is monitored |
|-------|------------------|
| ALB | Request count, error rates, response time, unhealthy hosts, rejected connections |
| ECS | CPU, memory, running task count, deployment events |
| RDS | CPU, memory, storage, connections, read/write latency, disk queue, swap |
| ECR | Image scan findings via EventBridge alerts |
| Pipeline | Success, failure, and approval events via EventBridge |

All alarms support:
- **Static thresholds** — alert when a metric crosses a fixed value
- **ML-powered anomaly detection** — alert when a metric behaves unusually compared to its historical baseline (no threshold to configure)
- **SNS email notifications** — get alerted immediately when something goes wrong

---

## Technologies Used

| Category | Technology |
|----------|-----------|
| Infrastructure as Code | AWS CloudFormation |
| Compute | Amazon ECS Fargate |
| Container Registry | Amazon ECR |
| Load Balancing | AWS Application Load Balancer |
| Database | Amazon RDS (PostgreSQL / MySQL / MariaDB) |
| Secret Management | AWS Secrets Manager |
| CI/CD | AWS CodePipeline, AWS CodeBuild |
| Source Control | GitHub (via AWS CodeStar Connection) |
| Security | AWS WAFv2, VPC Security Groups, Network ACLs |
| Monitoring | Amazon CloudWatch Alarms, Dashboards, Anomaly Detection |
| Notifications | Amazon SNS, Amazon EventBridge |
| Parameter Store | AWS Systems Manager Parameter Store |
| Tracing (optional) | AWS X-Ray, AWS Distro for OpenTelemetry |
| Networking | Amazon VPC, NAT Gateway, VPC Endpoints, Internet Gateway |

---

## Key Design Patterns

### Dual Resource Pattern
CloudFormation does not support conditional `DeletionPolicy` or conditional resource attributes. Where this limitation exists (ECR deletion policy, ALB access logs), two separate resources are defined and only one is created based on a condition. This is a production-grade workaround, not a mistake.

### Phantom-Free Dashboards
CloudWatch dashboard widgets that reference non-existent alarms show as broken. To avoid this, ECS and Other ECS use Lambda-backed custom resources that build dashboard JSON dynamically — only including widgets for alarms that were actually created.

### SSM Environment Injection
Environment variables for ECS tasks are loaded from SSM Parameter Store at deploy time via a Lambda custom resource. This avoids storing configuration and secrets inside CloudFormation templates or task definitions.

### Consistent Cross-Stack Naming
All 7 stacks use the same `ProjectName-Environment` prefix for every export name. This makes cross-stack imports predictable and eliminates hardcoded ARNs or IDs between stacks.

---

## How to Deploy

### Prerequisites
- AWS CLI configured with appropriate permissions
- An AWS account
- A GitHub repository with your application code (for Pipeline stack)
- An ACM certificate in the same region (for HTTPS on ALB)

### Deployment Steps

**Step 1 — Deploy VPC**
```bash
aws cloudformation deploy \
  --template-file vpc-new.yaml \
  --stack-name Myproject-Dev-Vpc \
  --parameter-overrides ProjectName=Myproject Environment=Dev
```

**Step 2 — Deploy ALB**
```bash
aws cloudformation deploy \
  --template-file alb.yaml \
  --stack-name Myproject-Dev-Alb \
  --parameter-overrides ProjectName=Myproject Environment=Dev
```

**Step 3 — Deploy ECR**
```bash
aws cloudformation deploy \
  --template-file ecr.yaml \
  --stack-name Myproject-Dev-Ecr \
  --parameter-overrides ProjectName=Myproject Environment=Dev RepositoryName=myproject-app
```

**Step 4 — Deploy ECS**
```bash
aws cloudformation deploy \
  --template-file ecs.yaml \
  --stack-name Myproject-Dev-Ecs \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides ProjectName=Myproject Environment=Dev
```

**Step 5 — Deploy RDS**
```bash
aws cloudformation deploy \
  --template-file rds.yaml \
  --stack-name Myproject-Dev-Rds \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides ProjectName=Myproject Environment=Dev DBInstanceIdentifier=myproject-dev-db
```

**Step 6 — Deploy Pipeline**
```bash
aws cloudformation deploy \
  --template-file Pipeline.yaml \
  --stack-name Myproject-Dev-Pipeline \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides ProjectName=Myproject Environment=Dev
```

> The `--capabilities CAPABILITY_IAM` flag is required for stacks that create IAM roles.

---

## Parameter Consistency Rule

The following parameters **must match exactly** across all stacks you deploy together:

- `ProjectName` — must be identical in all stacks
- `Environment` — must be identical in all stacks
- `Owner` — used for tagging
- `CostCenter` — used for billing allocation

If these do not match, cross-stack imports will fail with an export not found error.

---

## Author

Built and deployed by a Senior DevOps Engineer as a demonstration of production-grade AWS infrastructure design using CloudFormation IaC.
