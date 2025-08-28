AWS Multi-Account / Multi-Region
---
q1)How do you deploy apps to multiple AWS accounts using a central DevOps account?
---

ans) this is a common enterprise setup where you have a central DevOps account that owns CI/CD tooling (Jenkins, CodePipeline, GitHub Actions, etc.)
and you need to deploy applications into multiple AWS accounts (Dev, QA, Prod, Business Units, etc.).

Here’s how it’s typically done:

Deployment Strategy from a Central DevOps Account

1. Use AWS Organizations + Cross-Account IAM Roles

Each target AWS account (e.g., Dev, QA, Prod) has a cross-account IAM role that the central DevOps account can assume.

Example role: CICDDeploymentRole with permissions to:

Deploy CloudFormation stacks

Push Docker images to ECR

Update ECS/EKS/EC2/Lambda, etc.

Trust policy of the role allows the DevOps account (pipeline IAM principal) to assume it

Trust Policy Example (in target account):

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<DevOpsAccountID>:role/CICDPipelineRole"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

2. CI/CD Pipeline in Central Account
   

Build happens in the DevOps account (CodeBuild, Jenkins, GitHub Actions runner, etc.).

Store artifacts in central S3/ECR.

Deploy stage assumes the cross-account role in the target account, then:

Runs aws cloudformation deploy

Or applies Terraform/Ansible

Or updates ECS/EKS clusters with the new app.

CodePipeline Example (Deploy Action):

{
  "Name": "DeployToProd",
  "ActionTypeId": {
    "Category": "Deploy",
    "Owner": "AWS",
    "Provider": "CloudFormation",
    "Version": "1"
  },
  "Configuration": {
    "ActionMode": "CREATE_UPDATE",
    "StackName": "my-app",
    "RoleArn": "arn:aws:iam::<ProdAccountID>:role/CICDDeploymentRole",
    "TemplatePath": "BuildOutput::template.yaml",
    "Capabilities": "CAPABILITY_NAMED_IAM"
  },
  "RunOrder": 1
}

3. Artifact Sharing

ECR for images → Central account pushes Docker images to ECR; other accounts pull them via cross-account permissions.

S3 for artifacts → Central account uploads build artifacts; deployment assumes role in target account and pulls from S3.

Alternative → Use AWS CodeArtifact (for packages/libraries).

4. Security Best Practices

Least privilege IAM policies for deployment roles.

Rotate keys and avoid static credentials → always use sts:AssumeRole.

Use parameterized pipelines to support many accounts/environments.

Use AWS SSM Parameter Store/Secrets Manager for sensitive values.

5. Architecture Diagram (Typical Flow)

   Developer --> GitHub/CodeCommit --> CI/CD Pipeline (DevOps Account)
      |                                      |
      v                                      v
  Build & Test --> Push Artifact (ECR/S3) --> AssumeRole into Target Account
                                               |
                                               v
                                        Deploy App (ECS/EKS/CFN/Lambda)

✅ In short:
You centralize build & CI/CD logic in one DevOps account, then use STS AssumeRole into target accounts for deployments. Artifacts are shared via ECR/S3,
and deployments are executed in the context of the target accounts, not the DevOps account.

q2) How do you design a multi-region deployment pipeline using Terraform?
---

ans) multi-region deployment with Terraform is a classic enterprise DevOps design challenge. The goal is usually:

Single source of truth (Terraform code in one repo)

Repeatable deployments across multiple AWS regions

Safe rollouts (no downtime, easy rollback)

Auditable & secure (state management, approvals, access control)

Let’s break it down:

Core Principles

1.Modularize Infrastructure

Create Terraform modules (e.g., VPC, ECS Service, RDS, S3, IAM).

Each region/environment (e.g., us-east-1, ap-south-1) uses the same modules with different variables.

2.Isolate State per Region

Use remote state (S3 + DynamoDB for locking).

Store one state file per region to avoid conflicts.

Example S3 bucket layout:

s3://org-terraform-states/
   ├── dev/us-east-1/terraform.tfstate
   ├── dev/ap-south-1/terraform.tfstate
   ├── prod/us-east-1/terraform.tfstate
   └── prod/eu-west-1/terraform.tfstate

3.Pipeline Controls Regions

The CI/CD pipeline controls which region(s) to apply to.

Can be:

Parallel deployments → deploy to multiple regions at once.

Sequential deployments → deploy to one region, run health checks, then promote to next.

terraform/
├── modules/
│   ├── vpc/
│   ├── ecs/
│   ├── rds/
│   └── s3/
├── envs/
│   ├── dev/
│   │   ├── us-east-1.tfvars
│   │   └── main.tf
│   ├── stage/
│   │   ├── us-east-1.tfvars
│   │   └── ap-south-1.tfvars
│   └── prod/
│       ├── us-east-1.tfvars
│       └── ap-south-1.tfvars
└── backend.tf


Terraform Multi-Region Setup
--

Example Terraform root module:

# providers.tf
provider "aws" {
  alias  = "use1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "aps1"
  region = "ap-south-1"
}

module "app_us_east_1" {
  source   = "./modules/app"
  providers = { aws = aws.use1 }
  env      = "prod"
  region   = "us-east-1"
}

module "app_ap_south_1" {
  source   = "./modules/app"
  providers = { aws = aws.aps1 }
  env      = "prod"
  region   = "ap-south-1"
}

Pipeline Design (Example with GitHub Actions / Jenkins / CodePipeline)
---

Pipeline Stages

Lint & Validate → terraform fmt, terraform validate

Plan (per region) → terraform plan -var-file=prod.use1.tfvars

Manual Approval (optional, for prod)

Apply (per region)

Parallel Deployment Example

Two jobs, each runs Terraform against different TF_STATE + REGION.

Sequential Deployment Example

Deploy to us-east-1, run smoke tests, then promote pipeline to ap-south-1.

GitHub Actions Example (Multi-Region)
--

name: Multi-Region Terraform Pipeline

on:
  push:
    branches: [ main ]

jobs:
  terraform-us-east-1:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -backend-config="key=prod/us-east-1/terraform.tfstate"
      - run: terraform plan -var-file="prod.us-east-1.tfvars"
      - run: terraform apply -auto-approve -var-file="prod.us-east-1.tfvars"

  terraform-ap-south-1:
    runs-on: ubuntu-latest
    needs: terraform-us-east-1
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -backend-config="key=prod/ap-south-1/terraform.tfstate"
      - run: terraform plan -var-file="prod.ap-south-1.tfvars"
      - run: terraform apply -auto-approve -var-file="prod.ap-south-1.tfvars"

Best Practices

✅ State isolation → separate state files per region + environment
✅ Workspaces or folders → use clear separation (env/region)
✅ Promotion flow → deploy → test → promote
✅ Global resources (like Route53, IAM, CloudFront) → manage separately from region-specific infra
✅ DR/HA design → use Route53 latency-based routing / Global Accelerator for traffic shifting across regions
✅ Secrets → use AWS SSM Parameter Store/Secrets Manager, not plain tfvars

In short:
A multi-region Terraform pipeline is just repeatable modules + isolated state + CI/CD orchestration per region. The pipeline decides where and in what order to run Terraform

