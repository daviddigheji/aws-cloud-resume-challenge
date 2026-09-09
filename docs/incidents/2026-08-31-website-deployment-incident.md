# Cloud Resume Website Deployment Incident & Recovery Runbook

**Project:** AWS Cloud Resume Challenge  
**Website:** `daviddigheji.com`  
**Repository:** `aws-cloud-resume-challenge`  
**AWS account used during troubleshooting:** `<AWS_ACCOUNT_ID>`  
**AWS Region:** `eu-west-2` (London)  
**Website S3 bucket:** `daviddigheji.com`  
**CloudFront distribution:** `E2ODEMEIX05YY8`  
**Deployment role:** `CloudResume-GitHub-DeployRole`  
**Deployment permission policy:** `CloudResumeWebsiteDeployPolicy`  
**Administrative CLI profile:** `aegis-security`  
**Incident date:** 31 August 2026  
**Runbook status:** Authentication remediation and workflow correction were pushed to `main`. Final green GitHub Actions run and live-site verification should be recorded after they are confirmed.

---

## 1. Purpose of this runbook

This document records the real troubleshooting work performed when the source files for `daviddigheji.com` were updated in GitHub but the public website continued to display the older version.

It is designed to serve four purposes:

1. **Recovery guide** — reproduce the troubleshooting process if the website stops updating again.
2. **Rebuild guide** — recreate the deployment pipeline securely from a fresh AWS/GitHub setup.
3. **Study note** — explain every important command, option, IAM concept, and result.
4. **Interview preparation** — turn the incident into a credible Cloud/DevOps/Security engineering story.

The core engineering principle used throughout the incident was:

> **Observe → Isolate → Verify → Change → Verify again → Document.**

### Memory hack

**O-I-V-C-V-D**

- **O**bserve the symptom.
- **I**solate the first failing component.
- **V**erify with evidence.
- **C**hange only what is necessary.
- **V**erify again.
- **D**ocument what happened.

---

## 2. Architecture before the incident

The website deployment path was intended to be:

```text
Local Git repository
        ↓
GitHub repository (main)
        ↓
GitHub Actions
        ↓
AWS authentication
        ↓
S3 bucket: daviddigheji.com
        ↓
CloudFront: E2ODEMEIX05YY8
        ↓
daviddigheji.com
```

However, the original GitHub Actions workflow authenticated to AWS using long-lived repository secrets:

```yaml
aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

That meant the deployment depended on a persistent AWS access key remaining valid.

### Risk of the old model

Long-lived access keys create several operational and security problems:

- they can remain valid until explicitly disabled or deleted;
- they must be rotated;
- they can leak through CI/CD configuration or other secret stores;
- they are difficult to tie to one specific workflow run;
- when the key is removed during security cleanup, automation may suddenly stop working.

The incident exposed exactly that weakness.

---

# Part I — Incident diagnosis

## 3. Initial symptom

The observed state was:

```text
GitHub repository contains updated website files     YES
Live daviddigheji.com displays those updates         NO
GitHub Actions deployment run                        FAILED
```

This immediately suggested that the problem was somewhere between GitHub and the live website.

### Possible causes considered

- GitHub Actions did not run.
- GitHub Actions could not authenticate to AWS.
- The S3 sync failed.
- The workflow referenced the wrong local path.
- The workflow used the wrong bucket.
- CloudFront invalidation failed.
- The workflow used the wrong CloudFront distribution.
- CloudFront cache still contained old content.
- Browser cache was stale.

### Important troubleshooting rule

Do **not** start changing S3 policies, CloudFront, DNS, or browser settings before finding the **first failing stage**.

If authentication fails, no later deployment step matters yet.

---

## 4. Root cause #1: invalid AWS credentials in GitHub Actions

The failing workflow showed an AWS authentication error equivalent to:

```text
The security token included in the request is invalid.
```

The existing workflow was inspected and contained:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: eu-west-2
```

### What this proved

The source code was not the problem. The pipeline could not obtain valid AWS credentials, so it could not reach the later commands:

```bash
aws s3 sync frontend/ s3://daviddigheji.com --delete
```

or:

```bash
aws cloudfront create-invalidation \
  --distribution-id E2ODEMEIX05YY8 \
  --paths "/*"
```

Therefore the website remained on the old version.

### Wrong recovery option

A quick but poor fix would have been:

```text
Create another IAM access key
→ store it in GitHub Secrets
→ make the workflow pass again
```

### Consequence of that wrong step

It restores the same architectural weakness that caused the incident.

### Correct decision

Replace the long-lived credential model with:

```text
GitHub Actions
      ↓
GitHub OIDC token
      ↓
AWS IAM trust policy
      ↓
Dedicated IAM role
      ↓
AWS STS temporary credentials
      ↓
S3 + CloudFront
```

### Memory hack

> **Humans → SSO. GitHub → OIDC. AWS → STS.**

---

# Part II — Verify the administrative identity

## 5. Authenticate to AWS through IAM Identity Center

The first command used was:

```bash
aws sso login --profile aegis-security
```

### Command explanation

- `aws` — starts the AWS Command Line Interface.
- `sso` — selects IAM Identity Center / SSO-related CLI functionality.
- `login` — starts an interactive SSO authentication session.
- `--profile aegis-security` — tells the AWS CLI which saved profile to authenticate.

The browser opened the Identity Center authorization flow and the CLI reported a successful login.

### Why this step came first

Before changing IAM, always prove **who you are** and **which access path you are using**.

Do not fix a broken CI/CD identity by falling back to another permanent administrator key.

---

## 6. Verify the current caller with STS

Command:

```bash
AWS_PAGER="" aws sts get-caller-identity \
  --profile aegis-security \
  --query 'Arn' \
  --output text
```

Actual result:

```text
arn:aws:sts::<AWS_ACCOUNT_ID>:assumed-role/AWSReservedSSO_Aegis-AdministratorAccess_f77b04e6d4d4156e/david.aegis
```

### Command explanation

- `AWS_PAGER=""` — disables the AWS CLI pager so output prints directly in the terminal. This is useful for scripting and evidence capture.
- `aws sts` — calls AWS Security Token Service.
- `get-caller-identity` — asks AWS to return the account and principal currently making the API request.
- `--profile aegis-security` — uses the Identity Center-backed profile.
- `--query 'Arn'` — uses a JMESPath query to return only the ARN field.
- `--output text` — prints a simple text result instead of JSON.

### What the result proved

The ARN contained:

```text
assumed-role/AWSReservedSSO_
```

That proved the administrator was using temporary SSO/STS role credentials rather than an IAM-user access key.

### Memory hack

> **Before changing AWS: WHO AM I?**

Use:

```bash
aws sts get-caller-identity
```

---

# Part III — Inspect existing GitHub OIDC configuration

## 7. List OpenID Connect providers

Command:

```bash
AWS_PAGER="" aws iam list-open-id-connect-providers \
  --profile aegis-security \
  --output table
```

Actual important result:

```text
arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

### Command explanation

- `aws iam` — selects AWS Identity and Access Management.
- `list-open-id-connect-providers` — lists OIDC identity providers registered in the AWS account.
- `--profile aegis-security` — performs the request using the secure SSO administrative profile.
- `--output table` — renders readable tabular output.

### What this proved

The GitHub OIDC provider already existed.

### Correct decision

Do **not** create a duplicate provider.

### Production lesson

> **Inspect before creating.**

Duplicate IAM identity objects make trust relationships harder to understand and audit.

---

## 8. Save the GitHub OIDC provider ARN into a shell variable

Command:

```bash
OIDC_ARN=$(AWS_PAGER="" aws iam list-open-id-connect-providers \
  --profile aegis-security \
  --query 'OpenIDConnectProviderList[?contains(Arn, `token.actions.githubusercontent.com`)].Arn | [0]' \
  --output text)

echo "$OIDC_ARN"
```

### Command explanation

#### `OIDC_ARN=$(...)`

`$(...)` is shell **command substitution**.

It means:

> Run the command inside the parentheses and store its output in the variable `OIDC_ARN`.

#### `--query ...`

The JMESPath query:

```text
OpenIDConnectProviderList[?contains(Arn, `token.actions.githubusercontent.com`)].Arn | [0]
```

means:

1. inspect `OpenIDConnectProviderList`;
2. keep entries whose ARN contains `token.actions.githubusercontent.com`;
3. return their ARN values;
4. select the first matching ARN.

#### `echo "$OIDC_ARN"`

Prints the variable so we can verify what was captured.

### Memory hack

> **`$()` = run it and remember the answer.**

---

## 9. Inspect the existing OIDC provider

Command:

```bash
AWS_PAGER="" aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn "$OIDC_ARN" \
  --profile aegis-security
```

Actual result included:

```json
{
  "Url": "token.actions.githubusercontent.com",
  "ClientIDList": [
    "sts.amazonaws.com"
  ]
}
```

### Command explanation

- `get-open-id-connect-provider` — retrieves the configuration of one registered OIDC provider.
- `--open-id-connect-provider-arn "$OIDC_ARN"` — supplies the provider ARN saved earlier.

### What this proved

AWS was configured to trust GitHub's OIDC identity service with the AWS STS audience:

```text
token.actions.githubusercontent.com
              ↓
sts.amazonaws.com
```

This was the prerequisite for using `sts:AssumeRoleWithWebIdentity` from GitHub Actions.

---

# Part IV — Identify the correct website resources

## 10. Identify the correct CloudFront distribution

Command:

```bash
AWS_PAGER="" aws cloudfront list-distributions \
  --profile aegis-security \
  --query 'DistributionList.Items[].{ID:Id,Domain:DomainName,Aliases:Aliases.Items,Status:Status}' \
  --output table
```

Actual relevant result:

```text
Alias:   daviddigheji.com
Domain:  d3rsd843gegld1.cloudfront.net
ID:      E2ODEMEIX05YY8
Status:  Deployed
```

A second distribution also existed:

```text
ID: E1KTEDUKYYS266
Alias: None
```

### Command explanation

- `aws cloudfront` — selects the CloudFront service.
- `list-distributions` — lists CloudFront distributions in the account.
- `--query 'DistributionList.Items[]...'` — reduces the large response to only useful fields.
- `{ID:Id,Domain:DomainName,Aliases:Aliases.Items,Status:Status}` — creates readable output labels.

### What this proved

The correct distribution for the public website was:

```text
E2ODEMEIX05YY8
```

### Wrong step

Do not guess the distribution ID from memory, especially when multiple distributions exist.

### Consequence

Invalidating or modifying the wrong distribution can waste time or affect an unrelated application.

### Memory hack

> **Resource ID + alias must agree.**

---

# Part V — Create a dedicated GitHub deployment role

## 11. Check whether the role already exists

Command:

```bash
AWS_PAGER="" aws iam get-role \
  --role-name CloudResume-GitHub-DeployRole \
  --profile aegis-security \
  --query 'Role.Arn' \
  --output text
```

Actual result:

```text
NoSuchEntity
The role with name CloudResume-GitHub-DeployRole cannot be found.
```

### Command explanation

- `get-role` — retrieves one IAM role by name.
- `--role-name CloudResume-GitHub-DeployRole` — names the role we want to check.
- `--query 'Role.Arn'` — returns only the role ARN if it exists.

### What `NoSuchEntity` meant

This was **not** an unexpected failure. It simply proved that the role did not exist yet.

### Production lesson

Always check before creating an IAM resource. A pre-existing role might already have a trust policy or consumers that must be understood first.

---

## 12. Capture the AWS account ID

Command:

```bash
ACCOUNT_ID=$(AWS_PAGER="" aws sts get-caller-identity \
  --profile aegis-security \
  --query Account \
  --output text)

echo "$ACCOUNT_ID"
```

Actual result:

```text
<AWS_ACCOUNT_ID>
```

### Command explanation

- `--query Account` — extracts only the AWS account ID from the STS response.
- `ACCOUNT_ID=$(...)` — stores that value in a shell variable.

### Why this was safer than repeatedly typing the account ID

It reduces transcription errors and makes later JSON generation repeatable.

### Important lesson

AWS account IDs are **identifiers**, not values you calculate with. Treat them as strings in application/configuration contexts.

---

## 13. Build the GitHub OIDC trust policy

Command used to create the temporary JSON file:

```bash
cat > /tmp/cloudresume-github-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GitHubOIDCTrust",
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:daviddigheji/aws-cloud-resume-challenge:ref:refs/heads/main"
        }
      }
    }
  ]
}
EOF

cat /tmp/cloudresume-github-trust.json
```

### Command explanation

#### `cat > file <<EOF`

This is a shell **here-document**.

It writes all text until the final `EOF` into the target file.

#### Why `<<EOF` was used rather than `<<'EOF'`

Unquoted `EOF` allows shell variable expansion.

Therefore:

```text
${ACCOUNT_ID}
```

was replaced with:

```text
<AWS_ACCOUNT_ID>
```

If `<<'EOF'` had been used, the literal text `${ACCOUNT_ID}` would have remained in the file.

### Memory hack

```text
<<EOF      = variables can expand
<<'EOF'    = keep content literal
```

### Trust policy explanation

#### Principal

```json
"Principal": {
  "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
}
```

This says the trusted external identity source is the registered GitHub OIDC provider.

#### Action

```json
"Action": "sts:AssumeRoleWithWebIdentity"
```

This allows a trusted OIDC identity to exchange its web identity token for temporary AWS role credentials.

#### Audience condition

```json
"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
```

The token must be intended for AWS STS.

#### Subject condition

```json
"token.actions.githubusercontent.com:sub": "repo:daviddigheji/aws-cloud-resume-challenge:ref:refs/heads/main"
```

The token must originate from:

```text
Repository: daviddigheji/aws-cloud-resume-challenge
Branch:     main
```

### Security consequence of omitting the `sub` restriction

A much broader set of GitHub workflows could potentially attempt to assume the role, depending on the trust configuration.

### Memory hack

> **Trust policy = WHO may become the role.**

---

## 14. Create the deployment role

Command:

```bash
AWS_PAGER="" aws iam create-role \
  --role-name CloudResume-GitHub-DeployRole \
  --assume-role-policy-document file:///tmp/cloudresume-github-trust.json \
  --description "OIDC deployment role for daviddigheji.com Cloud Resume" \
  --max-session-duration 3600 \
  --profile aegis-security
```

### Command explanation

- `create-role` — creates a new IAM role.
- `--role-name` — sets the role's friendly name.
- `--assume-role-policy-document` — defines the trust policy (who may assume the role).
- `file:///tmp/...json` — tells the AWS CLI to load JSON from a local file.
- `--description` — documents the purpose of the role.
- `--max-session-duration 3600` — caps the maximum role session at 3600 seconds (one hour).
- `--profile aegis-security` — creates the role using the secure SSO administrator.

### Why a role rather than a new IAM user

A role:

- does not need a permanent password;
- does not need a permanent access key;
- is assumed only when the trust policy is satisfied;
- receives short-lived credentials from STS.

---

## 15. Verify the role

Command:

```bash
AWS_PAGER="" aws iam get-role \
  --role-name CloudResume-GitHub-DeployRole \
  --profile aegis-security \
  --query 'Role.{RoleName:RoleName,Arn:Arn,MaxSessionDuration:MaxSessionDuration}' \
  --output table
```

Actual result confirmed:

```text
RoleName:           CloudResume-GitHub-DeployRole
Arn:                arn:aws:iam::<AWS_ACCOUNT_ID>:role/CloudResume-GitHub-DeployRole
MaxSessionDuration: 3600
```

### Command explanation

The query:

```text
Role.{RoleName:RoleName,Arn:Arn,MaxSessionDuration:MaxSessionDuration}
```

builds a compact object containing only the three fields needed for verification.

### Important IAM distinction

At this point the role had a **trust policy**, but no deployment permissions yet.

> **Trust = WHO can become the role.**  
> **Permissions = WHAT the role can do after it has been assumed.**

---

# Part VI — Apply least-privilege deployment permissions

## 16. Create the deployment permission policy JSON

The actual policy used was:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListWebsiteBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::daviddigheji.com"
    },
    {
      "Sid": "DeployWebsiteObjects",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::daviddigheji.com/*"
    },
    {
      "Sid": "InvalidateWebsiteCloudFront",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::<AWS_ACCOUNT_ID>:distribution/E2ODEMEIX05YY8"
    }
  ]
}
```

### What each permission does

#### `s3:ListBucket`

Allows the workflow to list objects in the website bucket so the sync process can compare source and destination.

#### `s3:GetBucketLocation`

Allows the AWS CLI to determine the bucket's region/location.

#### `s3:PutObject`

Allows upload of new or changed website files.

#### `s3:DeleteObject`

Required because the workflow uses:

```bash
--delete
```

Without `DeleteObject`, files removed from Git could remain in S3.

#### `cloudfront:CreateInvalidation`

Allows the workflow to ask CloudFront to expire cached paths after deployment.

### What we deliberately did not grant

We did not grant:

```json
"Action": "*",
"Resource": "*"
```

and we did not attach `AdministratorAccess`.

The deployment role cannot intentionally be used to administer unrelated AWS services.

### Memory hack

> **CI/CD should deploy the application, not administer the AWS account.**

### If a future S3 sync returns an access error

Do not immediately broaden the policy to `s3:*`.

Read the exact denied API action. If AWS reports that object metadata/read access is required, add only the necessary action (for example `s3:GetObject`) to the website-object ARN and retest.

---

## 17. Attach the policy as an inline role policy

Command:

```bash
AWS_PAGER="" aws iam put-role-policy \
  --role-name CloudResume-GitHub-DeployRole \
  --policy-name CloudResumeWebsiteDeployPolicy \
  --policy-document file:///tmp/cloudresume-deploy-policy.json \
  --profile aegis-security
```

### Command explanation

- `put-role-policy` — creates or updates an **inline policy** embedded in one role.
- `--role-name` — identifies the role receiving the policy.
- `--policy-name` — gives the inline policy a descriptive name.
- `--policy-document` — supplies the permission JSON.

### Why an inline policy was reasonable here

The policy existed for one dedicated deployment role and was not being reused across multiple roles.

If the same permissions needed to be shared by several roles, a customer-managed policy would normally be easier to reuse and govern.

---

## 18. Verify that the policy exists

Command:

```bash
AWS_PAGER="" aws iam list-role-policies \
  --role-name CloudResume-GitHub-DeployRole \
  --profile aegis-security \
  --output table
```

Actual result:

```text
CloudResumeWebsiteDeployPolicy
```

### Command explanation

`list-role-policies` lists **inline** policies on a role.

It does not list attached AWS-managed or customer-managed policies; those use a different API (`list-attached-role-policies`).

---

## 19. Read the actual role policy back from AWS

Command:

```bash
AWS_PAGER="" aws iam get-role-policy \
  --role-name CloudResume-GitHub-DeployRole \
  --policy-name CloudResumeWebsiteDeployPolicy \
  --profile aegis-security
```

### Why this verification matters

Do not rely only on the local JSON file. Reading the policy back from IAM proves what AWS actually stored.

### Memory hack

> **Create locally → apply → read back from AWS.**

---

# Part VII — Local path troubleshooting lesson

## 20. The failed manual `aws s3 sync` command

While the shell prompt showed the current directory was already `frontend`, this command was run:

```bash
aws s3 sync frontend/ s3://daviddigheji.com --delete
```

The AWS CLI returned:

```text
The user-provided path frontend/ does not exist.
```

### Why this happened

The current working directory was already:

```text
.../aws-cloud-resume-challenge/frontend
```

Therefore `frontend/` referred to:

```text
.../frontend/frontend/
```

which did not exist.

### Correct manual source path from inside `frontend/`

```bash
aws s3 sync ./ s3://daviddigheji.com --delete
```

### But why we did not use manual deployment as the final fix

The real objective was to repair GitHub Actions as the authoritative CI/CD path.

A manual upload could make the website look fixed while leaving the deployment pipeline broken.

### Memory hack

> **Path error? Check `pwd` before checking IAM.**

---

# Part VIII — Inspect and update the GitHub Actions workflow

## 21. Find the repository root safely

Command:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
echo "$REPO_ROOT"
```

Actual result:

```text
/Users/david/projects/aws-cloud-resume-challenge
```

### Command explanation

- `git rev-parse --show-toplevel` — asks Git for the absolute path of the repository root.
- `REPO_ROOT=$(...)` — stores that path in a variable.

### Why this was useful

It made later commands work regardless of whether the terminal was currently inside `frontend/`, `.github/`, or another subdirectory.

### Memory hack

> **`git rev-parse --show-toplevel` = Where is the root of this repo?**

---

## 22. List the workflow files

Command:

```bash
ls -l "$REPO_ROOT/.github/workflows/"
```

### Command explanation

- `ls` — lists directory contents.
- `-l` — uses long format, showing details such as permissions, owner, size, and modification time.

The output confirmed `deploy.yml` existed.

---

## 23. Inspect the current workflow

Command:

```bash
cat "$REPO_ROOT/.github/workflows/deploy.yml"
```

The original workflow was:

```yaml
name: Deploy Cloud Resume Website

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-2

      - name: Sync frontend files to S3
        run: aws s3 sync frontend/ s3://daviddigheji.com --delete

      - name: Invalidate CloudFront cache
        run: aws cloudfront create-invalidation --distribution-id E2ODEMEIX05YY8 --paths "/*"
```

### What this proved

The runner was definitely configured to use long-lived AWS secrets.

---

## 24. Replace the workflow with OIDC authentication

The workflow was rewritten to this production-style structure:

```yaml
name: Deploy Cloud Resume Website

on:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7.0.1

      - name: Configure AWS credentials with OIDC
        uses: aws-actions/configure-aws-credentials@v6.2.3
        with:
          role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/CloudResume-GitHub-DeployRole
          aws-region: eu-west-2
          role-session-name: cloudresume-${{ github.run_id }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Sync frontend files to S3
        run: aws s3 sync frontend/ s3://daviddigheji.com --delete

      - name: Invalidate CloudFront cache
        run: aws cloudfront create-invalidation --distribution-id E2ODEMEIX05YY8 --paths "/*"
```

### Explanation of the important workflow lines

#### `permissions: contents: read`

Allows workflow steps such as repository checkout to read the repository contents.

#### `permissions: id-token: write`

Allows GitHub Actions to request an OIDC identity token.

This **does not itself grant AWS permissions**. AWS still evaluates the OIDC provider, IAM role trust policy, and role permission policy.

#### `role-to-assume`

Specifies the exact AWS IAM role GitHub should request.

#### `aws-region: eu-west-2`

Sets the default AWS region for the action's credentials/session.

#### `role-session-name: cloudresume-${{ github.run_id }}`

Creates an STS session name tied to the GitHub Actions run ID.

This improves traceability in AWS logs.

Example conceptual session name:

```text
cloudresume-33361119693
```

#### `aws sts get-caller-identity`

Adds a deliberate identity-verification checkpoint inside the deployment pipeline.

Expected role pattern:

```text
assumed-role/CloudResume-GitHub-DeployRole/...
```

### Interview value

> “I used the GitHub run ID as the STS role-session name so a CI/CD deployment can be correlated with AWS audit logs.”

---

# Part IX — Secondary troubleshooting issue

## 25. Root cause #2: optional account-ID validation caused another workflow failure

The first OIDC workflow also included:

```yaml
allowed-account-ids: "<AWS_ACCOUNT_ID>"
```

The workflow failed with an account-ID validation error where the expected value was effectively reported without the leading zeros.

### Initial assumption

The first theory was that YAML had treated the ID as a numeric value and stripped a leading zero.

### Evidence corrected that assumption

Git later showed there was no local change to commit after “quoting” the account ID:

```text
nothing to commit, working tree clean
Everything up-to-date
```

This proved the value was already quoted in the committed workflow.

### Production lesson

> **Do not defend the first hypothesis. Follow the evidence.**

The optional `allowed-account-ids` guardrail was then removed, while retaining stronger controls already present:

- exact `role-to-assume` ARN;
- repository-restricted OIDC `sub` condition;
- `main` branch restriction;
- STS audience restriction;
- least-privilege role permissions.

---

## 26. Remove `allowed-account-ids` with `sed`

Command:

```bash
sed -i '' '/allowed-account-ids:/d' .github/workflows/deploy.yml
```

### Command explanation

- `sed` — stream editor used to transform text.
- `-i ''` — macOS syntax for editing the file **in place** without creating a backup suffix.
- `/allowed-account-ids:/` — matches a line containing that text.
- `d` — deletes the matching line.

### Verify removal

```bash
grep -n "allowed-account-ids" .github/workflows/deploy.yml
```

### Command explanation

- `grep` — searches text for a pattern.
- `-n` — prints matching line numbers.

Expected result after deletion:

```text
(no output)
```

No output was correct because the line no longer existed.

---

## 27. Inspect the exact Git change

Command:

```bash
git diff -- .github/workflows/deploy.yml
```

### Command explanation

- `git diff` — shows unstaged changes compared with the current commit.
- `--` — separates Git options from the file path.
- `.github/workflows/deploy.yml` — limits output to that file.

Expected change:

```diff
-          allowed-account-ids: "<AWS_ACCOUNT_ID>"
```

### Why this mattered

Before committing, we verified that only the intended line had changed.

---

# Part X — Safe Git commit and deployment trigger

## 28. Stage only the workflow file

Command:

```bash
git add .github/workflows/deploy.yml
```

### Why not `git add .`?

`git add .` stages all changes under the current directory.

In a repository containing unrelated modifications, evidence files, or accidental local changes, that can create an unsafe or noisy commit.

### Production rule

> **Stage deliberately.**

---

## 29. Verify what is staged

Command:

```bash
git diff --cached --name-only
```

### Command explanation

- `git diff` — compares Git states.
- `--cached` — compares the staging area (index) with the last commit.
- `--name-only` — shows only filenames rather than full diffs.

Expected:

```text
.github/workflows/deploy.yml
```

### Memory hack

> **Inspect → Stage → Inspect again → Commit.**

---

## 30. Commit the correction

Command:

```bash
git commit -m "Fix GitHub OIDC account validation"
```

Actual result:

```text
[main 752d6ee] Fix GitHub OIDC account validation
1 file changed, 1 deletion(-)
```

### Command explanation

- `git commit` — records the staged snapshot in Git history.
- `-m` — supplies the commit message directly on the command line.

### Why the commit message was good

It describes the operational/security change rather than a vague message such as `fix stuff`.

---

## 31. Push to GitHub

Command:

```bash
git push origin main
```

Actual successful output included:

```text
ee3ca8d..752d6ee  main -> main
```

### Command explanation

- `git push` — sends local commits to a remote repository.
- `origin` — the conventional name of the primary remote.
- `main` — the branch being pushed.

### What this accomplished

Because the workflow contains:

```yaml
on:
  push:
    branches:
      - main
```

a successful push to `main` automatically triggers GitHub Actions.

No separate manual “deploy” command is required.

---

## 32. Verify local and remote branch synchronization

Command:

```bash
git status -sb
```

Expected healthy result:

```text
## main...origin/main
```

### Command explanation

- `git status` — shows working tree and branch state.
- `-s` — short output.
- `-b` — includes branch/tracking information.

If output says `[ahead 1]`, the local branch contains a commit not yet pushed.

If it says `[behind 1]`, the remote contains a commit not yet present locally.

---

# Part XI — Final deployment verification procedure

## 33. GitHub Actions checks

After the push, open:

```text
GitHub
→ aws-cloud-resume-challenge
→ Actions
→ Deploy Cloud Resume Website
→ newest run
```

The target run should show:

```text
Checkout repository                  PASS
Configure AWS credentials with OIDC  PASS
Verify AWS identity                  PASS
Sync frontend files to S3            PASS
Invalidate CloudFront cache          PASS
```

### If “Configure AWS credentials with OIDC” fails

Inspect:

- the role ARN;
- GitHub repository name;
- branch name;
- OIDC provider;
- IAM trust `aud` condition;
- IAM trust `sub` condition.

Do not add static credentials as a workaround.

---

## 34. Verify the GitHub runner identity

The `Verify AWS identity` step runs:

```bash
aws sts get-caller-identity
```

Expected ARN pattern:

```text
arn:aws:sts::<AWS_ACCOUNT_ID>:assumed-role/CloudResume-GitHub-DeployRole/cloudresume-...
```

### What this proves

The runner is using:

```text
GitHub OIDC
→ CloudResume-GitHub-DeployRole
→ STS temporary credentials
```

and not an IAM access key.

---

## 35. Verify S3 received the current site

After a green workflow, verify key objects directly from the administrator CLI.

Example:

```bash
AWS_PAGER="" aws s3api head-object \
  --bucket daviddigheji.com \
  --key index.html \
  --profile aegis-security
```

### Command explanation

- `aws s3api` — uses the low-level S3 API commands.
- `head-object` — returns metadata for an S3 object without downloading the entire body.
- `--bucket` — specifies the bucket.
- `--key index.html` — specifies the object key.

Check fields such as `LastModified`, `ContentLength`, and `ETag`.

### Optional content comparison

To download the deployed `index.html` for comparison:

```bash
AWS_PAGER="" aws s3 cp \
  s3://daviddigheji.com/index.html \
  /tmp/live-index.html \
  --profile aegis-security
```

Then compare:

```bash
diff frontend/index.html /tmp/live-index.html
```

#### `diff` explanation

`diff` compares two text files line by line.

No output means they are identical.

---

## 36. Verify CloudFront invalidation

List recent invalidations:

```bash
AWS_PAGER="" aws cloudfront list-invalidations \
  --distribution-id E2ODEMEIX05YY8 \
  --profile aegis-security \
  --query 'InvalidationList.Items[0:5].[Id,Status,CreateTime]' \
  --output table
```

### Command explanation

- `list-invalidations` — returns CloudFront cache invalidation requests.
- `--distribution-id` — restricts the query to the website distribution.
- `[0:5]` — returns the first five items from the result list.

Expected final status:

```text
Completed
```

If the newest invalidation says `InProgress`, wait and query again before diagnosing the website as stale.

---

## 37. Verify the live website

After S3 and CloudFront are verified, perform a hard browser refresh.

On macOS Firefox:

```text
Command + Shift + R
```

You can also test headers from a terminal:

```bash
curl -I https://daviddigheji.com
```

### Command explanation

- `curl` — transfers data to/from URLs.
- `-I` — requests only HTTP response headers.

This can confirm the domain is responding through HTTP/HTTPS, although it does not by itself prove the page body is the newest version.

To search the live HTML for a known new phrase:

```bash
curl -s https://daviddigheji.com | grep -F "<KNOWN NEW TEXT>"
```

- `-s` — silent mode, suppressing progress output.
- `grep -F` — performs a fixed-string search rather than a regular-expression search.

---

# Part XII — Failure decision tree

## 38. Symptom: GitHub has new files, live site is old

Follow this sequence:

```text
1. Did GitHub Actions run?
        ↓ YES
2. Did OIDC authentication pass?
        ↓ YES
3. Did STS show CloudResume-GitHub-DeployRole?
        ↓ YES
4. Did S3 sync pass?
        ↓ YES
5. Does S3 contain the new index.html?
        ↓ YES
6. Did CloudFront invalidation complete?
        ↓ YES
7. Does the live URL still show old content?
        ↓ YES
8. Check CloudFront origin/behavior and browser cache.
```

### Rule

Stop at the **first NO** and troubleshoot that layer.

Do not skip directly to layer 8.

---

## 39. Symptom: `The security token included in the request is invalid`

Likely areas:

- old/disabled access key;
- invalid session credentials;
- expired temporary token;
- workflow still using static GitHub secrets.

For this project, the correct long-term remediation is OIDC, not another permanent key.

---

## 40. Symptom: `NoSuchEntity` from `get-role`

Interpretation:

The requested IAM role does not exist.

Correct response:

- if you expected it not to exist: proceed with controlled creation;
- if you expected it to exist: verify spelling, AWS account, and profile before creating anything.

---

## 41. Symptom: local path does not exist

Example:

```text
The user-provided path frontend/ does not exist.
```

Check:

```bash
pwd
ls -la
```

### `pwd`

Prints the current working directory.

### `ls -la`

- `-l` — long format;
- `-a` — includes hidden files/directories.

Do not change IAM permissions to fix a local filesystem path error.

---

## 42. Symptom: S3 `AccessDenied`

Read the exact denied action.

Check:

```bash
AWS_PAGER="" aws iam get-role-policy \
  --role-name CloudResume-GitHub-DeployRole \
  --policy-name CloudResumeWebsiteDeployPolicy \
  --profile aegis-security
```

Then add only the required action/resource if the workflow genuinely needs it.

### Wrong fix

```json
"Action": "s3:*",
"Resource": "*"
```

### Consequence

The deployment role gains access far beyond the website bucket.

---

## 43. Symptom: CloudFront invalidation fails

Check:

```text
Distribution ID = E2ODEMEIX05YY8
Permission       = cloudfront:CreateInvalidation
Role             = CloudResume-GitHub-DeployRole
```

Do not grant `cloudfront:*` unless there is a documented requirement.

---

## 44. Symptom: workflow succeeds but site is still stale

Verify in this exact order:

1. S3 object `LastModified`.
2. S3 object contents.
3. CloudFront invalidation exists.
4. Invalidation status is `Completed`.
5. CloudFront distribution alias is `daviddigheji.com`.
6. CloudFront origin points to the intended S3 origin.
7. Browser hard refresh / private browsing test.

### Why DNS is not first

If `daviddigheji.com` resolves and serves the old website, DNS is already taking the user somewhere. The first question is whether the content pipeline updated the origin/cache.

---

# Part XIII — Security design summary

## 45. Before

```text
GitHub Actions
      ↓
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
      ↓
Long-lived IAM credentials
      ↓
S3 / CloudFront
```

## 46. After

```text
GitHub Actions
      ↓
OIDC identity token
      ↓
AWS IAM trust policy
      ↓
CloudResume-GitHub-DeployRole
      ↓
Temporary STS credentials
      ↓
Least-privilege S3 + CloudFront access
```

### Improvements

- no permanent AWS deployment key stored in GitHub;
- trust restricted to one repository and `main` branch;
- AWS session is temporary;
- deployment permissions are resource-specific;
- STS identity is verified in the pipeline;
- GitHub run ID can be correlated to the AWS role session;
- failures are easier to isolate and audit.

---

# Part XIV — Wrong steps and consequences

## 47. Wrong: create another long-lived AWS access key

**Consequence:** repeats the original security weakness and creates another credential-rotation dependency.

**Better:** OIDC + STS role assumption.

---

## 48. Wrong: attach AdministratorAccess to the GitHub role

**Consequence:** a compromised workflow could perform broad AWS administration.

**Better:** allow only required S3 actions and `cloudfront:CreateInvalidation` on the website resources.

---

## 49. Wrong: make the S3 bucket public because the website is stale

**Consequence:** exposes the origin and bypasses the intended CloudFront/OAC security model.

**Better:** troubleshoot GitHub Actions, S3 deployment, and CloudFront invalidation independently.

---

## 50. Wrong: archive or ignore an error without finding its source

**Consequence:** the website may appear temporarily fixed while the deployment system remains broken.

**Better:** find the first failing stage and validate each downstream layer.

---

## 51. Wrong: change several components at once

**Consequence:** you lose causal evidence and cannot confidently say what fixed the incident.

**Better:** one controlled change followed by verification.

---

## 52. Wrong: run `git add .` without checking the working tree

**Consequence:** unrelated files or sensitive data could be staged accidentally.

**Better:**

```bash
git status --short
git add .github/workflows/deploy.yml
git diff --cached --name-only
```

---

# Part XV — Rebuild procedure from scratch

## 53. Clean production-style rebuild sequence

If recreating this deployment from a fresh AWS/GitHub setup, follow this order:

### Phase A — Confirm prerequisites

```text
[ ] GitHub repository exists.
[ ] Website files are under frontend/.
[ ] S3 website/origin bucket exists.
[ ] CloudFront distribution exists and serves the website.
[ ] Administrator uses IAM Identity Center / temporary credentials.
```

### Phase B — Register GitHub OIDC provider (only if absent)

First check:

```bash
AWS_PAGER="" aws iam list-open-id-connect-providers \
  --profile aegis-security
```

If `token.actions.githubusercontent.com` already exists, reuse it.

### Phase C — Create a dedicated trust policy

Restrict:

```text
aud = sts.amazonaws.com
sub = repo:<owner>/<repo>:ref:refs/heads/main
```

### Phase D — Create the role

```bash
AWS_PAGER="" aws iam create-role \
  --role-name CloudResume-GitHub-DeployRole \
  --assume-role-policy-document file:///tmp/cloudresume-github-trust.json \
  --description "OIDC deployment role for daviddigheji.com Cloud Resume" \
  --max-session-duration 3600 \
  --profile aegis-security
```

### Phase E — Add least-privilege deployment permissions

Grant only:

```text
S3 bucket listing/location
S3 website object put/delete
CloudFront CreateInvalidation for the website distribution
```

### Phase F — Configure GitHub workflow permissions

```yaml
permissions:
  contents: read
  id-token: write
```

### Phase G — Assume the dedicated role

```yaml
with:
  role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/CloudResume-GitHub-DeployRole
  aws-region: eu-west-2
  role-session-name: cloudresume-${{ github.run_id }}
```

### Phase H — Verify before deployment

```yaml
- name: Verify AWS identity
  run: aws sts get-caller-identity
```

### Phase I — Deploy and invalidate

```bash
aws s3 sync frontend/ s3://daviddigheji.com --delete
aws cloudfront create-invalidation --distribution-id E2ODEMEIX05YY8 --paths "/*"
```

### Phase J — Verify the live result

Check GitHub run → STS ARN → S3 object → CloudFront invalidation → website.

---

# Part XVI — Command reference sheet

## 54. Authentication commands

### Login with IAM Identity Center

```bash
aws sso login --profile aegis-security
```

**Purpose:** obtain a temporary administrative session.

### Verify identity

```bash
AWS_PAGER="" aws sts get-caller-identity \
  --profile aegis-security
```

**Purpose:** prove account and principal before making changes.

---

## 55. OIDC commands

### List providers

```bash
AWS_PAGER="" aws iam list-open-id-connect-providers \
  --profile aegis-security \
  --output table
```

### Get one provider

```bash
AWS_PAGER="" aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn "$OIDC_ARN" \
  --profile aegis-security
```

---

## 56. IAM role commands

### Check role

```bash
AWS_PAGER="" aws iam get-role \
  --role-name CloudResume-GitHub-DeployRole \
  --profile aegis-security
```

### Create role

```bash
AWS_PAGER="" aws iam create-role \
  --role-name CloudResume-GitHub-DeployRole \
  --assume-role-policy-document file:///tmp/cloudresume-github-trust.json \
  --description "OIDC deployment role for daviddigheji.com Cloud Resume" \
  --max-session-duration 3600 \
  --profile aegis-security
```

### Add inline policy

```bash
AWS_PAGER="" aws iam put-role-policy \
  --role-name CloudResume-GitHub-DeployRole \
  --policy-name CloudResumeWebsiteDeployPolicy \
  --policy-document file:///tmp/cloudresume-deploy-policy.json \
  --profile aegis-security
```

### List inline role policies

```bash
AWS_PAGER="" aws iam list-role-policies \
  --role-name CloudResume-GitHub-DeployRole \
  --profile aegis-security
```

### Read inline role policy

```bash
AWS_PAGER="" aws iam get-role-policy \
  --role-name CloudResume-GitHub-DeployRole \
  --policy-name CloudResumeWebsiteDeployPolicy \
  --profile aegis-security
```

---

## 57. CloudFront commands

### List distributions

```bash
AWS_PAGER="" aws cloudfront list-distributions \
  --profile aegis-security \
  --query 'DistributionList.Items[].{ID:Id,Domain:DomainName,Aliases:Aliases.Items,Status:Status}' \
  --output table
```

### Create invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id E2ODEMEIX05YY8 \
  --paths "/*"
```

### List invalidations

```bash
AWS_PAGER="" aws cloudfront list-invalidations \
  --distribution-id E2ODEMEIX05YY8 \
  --profile aegis-security \
  --output table
```

---

## 58. S3 commands

### Sync site from repository root

```bash
aws s3 sync frontend/ s3://daviddigheji.com --delete
```

### Sync site if already inside `frontend/`

```bash
aws s3 sync ./ s3://daviddigheji.com --delete
```

### Inspect deployed object metadata

```bash
AWS_PAGER="" aws s3api head-object \
  --bucket daviddigheji.com \
  --key index.html \
  --profile aegis-security
```

---

## 59. Git commands

### Find repository root

```bash
git rev-parse --show-toplevel
```

### Show short working-tree status

```bash
git status --short
```

### Show branch tracking state

```bash
git status -sb
```

### Inspect one file's changes

```bash
git diff -- .github/workflows/deploy.yml
```

### Stage only the workflow

```bash
git add .github/workflows/deploy.yml
```

### Verify staged filenames

```bash
git diff --cached --name-only
```

### Commit

```bash
git commit -m "Fix GitHub OIDC account validation"
```

### Push

```bash
git push origin main
```

---

# Part XVII — Interview preparation

## 60. “Tell me about a production-like incident you troubleshot.”

### Strong answer

I had a Cloud Resume website where the latest source files were present in GitHub but the live CloudFront site continued serving the previous version. I traced the deployment path from GitHub rather than changing S3 or DNS immediately and found that the GitHub Actions job was failing at AWS authentication because it still depended on invalid long-lived AWS credentials.

Instead of generating another access key, I migrated the deployment to GitHub OIDC. I verified the existing AWS OIDC provider, created a dedicated IAM role with a trust policy restricted to my repository and `main` branch, and granted only the S3 deployment and CloudFront invalidation permissions required by the pipeline. I also added STS caller-identity verification and a GitHub run-ID-based role session name for traceability.

During testing, I hit a second failure related to an optional account-ID validation setting. I checked the actual committed Git state, corrected my initial hypothesis, removed the unnecessary setting, committed only the workflow change, and pushed the new pipeline. The key improvement was that the deployment no longer depended on permanent AWS access keys.

---

## 61. “Why did you choose OIDC rather than replacing the access key?”

Because replacing the key would restore service but preserve the underlying weakness. OIDC allows GitHub to prove workload identity and obtain short-lived STS credentials only when the trusted workflow runs. That removes the requirement to store long-lived AWS credentials in GitHub.

---

## 62. “What is the difference between the IAM role trust policy and its permissions policy?”

The **trust policy** answers **WHO is allowed to assume the role**. In this case it trusts GitHub OIDC only for the specific repository and `main` branch.

The **permissions policy** answers **WHAT the assumed role can do**. In this case it grants only the S3 and CloudFront actions required to deploy the website.

### Memory hack

> **Trust = WHO. Permissions = WHAT.**

---

## 63. “How did you apply least privilege?”

I did not attach AdministratorAccess. The role can list the website bucket, upload/delete website objects, and create invalidations only for the CloudFront distribution serving `daviddigheji.com`. Unrelated AWS services and resources are outside the role's intended permission scope.

---

## 64. “How did you improve auditability?”

I added `aws sts get-caller-identity` as an explicit deployment step and used `cloudresume-${{ github.run_id }}` as the STS role-session name. That lets the AWS session be related back to a specific GitHub Actions run during log analysis.

---

## 65. “What did you learn from the account-ID validation troubleshooting?”

The first explanation was not fully correct. Git showed the account ID had already been quoted, so I revised the hypothesis based on evidence rather than continuing to make the same change. The lesson was that troubleshooting should be hypothesis-driven but evidence-controlled.

---

# Part XVIII — Memory map

## 66. Four-layer deployment memory hack

### SOURCE → IDENTITY → STORAGE → CACHE

```text
SOURCE
Is the new code really in GitHub?
        ↓
IDENTITY
Can GitHub obtain the intended AWS role?
        ↓
STORAGE
Did S3 receive the new files?
        ↓
CACHE
Did CloudFront invalidate and serve the new version?
```

If those four layers are checked in order, most failures in this website architecture can be isolated quickly.

---

## 67. Security memory hack

```text
Human administrator → IAM Identity Center → STS
GitHub workload      → OIDC                → STS
```

The common idea is:

> **Use identity federation and temporary credentials instead of permanent keys.**

---

# Part XIX — Evidence checklist

## 68. Recommended screenshots / outputs

For portfolio or incident evidence, capture:

```text
01-github-actions-original-authentication-failure.png
02-admin-sso-sts-identity.png
03-existing-github-oidc-provider.png
04-cloudfront-distribution-identification.png
05-github-deploy-role-trust-policy.png
06-github-deploy-role-permissions.png
07-workflow-oidc-configuration.png
08-github-actions-oidc-identity-verification.png
09-s3-sync-success.png
10-cloudfront-invalidation-success.png
11-live-website-updated.png
```

### Redact / never expose

- AWS secret access keys;
- session tokens;
- passwords;
- MFA QR codes/seeds;
- private keys;
- unrelated sensitive account data.

### What can usually remain visible when useful

- IAM role name;
- GitHub repository name;
- AWS service/resource names relevant to the architecture;
- successful STS assumed-role pattern;
- CloudFront distribution ID if there is no security reason to hide it.

---

# Part XX — Final incident record

## 69. Incident summary

```text
SYMPTOM
GitHub contained the latest website files, but daviddigheji.com displayed the older site.

FIRST ROOT CAUSE
GitHub Actions used invalid long-lived AWS access credentials.

SECURITY REMEDIATION
Migrated CI/CD authentication to GitHub OIDC + dedicated IAM role + temporary STS credentials.

TRUST RESTRICTIONS
Repository: daviddigheji/aws-cloud-resume-challenge
Branch: main
Audience: sts.amazonaws.com

DEPLOYMENT PERMISSIONS
S3 bucket listing/location
S3 website-object put/delete
CloudFront invalidation for E2ODEMEIX05YY8

SECOND TROUBLESHOOTING ISSUE
Optional allowed-account-ids validation caused another workflow failure.

CORRECTION
Removed the optional account-ID validation setting after Git evidence showed the quoting hypothesis did not explain the failure.

GIT COMMIT
752d6ee — Fix GitHub OIDC account validation

PUSH RESULT
ee3ca8d..752d6ee main -> main

FINAL VALIDATION STATUS IN THIS RUNBOOK
The remediation was successfully pushed. Record the final green GitHub Actions run, S3 verification, completed CloudFront invalidation, and live-site content check once confirmed.
```

---

## 70. Final engineering lesson

The objective was not simply to “make the website update again.”

The better outcome was to replace a fragile deployment authentication design with a more secure and auditable one:

```text
Broken long-lived access key
          ↓
Incident investigation
          ↓
OIDC workload identity
          ↓
Repository/branch-restricted trust
          ↓
Dedicated least-privilege IAM role
          ↓
Temporary STS credentials
          ↓
Verified S3 deployment
          ↓
Verified CloudFront invalidation
          ↓
Live website
```

That is the incident story to remember.
