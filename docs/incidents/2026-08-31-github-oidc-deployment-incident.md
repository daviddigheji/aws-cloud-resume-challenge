# GitHub Actions OIDC Deployment Incident

**Incident date:** 2026-08-31  
**Project:** AWS Cloud Resume Challenge  
**Service/component:** GitHub Actions CI/CD, AWS IAM, GitHub OIDC federation  
**Severity:** SEV-3 / Low-to-moderate impact  
**Status:** Resolved  
**Owner:** David Digheji  

---

## 1. Incident Summary

A deployment from the `aws-cloud-resume-challenge` GitHub repository failed while GitHub Actions was authenticating to AWS through OpenID Connect (OIDC).

The project had been migrated away from long-lived AWS access keys toward GitHub OIDC so that GitHub Actions could obtain short-lived AWS credentials by assuming an IAM role.

During the migration, the workflow contained an AWS account ID formatting/validation issue. This prevented the OIDC authentication step from completing successfully and blocked the deployment.

The configuration was corrected, the workflow was re-run, and the deployment subsequently completed successfully.

---

## 2. Impact

The incident affected the CI/CD deployment pipeline.

### User impact

- Production deployment from GitHub Actions was temporarily blocked.
- New website changes could not be deployed through the automated pipeline until the authentication issue was resolved.
- The existing production website remained available.

### Security impact

No evidence of credential exposure or unauthorized AWS access was identified.

The migration itself improved the security posture because GitHub Actions no longer needed long-lived AWS access keys stored as repository secrets.

---

## 3. Detection

The issue was detected when the GitHub Actions deployment workflow failed during AWS authentication / role assumption.

The troubleshooting sequence can also be seen in the repository commit history:

```text
0578877 Replace AWS access keys with GitHub OIDC deployment
ee3ca8d Fix quoted AWS account ID in OIDC workflow
752d6ee Fix GitHub OIDC account validation
```

These commits show the migration to OIDC followed by two corrective changes required to make authentication work correctly.

---

## 4. Symptoms

Observed symptoms included:

- GitHub Actions deployment workflow failed.
- AWS authentication through GitHub OIDC did not complete successfully.
- The workflow could not continue to the AWS deployment steps.
- The AWS account ID / OIDC configuration required correction before role assumption could succeed.

The exact raw GitHub Actions error text is not reproduced here because the incident record is intended to document the operational failure and resolution without exposing unnecessary account-specific information.

---

## 5. Technical Background

GitHub Actions can authenticate to AWS without storing permanent AWS access keys.

The trust flow is:

```text
GitHub Actions
      |
      | requests OIDC token
      v
GitHub OIDC provider
      |
      | presents signed identity token
      v
AWS IAM role trust policy
      |
      | validates repository / branch claims
      v
AWS STS AssumeRoleWithWebIdentity
      |
      | returns temporary credentials
      v
GitHub Actions deploys to AWS
```

This is preferable to long-lived IAM access keys because the credentials are:

- temporary;
- generated only when the workflow runs;
- scoped to an IAM role;
- controlled through the IAM trust policy;
- not stored permanently in GitHub Secrets.

---

## 6. Root Cause

The root cause was an AWS account ID formatting/validation problem in the GitHub Actions OIDC workflow configuration during the migration from long-lived AWS access keys to OIDC authentication.

Repository history shows two corrective changes:

1. Removal/correction of a quoted AWS account ID value.
2. Correction of GitHub OIDC AWS account validation.

The incorrectly formatted or validated account value caused the AWS authentication step to fail before deployment could continue.

---

## 7. Contributing Factors

The following factors contributed to the incident:

- The authentication method was being changed from static AWS credentials to OIDC.
- OIDC configuration requires multiple components to match correctly:
  - GitHub workflow permissions;
  - AWS account ID;
  - IAM role ARN;
  - IAM OIDC provider;
  - IAM trust policy;
  - repository and branch conditions.
- Small formatting errors in an account identifier or role configuration can cause the entire authentication step to fail.

---

## 8. Resolution

The issue was resolved by correcting the GitHub Actions OIDC configuration.

The relevant fixes were committed as:

```text
ee3ca8d Fix quoted AWS account ID in OIDC workflow
752d6ee Fix GitHub OIDC account validation
```

After the corrections:

1. The workflow was re-run.
2. GitHub successfully authenticated to AWS using OIDC.
3. Temporary AWS credentials were obtained.
4. The deployment workflow continued.
5. The GitHub Actions workflow completed successfully.

---

## 9. Verification

The resolution was verified using the following checks.

### 9.1 Git repository history

Command:

```bash
git --no-pager log --oneline -10
```

Purpose:

- `git` runs Git.
- `--no-pager` prints the result directly in the terminal instead of opening the `less` viewer.
- `log` displays commit history.
- `--oneline` shows one compact line per commit.
- `-10` limits the output to the latest ten commits.

Expected evidence included:

```text
752d6ee Fix GitHub OIDC account validation
ee3ca8d Fix quoted AWS account ID in OIDC workflow
0578877 Replace AWS access keys with GitHub OIDC deployment
```

### 9.2 Repository synchronization

Command:

```bash
git status
```

Purpose:

Checks whether the local `main` branch is synchronized with `origin/main` and whether uncommitted changes remain.

Expected result:

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

### 9.3 GitHub Actions

The latest deployment workflow was checked in:

```text
GitHub repository
→ Actions
→ deployment workflow
```

Verification criteria:

- workflow status was green / successful;
- AWS authentication step succeeded;
- deployment steps completed;
- branch was `main`.

### 9.4 Production validation

The production site was checked after deployment to confirm that:

- the website loaded successfully;
- HTTPS worked;
- CloudFront served the site correctly;
- the deployment had not introduced an outage.

---

## 10. Corrective and Preventive Actions

### Completed

- [x] Replaced long-lived AWS access keys with GitHub OIDC.
- [x] Corrected AWS account ID formatting.
- [x] Corrected OIDC account validation.
- [x] Re-ran and verified the GitHub Actions deployment.
- [x] Confirmed repository synchronization.
- [x] Retained OIDC as the deployment authentication mechanism.

### Preventive controls

- Keep using GitHub OIDC instead of long-lived AWS access keys.
- Restrict the IAM role trust policy to the intended GitHub repository and branch.
- Keep GitHub Actions permissions at the minimum required level.
- Review workflow YAML changes before merging.
- Validate AWS account and role values carefully.
- Avoid printing credentials, tokens, or sensitive AWS identifiers in logs.
- Capture evidence after major CI/CD authentication changes.
- Keep the related troubleshooting runbook current.

---

## 11. Related Runbook

Related operational documentation:

```text
docs/runbooks/02-github-oidc-deployment-troubleshooting-runbook.md
```

The runbook should be used when a future GitHub Actions deployment fails during AWS authentication or role assumption.

---

## 12. Related Commits

```text
0578877 Replace AWS access keys with GitHub OIDC deployment
ee3ca8d Fix quoted AWS account ID in OIDC workflow
752d6ee Fix GitHub OIDC account validation
```

---

## 13. Lessons Learned

1. OIDC significantly improves CI/CD credential security but must be configured precisely.
2. A small formatting issue can block the whole deployment pipeline.
3. Authentication failures should be investigated before changing unrelated deployment components.
4. Git commit history is valuable operational evidence because it shows the sequence of remediation.
5. A successful GitHub Actions run should be followed by production verification.
6. Incidents and runbooks should be maintained separately:
   - the incident records what happened;
   - the runbook explains how to diagnose and recover from a similar issue in the future.

---

## 14. Closure

The incident was closed after GitHub Actions successfully authenticated to AWS using OIDC and the deployment pipeline completed successfully.

The project now uses short-lived AWS credentials through OIDC rather than permanent AWS access keys, improving both operational reliability and security.

**Final status: RESOLVED**
