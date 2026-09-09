# Website Deployment Runbook

**Project:** AWS Cloud Resume Challenge  
**Website:** `https://daviddigheji.com`  
**Repository:** `aws-cloud-resume-challenge`  
**Runbook type:** Reusable operational procedure

---

## 1. Purpose

Use this runbook to deploy the Cloud Resume website safely and verify the complete delivery path:

```text
GitHub source
    ↓
GitHub Actions
    ↓
AWS identity
    ↓
S3 origin
    ↓
CloudFront
    ↓
daviddigheji.com
```

The objective is not merely to make a deployment succeed. The objective is to prove each layer independently.

---

## 2. Preconditions

Before deploying, verify:

```text
[ ] Git repository is clean or expected changes are understood
[ ] Correct branch is checked out
[ ] GitHub workflow exists
[ ] GitHub OIDC trust is configured
[ ] Deployment role exists
[ ] S3 website/origin bucket exists
[ ] CloudFront distribution serving daviddigheji.com exists
[ ] Administrator CLI authentication uses IAM Identity Center
```

---

## 3. Authenticate the administrator

Run:

```bash
aws sso login --profile aegis-security
```

### Explanation

- `aws` — starts the AWS CLI.
- `sso` — selects IAM Identity Center / SSO functionality.
- `login` — starts an interactive authentication session.
- `--profile aegis-security` — uses the saved `aegis-security` CLI profile.

### Why

Do not make infrastructure changes before proving that the administrative session uses the intended federated profile.

### Verify

```bash
AWS_PAGER="" aws sts get-caller-identity \
  --profile aegis-security
```

`sts` means **AWS Security Token Service**.

`get-caller-identity` returns the AWS account and role/session currently making the request.

`AWS_PAGER=""` disables the AWS CLI pager so output appears directly in the terminal.

Expected pattern:

```text
arn:aws:sts::<ACCOUNT>:assumed-role/<ROLE>/<SESSION>
```

---

## 4. Verify repository state

From the repository root:

```bash
git status -sb
```

### Explanation

- `git status` — shows repository state.
- `-s` — short format.
- `-b` — includes branch/tracking information.

Expected:

```text
## main...origin/main
```

If files are modified, understand them before deployment.

Check the repository root:

```bash
git rev-parse --show-toplevel
```

This prevents running deployment commands from the wrong local directory.

---

## 5. Verify website source path

List the expected website directory:

```bash
ls -la frontend/
```

### Explanation

- `ls` — lists files.
- `-l` — long format.
- `-a` — includes hidden entries.

If the project layout changes, update the workflow rather than blindly reusing an old path.

---

## 6. Verify the GitHub Actions workflow

List workflow files:

```bash
ls -la .github/workflows/
```

Inspect the deployment workflow:

```bash
sed -n '1,240p' .github/workflows/deploy.yml
```

### Explanation

- `sed` — stream editor.
- `-n` — suppresses automatic printing.
- `'1,240p'` — prints lines 1 through 240.

The workflow should use OIDC, not permanent AWS access keys.

Required workflow permissions:

```yaml
permissions:
  contents: read
  id-token: write
```

Required AWS authentication pattern:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/CloudResume-GitHub-DeployRole
    aws-region: eu-west-2
    role-session-name: cloudresume-${{ github.run_id }}
```

Recommended verification step:

```yaml
- name: Verify AWS identity
  run: aws sts get-caller-identity
```

---

## 7. Deploy to S3

From the repository root:

```bash
aws s3 sync frontend/ s3://daviddigheji.com --delete
```

### Explanation

- `aws s3 sync` — synchronizes a local directory and S3 prefix.
- `frontend/` — local website source directory.
- `s3://daviddigheji.com` — destination bucket.
- `--delete` — removes destination objects that no longer exist in the source.

### Important warning

`--delete` is destructive if the wrong source or destination is specified.

Before using it manually, confirm:

```bash
pwd
ls -la frontend/
```

---

## 8. Verify the deployed S3 object

Check metadata without downloading the whole object:

```bash
AWS_PAGER="" aws s3api head-object \
  --bucket daviddigheji.com \
  --key index.html \
  --profile aegis-security
```

### Explanation

- `s3api` — calls the lower-level S3 API.
- `head-object` — returns object metadata.
- `--bucket` — specifies the bucket.
- `--key` — specifies the object key.

Review fields such as:

```text
LastModified
ContentLength
ETag
```

Optional content comparison:

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

No `diff` output means the files are identical.

---

## 9. Identify the correct CloudFront distribution

Do not guess the distribution ID.

Run:

```bash
AWS_PAGER="" aws cloudfront list-distributions \
  --profile aegis-security \
  --query 'DistributionList.Items[].{ID:Id,Domain:DomainName,Aliases:Aliases.Items,Status:Status}' \
  --output table
```

### Explanation

- `cloudfront list-distributions` — lists distributions.
- `--query` — uses JMESPath to return only useful fields.
- `--output table` — formats the result as a table.

Choose the distribution whose alias includes:

```text
daviddigheji.com
```

Store the correct ID locally:

```bash
CLOUDFRONT_DISTRIBUTION_ID="<CLOUDFRONT_DISTRIBUTION_ID>"
```

---

## 10. Invalidate CloudFront

Run:

```bash
aws cloudfront create-invalidation \
  --distribution-id "$CLOUDFRONT_DISTRIBUTION_ID" \
  --paths "/*" \
  --profile aegis-security
```

### Explanation

- `create-invalidation` — tells CloudFront to stop serving selected cached objects.
- `--distribution-id` — selects the target distribution.
- `--paths "/*"` — invalidates all paths.
- `--profile aegis-security` — uses the intended administrator profile.

Use `"/*"` deliberately. Large/frequent invalidations should be considered against cache design and cost.

---

## 11. Verify invalidation completion

```bash
AWS_PAGER="" aws cloudfront list-invalidations \
  --distribution-id "$CLOUDFRONT_DISTRIBUTION_ID" \
  --profile aegis-security \
  --query 'InvalidationList.Items[0:5].[Id,Status,CreateTime]' \
  --output table
```

Expected final status:

```text
Completed
```

If the newest invalidation says `InProgress`, wait before diagnosing the site as stale.

---

## 12. Verify the live website

Check HTTPS headers:

```bash
curl -I https://daviddigheji.com
```

### Explanation

- `curl` — transfers data to/from a URL.
- `-I` — requests response headers only.

Then search for a known newly deployed phrase:

```bash
curl -s https://daviddigheji.com | grep -F "<KNOWN_NEW_TEXT>"
```

- `-s` — silent mode.
- `grep -F` — fixed-string search.

Also verify:

```text
https://daviddigheji.com
https://www.daviddigheji.com
```

Use a private/incognito browser window if browser caching is suspected.

---

## 13. Deployment verification sequence

Use this order every time:

```text
SOURCE → IDENTITY → STORAGE → CACHE → LIVE SITE
```

1. Is the intended commit in GitHub?
2. Did GitHub Actions run?
3. Did OIDC authentication succeed?
4. Did STS show the deployment role?
5. Did S3 receive the current files?
6. Did CloudFront invalidation complete?
7. Does the live website show the new content?

Stop at the first failed layer.

---

## 14. Evidence capture

Recommended evidence:

```text
evidence/deployment/01-github-actions-success.png
evidence/deployment/02-sts-deployment-role.png
evidence/deployment/03-s3-object-verification.png
evidence/deployment/04-cloudfront-invalidation-completed.png
evidence/deployment/05-live-website-updated.png
```

Public-repository redaction:

- redact account IDs where unnecessary;
- never expose secret keys or tokens;
- avoid exposing unrelated internal resource identifiers.

---

## 15. Interview defence

**Question:** How do you troubleshoot a stale website behind CloudFront?

**Answer structure:**

> I troubleshoot the delivery chain in order: source, workload identity, origin storage, CDN cache, and finally the browser/live endpoint. I stop at the first failing layer rather than changing several components at once.
