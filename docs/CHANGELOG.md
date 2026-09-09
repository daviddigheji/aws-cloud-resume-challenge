# AWS Cloud Resume Challenge — Change Log

This document records significant operational, security, infrastructure,
deployment, and documentation changes made to the AWS Cloud Resume Challenge.

---

## 2026-09-09 — ACM Certificate Renewal Recovery and Documentation

### Changes

- Restored ACM DNS validation for the CloudFront TLS certificate.
- Verified successful ACM certificate renewal.
- Confirmed HTTPS availability for the production website.
- Captured ACM renewal evidence.
- Added ACM certificate renewal incident documentation.
- Added ACM certificate renewal runbook.
- Added AWS Identity Center access verification runbook.
- Added operational runbooks and incident documentation to the repository.

### Repository Update

Commit:

`f8f85fe Add operational runbooks and incident documentation`

Branch:

`main`

Result:

**Successful**

---

## 2026-08-31 — GitHub Actions AWS OIDC Deployment Migration

### Changes

- Replaced long-lived AWS access keys with GitHub OIDC authentication.
- Configured GitHub Actions to obtain temporary AWS credentials.
- Corrected AWS account ID formatting.
- Corrected OIDC account validation.
- Verified successful GitHub Actions deployment after remediation.

### Related Commits

`0578877 Replace AWS access keys with GitHub OIDC deployment`

`ee3ca8d Fix quoted AWS account ID in OIDC workflow`

`752d6ee Fix GitHub OIDC account validation`

Result:

**Successful after remediation**

---

## Change Management Principle

Operational failures are documented under:

`docs/incidents/`

Repeatable operational procedures are documented under:

`docs/runbooks/`

Successful significant changes and deployments are recorded in this change log.