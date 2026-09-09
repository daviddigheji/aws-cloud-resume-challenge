# AWS IAM Identity Center Access & Identity Verification Runbook

**Purpose:** Securely authenticate to AWS from the CLI and Management Console using IAM Identity Center and temporary credentials.

---

## 1. CLI login

Run:

```bash
aws sso login --profile aegis-security
```

### Meaning

- `aws` — AWS CLI.
- `sso` — IAM Identity Center / SSO functionality.
- `login` — starts authentication.
- `--profile aegis-security` — selects the saved SSO-enabled profile.

A browser authentication flow may open.

---

## 2. Verify the caller

```bash
aws sts get-caller-identity --profile aegis-security
```

### What is STS?

`STS` means **AWS Security Token Service**.

AWS STS works with temporary security credentials and assumed-role sessions.

### What this command asks

> Which AWS identity is making this request?

Typical fields:

```text
UserId
Account
Arn
```

Expected ARN pattern when using an assumed role:

```text
arn:aws:sts::<ACCOUNT>:assumed-role/<ROLE>/<SESSION>
```

---

## 3. Why this matters

Before changing IAM, S3, CloudFront, ACM, Route 53, or other resources, verify:

```text
correct account
correct role
correct session
```

This prevents accidental work in the wrong AWS account or under an unexpected identity.

---

## 4. Open the AWS Management Console

Use the IAM Identity Center Access Portal configured for the account.

Public documentation should use:

```text
<AWS_ACCESS_PORTAL_URL>
```

rather than publishing the actual portal URL unnecessarily.

In the portal:

```text
Accounts
→ select AWS account
→ select assigned permission set
→ Management console
```

Once inside the Management Console, the top-right corner shows the current account and role/session.

---

## 5. Region awareness

Always verify the active region before opening a regional service.

Examples from this project:

```text
Primary project region: eu-west-2 (London)
CloudFront viewer ACM certificate: us-east-1 (N. Virginia)
```

The browser URL can also reveal the current region, for example:

```text
https://us-east-1.console.aws.amazon.com/...
```

---

## 6. Check profile configuration

To inspect the profile:

```bash
aws configure list --profile aegis-security
```

To inspect the AWS CLI config file carefully:

```bash
cat ~/.aws/config
```

### Security note

Do not paste the entire configuration publicly if it contains internal account or portal details you do not intend to expose.

---

## 7. Verify profile-specific SSO values

Depending on CLI configuration style, the SSO start URL may be stored under an `sso-session`.

Inspect:

```bash
grep -A 8 -B 2 "aegis-security" ~/.aws/config
```

If the profile references an SSO session, inspect that named section as well.

---

## 8. Common failure modes

### SSO token expired

Symptom:

```text
The SSO session associated with this profile has expired
```

Fix:

```bash
aws sso login --profile aegis-security
```

### Wrong account/role

Detect with:

```bash
aws sts get-caller-identity --profile aegis-security
```

Do not continue until the identity is correct.

### Wrong region

Symptoms may include:

```text
resource not found
certificate not visible
service appears empty
```

Check the region selector before assuming the resource does not exist.

---

## 9. Evidence capture

For private project evidence:

```text
evidence/identity/01-sso-login-success.png
evidence/identity/02-sts-caller-identity.png
evidence/identity/03-console-role-session.png
```

For a public repository:

- redact account ID if unnecessary;
- redact Access Portal URL;
- never show tokens or MFA secrets;
- keep the role/session pattern visible only when it adds useful evidence.
