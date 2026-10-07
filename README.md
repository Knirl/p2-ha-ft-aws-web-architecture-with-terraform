# PROJECT 2: Infrastructure as Code (IaC) with Terraform for AWS Web Architecture 

A Terraform reimplementation of [Project 1: Highly Available, Fault-Tolerant AWS Web Architecture](https://github.com/Knirl/aws-ha-fault-tolerant-architecture.git) — same infrastructure, same design goals, this time provisioned entirely as code instead of built manually through the AWS Console.

**Project 1 summary, for context:** a VPC spanning 2 Availability Zones, an Application Load Balancer distributing traffic to an Auto Scaling Group of EC2 instances (private subnets), a self-healing Multi-AZ RDS MySQL database, an S3 bucket, and CloudWatch dashboards/alarms wired to SNS — built and manually verified to survive instance failure, security group misconfiguration, and AZ-level database failover. Full architecture rationale, the "why these services" comparisons, and the original testing/troubleshooting log live in that repo's README — this document focuses on what changed by rebuilding it in Terraform.

## Table of Contents

- [Architecture](#architecture)
- [What this project intentionally does NOT include yet](#what-this-project-intentionally-does-not-include-yet)
- [What Changed, Building This in Terraform](#what-changed-building-this-in-terraform)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Build & Deploy](#build--deploy)
- [Verification & Testing](#verification--testing)
- [New Challenges Specific to the IaC Rebuild](#new-challenges-specific-to-the-iac-rebuild)
- [Cost Considerations](#cost-considerations)
- [Teardown](#teardown)
- [Roadmap: v2](#roadmap-v2)

---

## Architecture

Unchanged from Project 1:

![Architecture Diagram](./project1-architecture-diagram.png)

Security model, also unchanged: `Internet → alb-sg (0.0.0.0/0:80) → ec2-sg (from alb-sg only:80) → rds-sg (from ec2-sg only:3306)`.

## What this project intentionally does NOT include yet

Before the step-by-step summary, it's important to be explicit about this: **v1 uses no variables, no `variables.tf`, no `outputs.tf`, and no modules.** Every value (bucket names, CIDR blocks, instance types, region) is hardcoded directly into the resource blocks, and every resource lives in one flat folder (`project2/`) rather than being split into reusable modules. This was a deliberate choice, not an oversight — the goal of v1 was to learn and correctly implement each AWS service and Terraform concept without also juggling the added abstraction of variables/modules at the same time. Fixing this is the entire point of v2 (see the last section of this document).


## What Changed, Building This in Terraform

The final architecture is identical to the Console build, but implementing it as code forced me to explicitly architect every layer:

* **The Console hides structural complexity.** Project 1's networking layer was created via the "VPC and more" wizard in a single guided flow. Terraform has no equivalent shortcut—the VPC, 4 subnets, Internet Gateway, NAT Gateway, Elastic IP, 2 route tables, and their associations all had to be declared as 10+ individual resources. Rebuilding it as code clarified every lower-level routing and gateway component the wizard abstracts away.

* **Dependency ordering becomes explicit, not assumed.** In the Console, user interface flow implicitly enforces step ordering (you cannot attach a route table to a subnet before the subnet exists). In Terraform, resource references (e.g., a security group rule referencing another security group's ID) dynamically build an implicit dependency graph. Where no direct resource attribute reference exists but provision ordering is mandatory—such as forcing the Internet Gateway to deploy before the NAT Gateway—`depends_on` must be explicitly declared.

* **Secrets management becomes an intentional design pattern.** While the AWS Console can auto-generate credentials behind a single checkbox, writing the workflow in Terraform required deliberate architectural choices: generating credentials dynamically via `random_password`, committing them to AWS Secrets Manager, and referencing them directly inside the RDS resource block. This ensures sensitive credentials never exist in plain text within version control (though access controls must be enforced on the state file where state metadata resides).

* **State introduces a dedicated operational layer.** Unlike Console builds, Terraform relies on a state engine to map declared code to real-world AWS resource IDs. This introduced a foundational bootstrap step—provisioning an encrypted S3 bucket for remote state before the primary stack could initialize—solving the circular dependency of requiring infrastructure to store state.

* **Infrastructure reproducibility is mathematically enforced.** The entire multi-tier stack—NAT Gateway, ALB, Multi-AZ RDS, and scaling policies—was torn down and recreated from scratch across testing cycles via `terraform apply` and `terraform destroy`. This eliminated configuration drift and avoided manually repeating dozens of UI steps.

## Prerequisites

- An AWS account (this project uses paid resources — see [Cost Considerations](#cost-considerations))
- [Terraform CLI](https://developer.hashicorp.com/terraform/downloads) installed locally
- AWS CLI configured with an IAM user's programmatic access keys (not the root account)
- An AWS budget alert configured (recommended, not required) to catch unexpected spend

## Project Structure

```
.
├── state-bootstrap/          # One-time setup: creates the S3 bucket that holds
│   └── main.tf                # this project's remote state. Run once, then left alone.
│
└── project2/                  # The actual infrastructure
    ├── main.tf                 # terraform + backend + provider blocks
    ├── vpc.tf                   # VPC and 4 subnets across 2 AZs
    ├── networking.tf            # Internet Gateway, NAT Gateway, route tables
    ├── security-groups.tf       # Chained alb-sg → ec2-sg → rds-sg
    ├── rds.tf                   # DB subnet group, Secrets Manager password, RDS instance
    ├── ec2.tf                   # AMI lookup, Launch Template, Auto Scaling Group + policy
    ├── alb.tf                   # Target Group, Load Balancer, Listener
    ├── s3.tf                    # Private, versioned, encrypted static assets bucket
    ├── cloudwatch.tf             # Dashboard, alarm, SNS topic + subscription
    └── .gitignore                # Excludes state files, .tfvars, and .terraform/
```

## Build & Deploy

```bash
# One-time: bootstrap the remote state bucket
cd state-bootstrap
terraform init
terraform apply

# Deploy the actual architecture
cd ../project2
terraform init
terraform plan
terraform apply
```

Terraform resolves the correct creation order automatically from resource references — networking, then security groups, then RDS/EC2/ALB, then CloudWatch — with no manual sequencing required. Full resource-by-resource detail is in the `.tf` files themselves, each named by concern (see [Project Structure](#project-structure)).

## Verification & Testing

Same manual verification process as Project 1 — Terraform provisions infrastructure, it doesn't test runtime behavior, so testing still happens via the AWS Console and browser after `apply` completes: load balancing (Instance ID alternating on refresh), auto-healing (security group removed → Target Group unhealthy → alarm → SNS email → ASG replacement attempts → recovery on restoring the rule), and RDS failover (`Reboot with failover`, confirmed by the AZ change). See Project 1's README for the full walkthrough and results — behavior was identical here.

## New Challenges Specific to the IaC Rebuild

Issues that only came up because this was built in Terraform (see Project 1's README for the original Console-build issues, which don't repeat here):

* **Adopting S3 native Lockfiles Over DynamoDB:** Rather than provisioning a separate AWS DynamoDB table solely to manage state lock records, this build takes advantage of modern Terraform backend capabilities by using `use_lockfile = true`. This leverage S3's native conditional-write locking, eliminating additional resource overhead while maintaining concurrent execution safeguards.

* **Secrets Manager Retention Window:** Standard AWS Secrets Manager behavior holds deleted secrets in a 30-day recovery window. Setting `recovery_window_in_days = 0` was necessary to ensure that running `terraform destroy` completely purges secret material, allowing instant re-creation during iterative `terraform apply` test runs without dynamic naming collisions.


## Cost Considerations

| Resource | Free tier eligible? | Notes |
|---|---|---|
| EC2 t2/t3.micro | Yes (750 hrs/mo, 12 months) | |
| RDS db.t3.micro, single-AZ | Yes | |
| **RDS Multi-AZ** | No | ~doubles single-AZ cost |
| S3 | Yes (5GB) | |
| CloudWatch (basic) | Yes | |
| **Application Load Balancer** | No | ~$16–20/month |
| **NAT Gateway** | No | ~$32+/month + data processing |

Non-free-tier resources were run for a short, deliberate build-test-teardown window. Because everything is defined in Terraform, the full environment can be rebuilt in minutes (aside from RDS provisioning time) whenever needed again for review.

## Teardown

```bash
cd project2
terraform destroy
```

Terraform resolves deletion order automatically from its dependency graph. `state-bootstrap/` is intentionally left running rather than destroyed alongside it, since it holds this project's remote state.

## Roadmap: [v2](https://github.com/Knirl/p2p3-ha-ft-aws-web-architecture-terraform-cicd)

Everything below is a list of improvements for this project to level up, next project will be the version 2.

## Infrastructure & Architecture

- **Modular Terraform** — Refactored the flat configuration into reusable modules and decoupled the ASG from the ALB with `aws_autoscaling_attachment`.
- **Parameterized infrastructure** — Replaced hardcoded values such as CIDRs and instance sizes with typed variables and environment-specific configuration.
- **Input validation** — Added Terraform `validation` blocks to catch invalid configuration values early.
- **Standardized tagging** — Centralized common tags with `default_tags` and applied consistent resource tagging across modules.
- **Improved outputs** — Exposed useful infrastructure details such as ALB DNS, RDS endpoint, and VPC IDs without exposing credentials.

## Security & Secrets Management

- **KMS encryption** — Added a Customer Managed KMS Key for RDS and Secrets Manager.
- **Secure database credentials** — Generate credentials with `random_password` and store connection details in AWS Secrets Manager.
- **Secure EC2 access** — Added IAM roles/instance profiles and AWS Systems Manager Session Manager, eliminating the need for SSH access.
- **Least-privilege S3 access** — Restricted EC2 S3 permissions to the required bucket and actions.
- **Improved S3 security** — Enabled versioning, encryption, and S3 Block Public Access.

## Scalability & High Availability

- **Improved Auto Scaling** — Prevented Terraform from resetting ASG capacity changed by AWS Auto Scaling with `ignore_changes`.
- **Multi-AZ NAT Gateways** — Added one NAT Gateway per AZ with separate private routing for improved availability.
- **Configurable RDS availability** — Added an `rds_multi_az` option for environment-specific database availability.

## Cost & Environment Controls

- **Configurable cost controls** — Added toggles for cost-intensive resources such as NAT Gateways and Multi-AZ RDS.
- **Environment-aware infrastructure** — Added `dev`, `staging`, and `prod` configuration to control availability, cost, and production safeguards.

## Application Integration & Engineering Practices

- **S3 integration** — Separated the Terraform state bucket from application storage and integrated the application bucket with EC2 through IAM.
- **Terraform validation & formatting** — Use `terraform fmt -recursive` and `terraform validate` as part of the development workflow.
- **Security scanning** — Use Trivy to identify and document infrastructure security findings.
- **Configuration hygiene** — Added `terraform.tfvars.example` and excluded actual `terraform.tfvars` from version control.

