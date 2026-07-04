# aws-cloudgoat
# AWS Cloud Security Labs — CloudGoat

Hands-on AWS privilege escalation exercises built using [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat), a "vulnerable by design" AWS deployment tool created by Rhino Security Labs. Each folder in this repo documents one complete attack scenario: environment setup, reconnaissance, exploitation, verified impact, and remediation.

This project exists to build practical, applied experience in AWS IAM security and cloud attack chains — not just theory.

## Scenarios completed

| # | Scenario | Vulnerability Class | Folder |
|---|----------|---------------------|--------|
| 1 | IAM Privilege Escalation by Rollback | Unused/forgotten IAM policy versions | [`iam-privesc-by-rollback/`](./iam-privesc-by-rollback) |
| 2 | Vulnerable Lambda | SQL injection in a Lambda function → IAM privilege escalation | [`vulnerable-lambda/`](./vulnerable-lambda) |

More scenarios (SSRF against EC2 metadata service, and others) are in progress and will be added here.

## Environment

- Kali Linux (VM)
- Terraform
- AWS CLI v2
- CloudGoat (installed via `pipx`)
- A dedicated, isolated AWS test account (never a production or work account)

## Why CloudGoat

CloudGoat scenarios are modeled on real-world AWS misconfigurations and real breach patterns (e.g. IAM policy version sprawl, SSRF-to-instance-metadata attacks similar to the 2019 Capital One breach). Working through them hands-on builds the same recon → exploit → verify → remediate muscle memory used in real cloud security assessments and penetration tests.

## Disclaimer

All testing was performed exclusively against infrastructure I deployed myself in a dedicated, isolated AWS account created specifically for this purpose. No production systems, third-party infrastructure, or real user data were involved. All environments were destroyed immediately after each exercise.
