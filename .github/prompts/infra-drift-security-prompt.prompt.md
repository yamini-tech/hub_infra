---
mode: agent
agent: infra-drift-security
name: infra-drift-security-prompt
description: "Prompt for the infra-drift-security agent. Runs drift detection and security misconfiguration scans against live AWS infrastructure using Terraform refresh-only plans and AWS CLI read-only commands."
---

### Requirements

1. **Drift Scan:** Run `terraform init -reconfigure` for the target environment, then `terraform plan -refresh-only -out=tfplan` followed by `terraform show -json tfplan`. Parse the JSON output to identify resources that have changed outside Terraform. Flag added, modified, and deleted resources.

2. **Security Scan:** Run the following checks against live AWS and report findings with severity:
   - S3 bucket public access (`get-bucket-policy-status`, ACL check)
   - Security groups with `0.0.0.0/0` ingress
   - Unencrypted RDS, EBS, and S3 resources
   - IAM policies with `"Effect": "Allow", "Action": "*"` (wildcard)
   - CloudTrail and VPC Flow Logs disabled
   - Default VPCs in use
   - Resources missing standard tags (`Environment`, `Project`, `ManagedBy`)

3. **Report Format:** Produce a structured report with sections for Drift and Security findings.

### Constraints

- Read-only operations only — never modify infrastructure, state, or code
- Use `aws` CLI with the environment-matching profile or `AWS_PROFILE` env var
- For cross-account access, assume a read-only role if `--role-arn` is provided
- Never expose or log access keys, secret keys, or session tokens
- If `aws configure` or AWS credentials are missing, report and stop — do not proceed

### Success Criteria

- Drift report lists each drifted resource with: resource address, expected vs actual attribute values, and drift origin (manual console change, external tool, unknown)
- Security report lists each finding with: resource ARN, severity, rule violated, and remediation hint
- Report ends with a go/no-go recommendation for CI deployment gate
- No AWS resources are created, modified, or deleted during the scan

### Output Format

```
## Drift & Security Scan Report — {environment}

### Drift Findings
| Resource | Expected | Actual | Origin |
|---|---|---|---|
| aws_s3_bucket.my_bucket | acl = "private" | acl = "public-read" | Manual console change |
| ... | | | |

### Security Findings
#### Critical
- [resource ARN] — [finding description] → [remediation hint]

#### Warnings
- ...

#### Info
- ...

### Risk Summary
[go / no-go] — [brief rationale]
```

### Usage Template

```
Run a {full / drift / security} scan on {environment}.
{Optional: specific region, profile, or additional checks to include}
Show the full report and wait for my review.
```

### Chat Example

```
User: Run a full drift + security scan on the staging environment.
```

Agent (expected):
- Runs `terraform init -reconfigure` pointing at staging remote state
- Runs `terraform plan -refresh-only` and parses the diff
- Queries AWS for security misconfigurations
- Produces structured findings report
- Waits for user to review the report before any next steps
