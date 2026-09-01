# Secure GitOps Infrastructure

Terraform infrastructure for the Secure GitOps Platform running on AWS EKS.

This repository provides the AWS infrastructure required to run a secure, GitOps-driven Kubernetes platform, including networking, Amazon EKS, managed node groups, IAM, ECR, encryption, and supporting AWS resources.

---

## Architecture

The lab infrastructure uses a cost-conscious AWS network design with two public
subnets and no NAT Gateway. EKS worker nodes are deployed into these public
subnets, while the Kubernetes API endpoint supports both private and restricted
public access.

    Internet
        |
        v
    +----------------------+
    |      AWS Region      |
    |     ap-south-1       |
    +----------+-----------+
               |
               v
    +----------------------+
    |         VPC          |
    |     10.20.0.0/16     |
    +----------+-----------+
               |
        +------+------+
        |             |
        v             v
    Public Subnet  Public Subnet
    10.20.1.0/24   10.20.2.0/24
        |             |
        +------+------+
               |
               v
    +----------------------+
    |      Amazon EKS      |
    | secure-gitops-cluster|
    +----------+-----------+
               |
        +------+------+
        |             |
        v             v
    EKS Node Group    EKS Add-ons
      general         CoreDNS
      t3.large        VPC CNI
                      kube-proxy
                      EBS CSI
                      Pod Identity
               |
               v
             Argo CD
               |
        +------+-------+
        |      |       |
        v      v       v
       Dev  Staging Production

Network characteristics:

- VPC CIDR: `10.20.0.0/16`
- Availability Zones: 2
- Public subnets: `10.20.1.0/24` and `10.20.2.0/24`
- NAT Gateway: disabled for the lab
- DNS support: enabled
- DNS hostnames: enabled
- EKS private API access: enabled
- EKS public API access: enabled
- Public EKS API access is restricted to an explicit `/32` CIDR
---

## Project Overview

This repository is the Infrastructure as Code layer of the Secure GitOps Platform.

The overall project separates infrastructure provisioning from application delivery.

    Terraform
        |
        v
    AWS Infrastructure
        |
        v
    Amazon EKS
        |
        v
    Argo CD
        |
        v
    GitOps Configuration
        |
        v
    Kubernetes Applications

Terraform manages the AWS infrastructure.

Argo CD manages Kubernetes application deployment.

---

## Repository Structure

    secure-gitops-infrastructure/
    |
    +-- bootstrap/
    |   +-- main.tf
    |   +-- providers.tf
    |   +-- variables.tf
    |   +-- outputs.tf
    |   +-- versions.tf
    |   +-- terraform.tfvars.example
    |
    +-- environments/
    |   |
    |   +-- lab/
    |   |   +-- backend.tf
    |   |   +-- ecr.tf
    |   |   +-- eks.tf
    |   |   +-- iam.tf
    |   |   +-- network.tf
    |   |   +-- outputs.tf
    |   |   +-- providers.tf
    |   |   +-- variables.tf
    |   |   +-- versions.tf
    |   |
    |   +-- github-oidc-ecr/
    |       +-- ecr.tf
    |       +-- oidc.tf
    |       +-- outputs.tf
    |       +-- providers.tf
    |       +-- variables.tf
    |       +-- versions.tf
    |
    +-- docs/
    |   +-- screenshots/
    |
    +-- .gitignore
    +-- README.md

---

# Infrastructure Components

## AWS VPC

The lab environment provisions a dedicated VPC for the Secure GitOps Platform.

The lab infrastructure includes:

- VPC
- Two public subnets
- Internet Gateway
- Route tables
- Route table associations
- Security groups
- Network ACL configuration
- VPC flow logging where configured

NAT Gateway is intentionally disabled in the lab to reduce cost.

## Amazon EKS

The lab environment provisions an Amazon EKS cluster.

Example cluster:

    secure-gitops-cluster

AWS Region:

    ap-south-1

The EKS environment includes:

- EKS cluster
- Managed node group
- IAM roles
- EKS access configuration
- EKS add-ons
- Pod identity configuration where enabled
- KMS encryption
- Kubernetes networking configuration

Verify the cluster:

    aws eks describe-cluster \
      --name secure-gitops-cluster \
      --region ap-south-1

Verify Kubernetes connectivity:

    kubectl get nodes

Expected result:

    NAME                                         STATUS
    ip-10-20-x-x.ap-south-1.compute.internal    Ready

---

# EKS Security

The Kubernetes API endpoint should not be unnecessarily exposed to the entire Internet.

For the lab environment, public endpoint access can be restricted to the administrator's current public IP.

Find the current public IP:

    curl https://api.ipify.org
    echo

Check the configured EKS public endpoint restrictions:

    aws eks describe-cluster \
      --name secure-gitops-cluster \
      --region ap-south-1 \
      --query "cluster.resourcesVpcConfig.publicAccessCidrs"

For a single administrator IP, use a /32 CIDR.

Example:

    223.123.14.165/32

If the public IP changes, Kubernetes API access may stop working.

Update the allowed CIDR:

    aws eks update-cluster-config \
      --region ap-south-1 \
      --name secure-gitops-cluster \
      --resources-vpc-config \
      endpointPublicAccess=true,publicAccessCidrs="YOUR_PUBLIC_IP/32"

Refresh kubeconfig:

    aws eks update-kubeconfig \
      --region ap-south-1 \
      --name secure-gitops-cluster

Test:

    kubectl get nodes

---

# Encrypted Node Storage

EKS worker node root volumes should be encrypted.

Encryption is provided using AWS KMS where configured.

Security considerations include:

- Encrypted EBS volumes
- IAM roles
- Security groups
- Restricted Kubernetes API access
- EKS access controls
- Worker nodes deployed in the lab subnets defined by Terraform

---

# Amazon ECR

Amazon Elastic Container Registry is used to store container images.

The container delivery flow is:

    Developer
        |
        v
    GitHub
        |
        v
    GitHub Actions
        |
        v
    Docker Build
        |
        v
    Amazon ECR
        |
        v
    Kubernetes / Argo CD

ECR lifecycle policies can be used to remove old images and control storage costs.

---

# GitHub OIDC

The repository contains infrastructure for GitHub Actions authentication using OpenID Connect.

The objective is to avoid storing long-lived AWS access keys inside GitHub Actions.

Authentication flow:

    GitHub Actions
          |
          | OIDC token
          v
       AWS IAM
          |
          | AssumeRoleWithWebIdentity
          v
    Temporary AWS credentials
          |
          v
       AWS resources

This provides a more secure authentication model for CI/CD.

---

# IAM

IAM resources are defined using Terraform.

The infrastructure uses:

- IAM roles
- IAM policies
- IAM policy attachments
- EKS access configuration
- GitHub OIDC trust relationships
- Service-specific permissions

The principle of least privilege should be applied whenever permissions are added or modified.

Avoid unnecessarily broad permissions such as:

    Action   = "*"
    Resource = "*"

unless there is a documented infrastructure requirement.

---

# Encryption

Security-sensitive infrastructure uses AWS encryption mechanisms where supported.

Examples include:

- AWS KMS
- Encrypted EBS volumes
- EKS secrets encryption where configured
- S3 server-side encryption where applicable

Encryption settings should be reviewed before production deployment.

---

# Terraform

Terraform is the Infrastructure as Code tool used by this repository.

The environment used during Phase 16 validation reported:

    Terraform v1.15.5

Terraform version should be pinned or managed consistently for reproducible infrastructure deployments.

---

# Terraform Workflow

The standard Terraform workflow is:

    terraform init
          |
          v
    terraform fmt
          |
          v
    terraform validate
          |
          v
    terraform plan
          |
          v
    terraform apply

For infrastructure destruction:

    terraform plan -destroy
          |
          v
    terraform destroy

---

# Terraform Initialization

Move into the lab environment:

    cd environments/lab

Initialize Terraform:

    terraform init

If the environment uses a remote backend, Terraform initializes the configured backend.

---

# Terraform Formatting

Check formatting:

    terraform fmt -check -recursive

Automatically format Terraform:

    terraform fmt -recursive

---

# Terraform Validation

Validate the configuration:

    terraform validate

Expected result:

    Success! The configuration is valid.

The lab environment was validated successfully during Phase 16.

---

# Terraform Plan

Before applying infrastructure changes:

    terraform plan

Review the plan carefully.

Terraform should clearly show the proposed operations.

Example:

    Plan: X to add, Y to change, Z to destroy.

Never blindly apply a plan containing unexpected resource destruction.

---

# Terraform Apply

Apply infrastructure changes:

    terraform apply

Review the proposed changes and confirm only when the changes are expected.

For production infrastructure, prefer an approved CI/CD workflow rather than uncontrolled manual changes.

---

# Terraform Destroy

The lab environment is designed to be disposable.

Before destroying the environment:

    terraform plan -destroy

Review all resources that will be removed.

If the destruction plan is expected:

    terraform destroy

Terraform destroy should only be executed after all required evidence has been captured.

---

# Terraform State Security

Terraform state can contain sensitive infrastructure information.

Terraform state files must not be committed to Git.

The repository ignores Terraform state files using:

    *.tfstate
    *.tfstate.*

Verify that no Terraform state is tracked:

    git ls-files | grep -E 'terraform\.tfstate' || echo "No Terraform state tracked"

Expected result:

    No Terraform state tracked

Verify ignored state files:

    git check-ignore -v \
      terraform.tfstate \
      bootstrap/terraform.tfstate \
      bootstrap/terraform.tfstate.backup

The repository was verified during Phase 16 and no Terraform state files were tracked.

---

# Terraform Lock Files

Terraform provider lock files such as:

    .terraform.lock.hcl

should normally be committed.

They help maintain provider version consistency and improve reproducibility.

---

# CI/CD

Terraform infrastructure changes should be validated automatically through CI.

Recommended workflow:

    Pull Request
         |
         v
    Terraform Format Check
         |
         v
    Terraform Validate
         |
         v
    Terraform Plan
         |
         v
    Review
         |
         v
    Merge
         |
         v
    Apply

CI should fail if formatting or validation fails.

---

# Infrastructure Security Audit

Before infrastructure changes are merged, review:

- Publicly exposed resources
- Unrestricted security groups
- Excessive IAM permissions
- Unencrypted storage
- Public S3 buckets
- Open Kubernetes API access
- Hard-coded credentials
- Secrets committed to Git
- Unnecessary AWS resources
- Unexpected Terraform destruction

---

# Secrets Management

Never commit credentials or secrets to this repository.

Do not commit:

- AWS access keys
- AWS secret keys
- Passwords
- Private keys
- Kubernetes tokens
- API tokens
- Database passwords
- Terraform sensitive values

Use appropriate secret-management mechanisms such as:

- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- Kubernetes Secrets
- GitHub Actions Secrets
- GitHub OIDC

Long-lived AWS credentials should be avoided in CI/CD.

---

# AWS Authentication

Before running AWS or Terraform commands, verify the active AWS identity:

    aws sts get-caller-identity

Always verify that the correct AWS account and identity are being used before modifying infrastructure.

---

# EKS Access

Update the local kubeconfig:

    aws eks update-kubeconfig \
      --region ap-south-1 \
      --name secure-gitops-cluster

Verify connectivity:

    kubectl get nodes

Verify Argo CD applications:

    kubectl get applications -n argocd

---

# GitOps Architecture

The complete delivery architecture is:

    Developer
        |
        v
    GitHub Application Repository
        |
        v
    GitHub Actions
        |
        +----------------+
        |                |
        v                v
    Docker Build       Tests
        |
        v
    Amazon ECR
        |
        v
    GitOps Configuration
        |
        v
    Argo CD
        |
        v
    Amazon EKS
        |
        +-------------------+
        |                   |
        v                   v
      Dev/Staging       Production

Terraform manages the AWS layer.

Argo CD manages the Kubernetes application layer.

---

# Related Repositories

The Secure GitOps project is divided into multiple repositories.

## Application Repository

Repository:

    secure-gitops-app

Purpose:

- Application source code
- Dockerfile
- Application configuration
- Application CI/CD

---

## Platform Repository

Repository:

    secure-gitops-platform

Purpose:

- Argo CD configuration
- ApplicationSets
- Argo CD Projects
- Monitoring configuration
- Environment values
- Kubernetes platform configuration
- GitOps deployment configuration

---

## Helm Repository

Repository:

    secure-gitops-helm-chart

Purpose:

- Helm chart
- Kubernetes templates
- Application deployment configuration
- Service configuration
- ConfigMaps
- Secrets references
- Monitoring configuration

---

# Environment Model

The platform contains separate environments:

    dev
    staging
    production

Terraform provides the underlying AWS infrastructure.

The GitOps repository controls application deployment.

This separation provides:

- Controlled promotion
- Environment-specific configuration
- Git-based auditing
- Repeatable deployments
- Reduced manual Kubernetes changes

---

# Monitoring

The Secure GitOps Platform includes Kubernetes monitoring.

The monitoring stack includes components such as:

- Prometheus
- Grafana
- Alertmanager
- kube-state-metrics
- node-exporter

Application metrics are exposed through Prometheus-compatible endpoints where configured.

Example metrics endpoint:

    /metrics

Monitoring architecture:

    Kubernetes
        |
        +------------------+
        |                  |
        v                  v
    Application        Node Metrics
        |                  |
        +--------+---------+
                 |
                 v
             Prometheus
                 |
          +------+------+
          |             |
          v             v
       Grafana      Alertmanager

During platform validation, Prometheus successfully reported application and infrastructure scrape targets as UP.

---

# Git Workflow

The main branch is protected.

Infrastructure changes should use feature branches and pull requests.

Create a feature branch:

    git switch -c feature/my-change

Make the required changes.

Run validation:

    terraform fmt -check -recursive
    terraform validate

Review changes:

    git diff

Check status:

    git status

Commit:

    git add .
    git commit -m "feat: describe infrastructure change"

Push:

    git push -u origin feature/my-change

Create a pull request.

After CI passes and the change is reviewed, merge the pull request.

---

# Branch Protection

The `main` branch is intended to use a pull-request-based workflow.

Recommended protection rules:

- Pull requests required for normal changes
- Required Terraform validation checks
- Successful CI validation before merge
- Code review where appropriate
- Direct pushes to `main` should not be used for normal changes

The repository currently uses pull requests for infrastructure changes. Branch
protection should be enabled in GitHub repository settings so that the required
CI checks are enforced server-side.
---

# Troubleshooting

## kubectl Cannot Connect to EKS

First check the current public IP:

    curl https://api.ipify.org
    echo

Check the EKS public access restrictions:

    aws eks describe-cluster \
      --name secure-gitops-cluster \
      --region ap-south-1 \
      --query "cluster.resourcesVpcConfig.publicAccessCidrs"

If the IP has changed, update the CIDR:

    aws eks update-cluster-config \
      --region ap-south-1 \
      --name secure-gitops-cluster \
      --resources-vpc-config \
      endpointPublicAccess=true,publicAccessCidrs="YOUR_PUBLIC_IP/32"

Then update kubeconfig:

    aws eks update-kubeconfig \
      --region ap-south-1 \
      --name secure-gitops-cluster

Test:

    kubectl get nodes

---

# Important EKS Public IP Issue

A home or ISP public IP can change.

For example, if the previous allowed IP was:

    223.123.11.169/32

but the current public IP becomes:

    223.123.5.218

the EKS API endpoint will reject the new source IP.

This results in errors such as:

    dial tcp <EKS_ENDPOINT>:443: i/o timeout

The solution is to update the EKS public endpoint CIDR to the current public IP.

Always use:

    CURRENT_PUBLIC_IP/32

rather than only the IP address.

---

# Terraform Validation Failure

Run:

    terraform fmt -recursive

Then:

    terraform validate

If initialization is required:

    terraform init

Review the resulting configuration and Git diff.

---

# Unexpected Terraform Changes

Never immediately run:

    terraform apply

when the plan contains unexpected changes.

Instead run:

    terraform plan

Look for:

- destroy
- replace
- change

Pay particular attention to:

- EKS cluster
- EKS node groups
- VPC
- NAT Gateway
- IAM roles
- Security groups
- ECR
- KMS resources

---

# Cost Management

AWS infrastructure can continue generating costs even when applications are not being actively used.

For a temporary lab environment, infrastructure should be destroyed when no longer required.

Before destruction:

    terraform plan -destroy

Then:

    terraform destroy

Before destruction:

- Capture final evidence
- Verify Git repositories are pushed
- Verify required documentation is committed
- Verify Terraform state is safe
- Verify no required AWS resources will be lost accidentally

---

# Phase 16

Phase 16 is the final hardening and cleanup phase of the Secure GitOps project.

Objectives:

    1. Final security/repository audit
    2. Improve infrastructure README
    3. Improve platform README + architecture diagram
    4. Add/verify Terraform CI
    5. Add/verify Helm CI
    6. Final GitHub audit
    7. Capture final evidence
    8. Terraform destroy

The final destruction step is intentional for the lab environment and is used to prevent unnecessary AWS costs after the project is completed.

---

# Phase 16 Infrastructure Validation

The infrastructure repository was checked for Terraform configuration.

Terraform files include:

    bootstrap/main.tf
    bootstrap/providers.tf
    bootstrap/variables.tf
    bootstrap/outputs.tf
    bootstrap/versions.tf

    environments/lab/backend.tf
    environments/lab/ecr.tf
    environments/lab/eks.tf
    environments/lab/iam.tf
    environments/lab/network.tf
    environments/lab/outputs.tf
    environments/lab/providers.tf
    environments/lab/variables.tf
    environments/lab/versions.tf

    environments/github-oidc-ecr/ecr.tf
    environments/github-oidc-ecr/oidc.tf
    environments/github-oidc-ecr/outputs.tf
    environments/github-oidc-ecr/providers.tf
    environments/github-oidc-ecr/variables.tf
    environments/github-oidc-ecr/versions.tf

Terraform formatting and validation were successfully executed for the lab environment.

---

# Phase 16 State Security Verification

The repository was checked for Terraform state tracking.

Command:

    git ls-files | grep -E 'terraform\.tfstate' || echo "No Terraform state tracked"

Result:

    No Terraform state tracked

Terraform state files are ignored through:

    *.tfstate
    *.tfstate.*

This prevents local Terraform state from being accidentally committed.

---

# Phase 16 Git Verification

Before completing the phase:

    git status

The working tree should be clean.

Review recent commits:

    git log --oneline -5

Verify remote:

    git remote -v

Infrastructure changes should be merged through pull requests rather than direct pushes to main.

---

# Final Validation Checklist

## Infrastructure

- [ ] Terraform formatting passes
- [ ] Terraform validation passes
- [ ] Terraform plan reviewed
- [ ] No Terraform state tracked
- [ ] No secrets committed
- [ ] EKS security reviewed
- [ ] Node root volumes encrypted
- [ ] IAM reviewed
- [ ] Network security reviewed
- [ ] ECR configuration reviewed

## Git

- [ ] Working tree clean
- [ ] Feature branches used
- [ ] Pull requests used
- [ ] Main branch protected
- [ ] CI checks passing
- [ ] No sensitive files committed

## Kubernetes

- [ ] EKS cluster healthy
- [ ] Nodes Ready
- [ ] Argo CD healthy
- [ ] Applications synced
- [ ] Monitoring healthy
- [ ] Metrics available

## Documentation

- [ ] Infrastructure README complete
- [ ] Platform README complete
- [ ] Architecture documented
- [ ] Evidence captured
- [ ] Repository structure documented
- [ ] Security controls documented

## Cost Control

- [ ] Final evidence captured
- [ ] Terraform destroy plan reviewed
- [ ] Lab infrastructure destroyed when no longer required

---

# Final Verification Commands

Run Terraform validation:

    cd environments/lab
    terraform fmt -check -recursive
    terraform validate

Check Git:

    git status
    git log --oneline -5

Check EKS:

    kubectl get nodes

Check Argo CD:

    kubectl get applications -n argocd

Check monitoring:

    kubectl get pods -n monitoring

Check ServiceMonitors:

    kubectl get servicemonitor -A

Check Terraform state tracking:

    git ls-files | grep -E 'terraform\.tfstate' || echo "No Terraform state tracked"

---

# Project Outcome

The Secure GitOps project demonstrates a production-oriented DevOps and Cloud Engineering workflow using:

- AWS
- Amazon EKS
- Terraform
- Kubernetes
- Docker
- Amazon ECR
- IAM
- GitHub
- GitHub Actions
- GitHub OIDC
- Argo CD
- Helm
- Prometheus
- Grafana
- Alertmanager
- GitOps
- Infrastructure as Code
- CI/CD
- Security controls
- Monitoring and observability

The architecture separates infrastructure provisioning from application delivery while keeping both processes automated, auditable, and Git-driven.

---

# Security Principles

The project follows these core security principles:

1. Infrastructure as Code
2. Least privilege IAM
3. Encrypted infrastructure
4. Restricted EKS API access
5. GitHub OIDC instead of long-lived CI credentials
6. Protected main branches
7. Pull-request based changes
8. Automated validation
9. No Terraform state committed
10. No secrets committed
11. Git-based auditability
12. Disposable lab infrastructure

---

# Operational Principles

The project follows these operational principles:

- Validate before apply
- Review Terraform plans
- Avoid manual Kubernetes drift
- Use Git as the desired-state source
- Use Argo CD for Kubernetes reconciliation
- Monitor infrastructure and applications
- Capture evidence before destructive operations
- Destroy unused lab resources to control costs

---

# Author

Muhammad Baqir Nawaz

Cloud / DevOps Engineer

GitHub:

    https://github.com/baqir-ops

Project:

    Secure GitOps Platform

---

# License

This project is intended for learning, portfolio development, and demonstration of DevOps and Cloud Engineering practices.

Before using this infrastructure in production, perform an organization-specific security, reliability, cost, and compliance review.
