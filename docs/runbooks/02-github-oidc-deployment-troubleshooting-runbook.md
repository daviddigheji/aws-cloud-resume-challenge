# GitHub OIDC Deployment Troubleshooting Runbook

**Purpose:** Troubleshoot GitHub Actions authentication to AWS using OIDC and temporary STS credentials.

---

## 1. Desired authentication architecture

```text
GitHub Actions
    ↓
GitHub OIDC token
    ↓
AWS IAM OIDC provider
    ↓
IAM role trust policy
    ↓
CloudResume-GitHub-DeployRole
    ↓
STS temporary credentials
    ↓
S3 + CloudFront
```

Do not restore long-lived AWS access keys as the normal deployment model.

---

## 2. Typical symptoms

Examples:

```text
The security token included in the request is invalid
Could not assume role with OIDC
Not authorized to perform sts:AssumeRoleWithWebIdentity
AccessDenied
NoSuchEntity
```

---

## 3. Verify administrator identity first

```bash
aws sso login --profile aegis-security
```

Then:

```bash
AWS_PAGER="" aws sts get-caller-identity \
  --profile aegis-security
```

### Why

Before troubleshooting IAM, prove that you are inspecting the intended AWS account and role.

---

## 4. List registered OIDC providers

```bash
AWS_PAGER="" aws iam list-open-id-connect-providers \
  --profile aegis-security
```

### Explanation

- `iam` — selects AWS Identity and Access Management.
- `list-open-id-connect-providers` — lists OIDC identity providers registered in the account.

You are looking for a provider whose ARN corresponds to:

```text
token.actions.githubusercontent.com
```

---

## 5. Capture the GitHub OIDC provider ARN

```bash
OIDC_ARN=$(AWS_PAGER="" aws iam list-open-id-connect-providers \
  --profile aegis-security \
  --query 'OpenIDConnectProviderList[?contains(Arn, `token.actions.githubusercontent.com`)].Arn | [0]' \
  --output text)
```

Then:

```bash
echo "$OIDC_ARN"
```

### Explanation

`$()` means:

> Run the command inside the parentheses and store its output.

The JMESPath query:

```text
OpenIDConnectProviderList
→ keep ARN values containing token.actions.githubusercontent.com
→ return the first match
```

---

## 6. Inspect the OIDC provider

```bash
AWS_PAGER="" aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn "$OIDC_ARN" \
  --profile aegis-security
```

Verify:

```text
Url: token.actions.githubusercontent.com
ClientIDList contains: sts.amazonaws.com
```

This proves AWS is configured to accept GitHub's OIDC issuer with the AWS STS audience.

---

## 7. Check whether the deployment role exists

```bash
AWS_PAGER="" aws iam get-role \
  --role-name CloudResume-GitHub-DeployRole \
  --profile aegis-security
```

If AWS returns:

```text
NoSuchEntity
```

the role does not exist.

Do not create a second similarly named role until you confirm you are in the correct account.

---

## 8. Trust policy requirements

The role trust policy should restrict **who** can assume the role.

Conceptual example:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:daviddigheji/aws-cloud-resume-challenge:ref:refs/heads/main"
    }
  }
}
```

### Memory hack

```text
Trust policy = WHO can assume the role
Permissions policy = WHAT the role can do
```

---

## 9. Deployment permissions

The deployment role should receive only the permissions needed for:

```text
S3 bucket listing/location
S3 object put/delete for the website bucket
CloudFront CreateInvalidation for the website distribution
```

Avoid:

```json
"Action": "*",
"Resource": "*"
```

Do not attach `AdministratorAccess` to a CI/CD deployment role.

---

## 10. Verify the role policy

List inline policies:

```bash
AWS_PAGER="" aws iam list-role-policies \
  --role-name CloudResume-GitHub-DeployRole \
  --profile aegis-security
```

Read a specific inline policy:

```bash
AWS_PAGER="" aws iam get-role-policy \
  --role-name CloudResume-GitHub-DeployRole \
  --policy-name CloudResumeWebsiteDeployPolicy \
  --profile aegis-security
```

---

## 11. Verify GitHub workflow permissions

The workflow needs:

```yaml
permissions:
  contents: read
  id-token: write
```

`id-token: write` allows GitHub Actions to request an OIDC identity token for the job.

---

## 12. Verify role assumption in the workflow

Recommended step:

```yaml
- name: Verify AWS identity
  run: aws sts get-caller-identity
```

Expected ARN pattern:

```text
arn:aws:sts::<ACCOUNT>:assumed-role/CloudResume-GitHub-DeployRole/cloudresume-...
```

If the ARN shows the wrong role, stop before S3 or CloudFront changes.

---

## 13. Secondary issue: optional account-ID validation

During the August 2026 incident, an optional `allowed-account-ids` setting created an additional workflow failure.

Troubleshooting rule:

1. inspect the committed workflow;
2. inspect the exact error;
3. avoid repeating an assumed fix without evidence;
4. remove an optional guardrail only when stronger trust controls already exist and the error demonstrates that it is the failing element.

If removing the setting on macOS:

```bash
sed -i '' '/allowed-account-ids:/d' .github/workflows/deploy.yml
```

### Explanation

- `sed` — stream editor.
- `-i ''` — edits the file in place on macOS while creating no backup suffix.
- `/allowed-account-ids:/d` — deletes the matching line.

Verify:

```bash
grep -n "allowed-account-ids" .github/workflows/deploy.yml
```

No output means the line is absent.

---

## 14. Safe Git change procedure

Inspect changes:

```bash
git diff -- .github/workflows/deploy.yml
```

Stage only the intended file:

```bash
git add .github/workflows/deploy.yml
```

Verify staged filenames:

```bash
git diff --cached --name-only
```

Commit:

```bash
git commit -m "Fix GitHub OIDC account validation"
```

Push:

```bash
git push origin main
```

### Why not `git add .`?

It can accidentally stage unrelated files, generated content, or sensitive data.

---

## 15. Failure decision tree

```text
GitHub workflow failed
      ↓
Did configure-aws-credentials succeed?
      ↓ NO
Check id-token permission
      ↓
Check OIDC provider
      ↓
Check role trust policy
      ↓
Check repo/branch subject restriction
      ↓
Check role name/account
      ↓
Check exact workflow configuration
```

After authentication succeeds:

```text
STS identity
   ↓
S3 permissions
   ↓
CloudFront permissions
```

---

## 16. Evidence capture

Recommended files:

```text
evidence/oidc/01-original-authentication-failure.png
evidence/oidc/02-github-oidc-provider.png
evidence/oidc/03-deployment-role-trust-policy.png
evidence/oidc/04-deployment-role-permissions.png
evidence/oidc/05-workflow-oidc-configuration.png
evidence/oidc/06-sts-assumed-role-success.png
```

Never expose:

- GitHub secrets
- AWS access keys
- temporary session tokens
- private keys
- MFA seeds
