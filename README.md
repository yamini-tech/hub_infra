# hub_infra — CixioHub Infrastructure

Infrastructure-as-code for CixioHub's AWS environment. Manages all cloud resources via Terraform, the Mosquitto MQTT broker, and CI/CD pipelines.

## Tech Stack

- **Terraform** >= 1.7.0 with AWS provider
- **Mosquitto MQTT** 2.x — message broker
- **GitHub Actions** — CI/CD for Terraform validation and deployment

## Project Structure

```
hub_infra/
├── terraform/
│   ├── main.tf               # Module orchestration
│   ├── variables.tf           # Region, environment, secrets
│   ├── outputs.tf             # Endpoints consumed by services
│   └── modules/
│       ├── vpc/               # VPC, subnets, security groups
│       ├── rds/               # PostgreSQL 16
│       ├── elasticache/       # Redis 7.1
│       └── s3/                # Encrypted file storage
├── mosquitto/
│   └── mosquitto.conf         # MQTT broker configuration
├── workflows/
│   └── ci.yml                 # CI pipeline
└── .github/
    ├── agents/                # AI agent definitions (see below)
    └── prompts/               # Agent prompt files
```

## Using the Agents

All agents are invoked by prefixing your prompt with `@agent-name`. The coordinator (`@infra-agent`) routes requests to the correct single-task agent.

| Agent | When to use | Example prompt |
|---|---|---|
| `@infra-agent` | Unsure which agent to use | "I need to update our VPC configuration" |
| `@infra-terraform` | Create or update Terraform modules | "Add an RDS module with PostgreSQL 16" |
| `@infra-mosquitto` | Configure Mosquitto MQTT broker | "Add a WebSocket listener on port 9001 with TLS" |
| `@infra-ci` | Create or update CI workflows | "Create an infra.yml workflow for dev, staging, prod" |
| `@infra-planner` | Generate a plan before coding | "Plan adding ElastiCache Redis with multi-AZ" |
| `@infra-drift-security` | Detect drift and security misconfigurations | "Scan prod for unauthorized AWS changes" |
| `@infra-finops` | Analyze costs and right-size resources | "Find underutilized RDS instances in dev" |
| `@infra-code-reviewer` | Review Terraform and config changes before merge | "Review modules/rds/main.tf for security" |

Agent definitions are in `.github/agents/`. Each file documents its scope, inputs, outputs, and example prompts.

## Setup

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
# Fill in db_password, jwt_secret
terraform init
terraform plan
terraform apply
```

### Remote state (for team use)

Uncomment the `backend "s3"` block in `main.tf` and set `S3_STATE_BUCKET` before running `terraform init`.
