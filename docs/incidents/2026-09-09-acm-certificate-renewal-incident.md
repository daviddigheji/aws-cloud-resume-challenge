# Incident Record — ACM Certificate Renewal Alert

**Incident window:** 8–9 September 2026  
**Service:** HTTPS certificate for `daviddigheji.com` / `www.daviddigheji.com`  
**AWS service:** AWS Certificate Manager (ACM)  
**Certificate region:** `us-east-1`  
**Status:** Resolved / renewal verified

---

## 1. Summary

AWS sent an alert stating that ACM had been unable to automatically renew the DNS-validated SSL/TLS certificate protecting the website.

The existing certificate was approaching its previous expiration date, so the alert was treated as time-sensitive.

---

## 2. Alert condition

AWS reported that:

```text
Automatic ACM renewal using DNS validation had failed.
```

Potential impact if not resolved:

```text
certificate expires
→ browser HTTPS warnings / TLS failure
→ website or application may become unreachable over HTTPS
```

---

## 3. Domains involved

```text
daviddigheji.com
www.daviddigheji.com
```

---

## 4. Investigation

The investigation focused on the ACM DNS-validation path.

Actions included:

1. authenticating through AWS IAM Identity Center;
2. opening the AWS Management Console;
3. switching from `eu-west-2` to `us-east-1`;
4. opening AWS Certificate Manager;
5. inspecting the existing certificate rather than requesting a new one;
6. checking domain validation/renewal state;
7. investigating the ACM CNAME validation records in DNS;
8. using DNS lookup commands such as `dig` to test CNAME resolution;
9. rechecking ACM for final renewal status and new validity dates.

---

## 5. DNS verification approach

The validation CNAME records were checked using the pattern:

```bash
dig CNAME <ACM_VALIDATION_NAME> +short
```

Authoritative DNS could also be queried directly:

```bash
dig @<AUTHORITATIVE_NAMESERVER> \
  CNAME <ACM_VALIDATION_NAME> \
  +short
```

This helped distinguish DNS publication/authority issues from cached resolver behavior.

Exact validation tokens are intentionally omitted from this public incident record.

---

## 6. Final ACM state

On 9 September 2026, ACM showed:

```text
Certificate status: Issued
Renewal status: Success
In use: Yes
Validation method: DNS
Renewal eligibility: Eligible
```

Both domains showed successful renewal/validation.

The certificate details showed:

```text
Issued at: 8 September 2026
Not before: 8 September 2026
Not after: 24 March 2027 23:59:59 UTC
```

This confirmed that the previous expiration risk had been removed.

---

## 7. Root cause statement

The original alert established that ACM could not complete automatic renewal through DNS validation at that time.

The troubleshooting process focused on ACM validation CNAME availability and DNS resolution.

Because the final verified state already showed successful renewal, this incident record does not claim a more specific root cause than the evidence supports.

This is intentional: incident documentation should distinguish between:

```text
confirmed fact
```

and:

```text
unproven hypothesis
```

---

## 8. Resolution

The incident was closed only after verifying:

```text
ACM renewal status = Success
certificate = In use
renewal eligibility = Eligible
new Not after date = 24 March 2027
```

No replacement certificate was required.

No unnecessary Route 53 record creation was performed after successful renewal was visible.

---

## 9. Preventive actions

1. Keep ACM validation CNAME records present in authoritative DNS.
2. Do not delete validation records after initial certificate issuance.
3. Keep validation CNAMEs DNS-resolvable.
4. Verify ACM renewal eligibility periodically.
5. Treat future ACM expiration alerts as operational incidents.
6. Capture the new `Not after` date as closure evidence.
7. Avoid publishing validation CNAME tokens in a public repository.

---

## 10. Evidence

Recommended evidence file:

```text
evidence/acm/acm-certificate-renewal-success-2026-09-09.png
```

Public version should retain:

```text
In use: Yes
Domain name
Validation method: DNS
Renewal eligibility: Eligible
Issued at
Not after
```

Crop/redact unnecessary:

```text
certificate identifier
full ARN
account ID
validation CNAME names/values
```

---

## 11. Related runbooks

- `../runbooks/04-acm-certificate-renewal-runbook.md`
- `../runbooks/05-aws-identity-center-access-verification-runbook.md`
