# ACM Certificate Renewal Runbook

**Purpose:** Diagnose and verify automatic renewal of the ACM certificate protecting `daviddigheji.com` and `www.daviddigheji.com`.

**Important region:** `us-east-1` for the CloudFront viewer certificate.

---

## 1. Trigger conditions

Use this runbook when AWS sends an alert similar to:

```text
AWS Certificate Manager was unable to renew the certificate automatically using DNS validation.
```

Treat the alert as time-sensitive because certificate expiry can cause HTTPS failures.

---

## 2. Do not do these first

Do not immediately:

- delete the current ACM certificate;
- request a replacement certificate;
- remove the current CloudFront certificate association;
- change unrelated DNS records;
- click "Create records in Route 53" when Cloudflare or another provider is authoritative;
- expose ACM validation CNAME tokens in public screenshots.

First inspect the existing certificate and DNS validation state.

---

## 3. Authenticate to AWS

CLI:

```bash
aws sso login --profile aegis-security
```

Verify:

```bash
AWS_PAGER="" aws sts get-caller-identity \
  --profile aegis-security
```

For console access, use the IAM Identity Center Access Portal and open the assigned Management Console role.

Do not publish the private Access Portal URL in a public repository.

---

## 4. Open ACM in the correct region

In the AWS Management Console:

```text
AWS Certificate Manager
→ Region: US East (N. Virginia) / us-east-1
→ Certificates
→ Select certificate for daviddigheji.com
```

Why `us-east-1`?

The CloudFront viewer certificate is managed there even if other application resources are primarily in London.

---

## 5. Inspect certificate status

Record:

```text
Certificate status
Renewal status
In use
Domains
Validation method
Renewal eligibility
Not after
```

Healthy target state:

```text
Certificate status: Issued
Renewal status: Success
In use: Yes
Validation method: DNS
Renewal eligibility: Eligible
```

Both domains should show success:

```text
daviddigheji.com
www.daviddigheji.com
```

---

## 6. Inspect ACM validation CNAMEs

For each domain, ACM displays:

```text
CNAME name
CNAME value
```

Do not commit these tokens to a public repository unless there is a documented reason.

If DNS is managed in Cloudflare, verify that the matching CNAME records exist there.

For validation-only CNAMEs, use DNS-only behavior rather than an unnecessary proxy layer.

---

## 7. Verify DNS from the terminal

Use the actual CNAME name shown by ACM.

Example pattern:

```bash
dig CNAME <ACM_VALIDATION_NAME_FOR_ROOT> +short
```

For `www`:

```bash
dig CNAME <ACM_VALIDATION_NAME_FOR_WWW> +short
```

### Explanation

- `dig` — DNS lookup utility.
- `CNAME` — request the canonical-name record type.
- `+short` — display concise output.

Expected:

```text
<ACM_VALIDATION_TARGET>.acm-validations.aws.
```

If no result appears, verify the DNS record at the authoritative provider.

---

## 8. Query an authoritative nameserver directly

If Cloudflare is authoritative, identify the real Cloudflare nameserver and query it:

```bash
dig @<AUTHORITATIVE_NAMESERVER> \
  CNAME <ACM_VALIDATION_NAME> \
  +short
```

### Why

This separates:

```text
authoritative DNS problem
```

from:

```text
recursive resolver/cache problem
```

---

## 9. Recheck ACM

After DNS is correct, return to ACM and inspect:

```text
Renewal status
Domain validation status
Issued at
Not after
```

Do not assume success merely because DNS looks correct.

The final proof is ACM reporting successful renewal.

---

## 10. Verify the renewed validity period

The incident is not closed until the expiration date has moved forward.

For the September 2026 incident, the final verified state was:

```text
Issued at: 8 September 2026
Not after: 24 March 2027 23:59:59 UTC
Renewal status: Success
Renewal eligibility: Eligible
In use: Yes
Validation method: DNS
```

---

## 11. Verify HTTPS publicly

```bash
curl -I https://daviddigheji.com
```

Then:

```bash
curl -I https://www.daviddigheji.com
```

A successful HTTPS response helps verify that the public endpoints remain reachable.

For certificate details, a browser certificate viewer or TLS inspection tool may also be used when required.

---

## 12. Future prevention

Keep the ACM DNS validation CNAME records permanently unless the certificate/domain design is intentionally retired.

Do not treat validation CNAMEs as temporary records that should be deleted after issuance.

Periodically verify:

```text
ACM status = Issued
Renewal eligibility = Eligible
Certificate = In use
Validation records = resolvable
```

---

## 13. Evidence capture

Recommended public-safe evidence:

```text
evidence/acm/acm-certificate-renewal-success-2026-09-09.png
```

The screenshot should show:

```text
In use: Yes
Domain name: daviddigheji.com
Validation method: DNS
Renewal eligibility: Eligible
Issued at
Not after
```

Before publishing, crop/redact:

- certificate identifier if unnecessary;
- full ARN;
- account ID;
- validation CNAME names/values;
- unrelated internal identifiers.

---

## 14. Decision tree

```text
AWS renewal alert
      ↓
Open ACM us-east-1
      ↓
Renewal already Success?
   YES → verify new Not after date → verify HTTPS → close incident
   NO
      ↓
Check both domain validation statuses
      ↓
Check CNAME records at authoritative DNS
      ↓
Verify with dig
      ↓
Correct DNS only if necessary
      ↓
Recheck ACM
      ↓
Success + new expiry date
```
