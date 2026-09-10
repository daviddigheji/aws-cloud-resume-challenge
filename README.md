# AWS Cloud Resume Challenge — David Digheji

A production-style serverless cloud resume platform built on AWS, demonstrating cloud engineering, infrastructure automation, CI/CD, security, troubleshooting, incident response, and operational documentation.

## Live Website

https://daviddigheji.com

---

## Project Overview

This project implements a serverless resume website using AWS services and modern DevOps practices.

The project goes beyond deploying a static website. It demonstrates the operational lifecycle of a cloud application, including:

- AWS infrastructure deployment
- Serverless application integration
- CI/CD automation
- GitHub Actions authentication using AWS OIDC
- HTTPS and TLS certificate management
- Production troubleshooting
- Incident response
- Operational runbooks
- Change management
- Evidence collection and verification

---

## Architecture

```text
                    Internet
                       |
                       v
                  CloudFront
                       |
                       v
                      S3
                       |
             Static Resume Website
                       |
                       | API Request
                       v
                  API Gateway
                       |
                       v
                    Lambda
                       |
                       v
                   DynamoDB
                       |
                       v
                 Visitor Counter
```

<img width="260" height="540" alt="AWS Cloud Resume architecture" src="https://github.com/user-attachments/assets/c9697b4f-92d5-432b-8612-c37b6a1598f3" />

### Deployment Flow

```text
Developer
    |
    v
GitHub Repository
    |
    v
GitHub Actions
    |
    | AWS OIDC
    v
AWS IAM / STS
    |
    | Temporary AWS Credentials
    v
Deploy Website
    |
    +----> S3
    |
    +----> CloudFront Cache Invalidation
```

---

## Technologies Used

### AWS

- Amazon S3
- Amazon CloudFront
- Amazon API Gateway
- AWS Lambda
- Amazon DynamoDB
- AWS Certificate Manager (ACM)
- AWS IAM
- AWS Security Token Service (STS)

### DevOps and Infrastructure

- Git
- GitHub
- GitHub Actions
- GitHub OIDC
- Terraform
- AWS CLI

---

## Features

- Serverless resume website hosted on AWS
- HTTPS-enabled production website
- CloudFront content delivery
- Serverless visitor counter
- DynamoDB visitor data storage
- API Gateway frontend/backend integration
- Lambda serverless backend
- Automated deployment using GitHub Actions
- CloudFront cache invalidation after deployment
- AWS OIDC authentication for CI/CD
- Infrastructure as Code using Terraform
- Operational runbooks
- Incident documentation
- Change management
- Deployment and troubleshooting evidence

---

## CI/CD Pipeline

Changes pushed to the `main` branch are deployed using GitHub Actions.

```text
Code Change
    |
    v
Git Commit
    |
    v
GitHub Push
    |
    v
GitHub Actions
    |
    v
AWS OIDC Authentication
    |
    v
Temporary AWS Credentials
    |
    v
S3 Deployment
    |
    v
CloudFront Cache Invalidation
    |
    v
Production Verification
```

### AWS Authentication

The deployment pipeline uses **GitHub OpenID Connect (OIDC)** rather than long-lived AWS access keys.

GitHub Actions obtains short-lived AWS credentials by assuming an IAM role through AWS STS. This reduces the security risk associated with storing permanent AWS credentials in GitHub Secrets.

---

## Security

- HTTPS using Amazon CloudFront and AWS Certificate Manager
- AWS IAM least-privilege access
- GitHub Actions authentication through AWS OIDC
- Short-lived AWS STS credentials
- Removal of long-lived AWS deployment credentials
- Controlled CI/CD permissions
- TLS certificate monitoring and renewal
- Operational security verification after changes

---

## Operations and Reliability

### Runbooks

[View Operational Runbooks](docs/runbooks/)

### Incident Reports

- [2026-08-31 Website Deployment Incident](docs/incidents/2026-08-31-website-deployment-incident.md)
- [2026-08-31 GitHub OIDC Deployment Incident](docs/incidents/2026-08-31-github-oidc-deployment-incident.md)
- [2026-09-09 ACM Certificate Renewal Incident](docs/incidents/2026-09-09-acm-certificate-renewal-incident.md)

### Change Management

[View Project Change Log](docs/CHANGELOG.md)

---

## Key Troubleshooting Experience

### GitHub Actions OIDC Deployment

The project was migrated from long-lived AWS access credentials to GitHub OIDC.

During implementation, an AWS account ID / OIDC validation issue caused the deployment workflow to fail. The issue was diagnosed and corrected through these commits:

```text
0578877 Replace AWS access keys with GitHub OIDC deployment
ee3ca8d Fix quoted AWS account ID in OIDC workflow
752d6ee Fix GitHub OIDC account validation
```

The deployment pipeline was then successfully verified.

### ACM Certificate Renewal

An AWS Certificate Manager certificate used by the production website was unable to renew automatically because DNS validation could not be completed.

The DNS validation configuration was investigated and corrected, certificate renewal was verified, production HTTPS functionality was confirmed, and the recovery process was documented in both an incident report and operational runbook.

---

## What I Learned

- Building serverless applications on AWS
- Designing frontend/backend cloud integrations
- Amazon CloudFront and S3 architecture
- API Gateway and Lambda integration
- DynamoDB serverless data storage
- Infrastructure as Code
- CI/CD implementation
- GitHub Actions
- AWS IAM
- AWS STS
- GitHub OIDC federation
- Short-lived cloud credentials
- TLS/SSL certificate management
- DNS troubleshooting
- Production deployment verification
- Incident response
- Root-cause analysis
- Operational runbooks
- Change management
- Evidence-driven troubleshooting

---

## Repository Structure

```text
aws-cloud-resume-challenge/
├── .github/
├── docs/
│   ├── CHANGELOG.md
│   ├── incidents/
│   └── runbooks/
├── evidence/
├── terraform/
└── README.md
```

---

## Author

**David Digheji**

Cloud / Systems / DevOps Engineer

Portfolio: https://daviddigheji.com

GitHub: https://github.com/daviddigheji
