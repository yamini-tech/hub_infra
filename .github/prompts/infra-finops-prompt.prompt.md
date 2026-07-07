---
mode: agent
agent: infra-finops
name: infra-finops-prompt
description: "Prompt for the infra-finops agent. Scans Terraform definitions and live AWS usage data to identify low-utilization resources, orphaned assets, and savings opportunities, then produces a structured cost optimization report with specific Terraform change suggestions."
---

### Requirements

1. **Parse Terraform definitions:** Read each module in `terraform/modules/` to identify current resource configurations — `instance_class`, `node_type`, `allocated_storage`, `engine_version`, lifecycle rules, etc.

2. **Query live AWS usage** for the target environment(s):
   - CloudWatch `get-metric-statistics` for RDS `CPUUtilization` (averaged over `lookback_days`)
   - CloudWatch `get-metric-statistics` for ElastiCache `CPUUtilization` and `DatabaseMemoryUsageCount`
   - `aws ec2 describe-volumes` with `status: available` — orphaned EBS volumes
   - `aws ec2 describe-addresses` — Elastic IPs not associated to an ENI
   - `aws s3api list-objects` with storage class analysis for the S3 bucket
   - CloudWatch `get-metric-statistics` for ALB `RequestCount`
   - AWS Cost Explorer `get-reservation-utilization` for RI coverage

3. **Compare against thresholds** and flag resources below the configured warning levels.

4. **Calculate savings estimates** using current AWS on-demand pricing for the region (`ap-south-1`).

5. **Map findings to specific Terraform changes** — reference exact file paths and line numbers (e.g., `modules/rds/main.tf:20`).

### Constraints

- Read-only — never modify Terraform code, state, or live infrastructure
- Use `aws` CLI with the environment-matching profile or `AWS_PROFILE` env var
- Default region: `ap-south-1`
- Never expose or log access keys, secret keys, or session tokens
- If AWS credentials are missing or expired, report and stop
- S3 checks: list up to 1000 objects for storage class analysis (sample-based, not full inventory)

### Success Criteria

- Report lists each finding with: resource ARN, current config, actual utilization, recommended change, and estimated monthly savings
- Terraform change suggestions include exact file path and line number
- Total estimated monthly savings clearly stated
- Recommendations are prioritized by savings potential (largest first)
- Go/no-go recommendation for proceeding with cost-optimization changes

### Output Format

```
## FinOps Cost Analysis — {environment}

### Savings Opportunities (ordered by impact)

| Resource | Current | Utilization | Recommended | Monthly Savings |
|---|---|---|---|---|
| rds.postgres (dev) | db.t3.micro | CPU: 2.1% avg | db.t4g.micro | ~$3.50 |
| elasticache.redis (dev) | cache.t3.micro | CPU: 1.1%, Mem: 12% | cache.t4g.micro | ~$2.80 |
| ebs-orphan-001 | 30GB gp3 (unattached) | — | Delete | ~$3.60 |
| eip-unused-001 | 1 Elastic IP | 0 associations | Release | ~$3.65 |

### Total Estimated Savings: ~$13.55/month

### Terraform Changes Required

1. `modules/rds/main.tf:20` — Change `instance_class` from `"db.t3.micro"` to `"db.t4g.micro"`
2. `modules/elasticache/main.tf:15` — Change `node_type` from `"cache.t3.micro"` to `"cache.t4g.micro"`

### Go/No-Go: Go — all changes are safe downsizes within the same tier.
```

### Usage Template

```
Run a {full / quick} FinOps scan on {environment(s)}.
{Optional: custom thresholds, lookback period, or specific resource types to check}
Show the report and wait for my review.
```

### Chat Example

```
User: Run a FinOps scan on dev. I want to know if our RDS and Redis are over-provisioned.
```

Agent (expected):
- Reads `modules/rds/main.tf` and `modules/elasticache/main.tf` for current config
- Queries CloudWatch for 14-day CPU and memory metrics
- Checks for orphaned EBS volumes and unused Elastic IPs
- Calculates savings estimates using AWS pricing
- Produces structured report with file:line Terraform change references
- Waits for user to review before any next steps
