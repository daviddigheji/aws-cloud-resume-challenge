# CloudFront Cache Invalidation Runbook

**Purpose:** Safely invalidate and verify CloudFront cache after a website deployment.

---

## 1. When to use this runbook

Use when:

- S3 contains the new website files;
- GitHub deployment completed;
- the public website still appears stale;
- CloudFront must be forced to re-fetch current objects.

Do not begin here if AWS authentication or S3 deployment already failed.

---

## 2. Authenticate

```bash
aws sso login --profile aegis-security
```

Verify:

```bash
AWS_PAGER="" aws sts get-caller-identity \
  --profile aegis-security
```

---

## 3. Discover the correct distribution

```bash
AWS_PAGER="" aws cloudfront list-distributions \
  --profile aegis-security \
  --query 'DistributionList.Items[].{ID:Id,Domain:DomainName,Aliases:Aliases.Items,Status:Status}' \
  --output table
```

Find the distribution whose alias contains:

```text
daviddigheji.com
```

### Why

Never invalidate a distribution ID from memory when multiple distributions exist.

---

## 4. Store the ID in a shell variable

```bash
CLOUDFRONT_DISTRIBUTION_ID="<CLOUDFRONT_DISTRIBUTION_ID>"
```

Verify:

```bash
echo "$CLOUDFRONT_DISTRIBUTION_ID"
```

---

## 5. Create the invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id "$CLOUDFRONT_DISTRIBUTION_ID" \
  --paths "/*" \
  --profile aegis-security
```

### Explanation

- `create-invalidation` — creates an invalidation request.
- `--distribution-id` — specifies the distribution.
- `--paths "/*"` — marks all paths as stale.
- `--profile` — selects the intended CLI profile.

---

## 6. Verify the invalidation

```bash
AWS_PAGER="" aws cloudfront list-invalidations \
  --distribution-id "$CLOUDFRONT_DISTRIBUTION_ID" \
  --profile aegis-security \
  --query 'InvalidationList.Items[0:5].[Id,Status,CreateTime]' \
  --output table
```

Expected final state:

```text
Completed
```

`InProgress` means CloudFront has not finished yet.

---

## 7. Verify origin first

Before blaming CloudFront, verify that S3 contains the current object:

```bash
AWS_PAGER="" aws s3api head-object \
  --bucket daviddigheji.com \
  --key index.html \
  --profile aegis-security
```

If the S3 object is old, invalidation cannot fix the deployment.

---

## 8. Verify the public endpoint

```bash
curl -I https://daviddigheji.com
```

Then search content:

```bash
curl -s https://daviddigheji.com | grep -F "<KNOWN_NEW_TEXT>"
```

Also test in a private/incognito browser window.

---

## 9. Troubleshooting sequence

If the site is still stale:

```text
1. Verify GitHub source
2. Verify workflow success
3. Verify STS deployment role
4. Verify S3 LastModified
5. Compare S3 object content
6. Verify invalidation exists
7. Verify invalidation is Completed
8. Verify CloudFront alias
9. Verify CloudFront origin
10. Test browser/private session
```

---

## 10. Wrong fixes

### Wrong: change DNS first

If the domain resolves and shows the old website, DNS is already taking the request somewhere.

Investigate content delivery first.

### Wrong: make S3 public

This can bypass the intended CloudFront/OAC security model.

### Wrong: invalidate a guessed distribution

You may modify an unrelated environment.

---

## 11. Evidence capture

```text
evidence/cloudfront/01-distribution-identification.png
evidence/cloudfront/02-invalidation-created.png
evidence/cloudfront/03-invalidation-completed.png
evidence/cloudfront/04-live-site-current.png
```

Redact unrelated account information.
