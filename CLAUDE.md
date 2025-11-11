# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Alex** (Agentic Learning Equities eXplainer) is a multi-agent enterprise-grade SaaS financial planning platform built as a capstone project for the "AI in Production" course. It demonstrates production-ready AI agent orchestration using AWS serverless architecture.

**Technology Stack:**
- **Agents**: OpenAI Agents SDK with LiteLLM + AWS Bedrock (Nova Pro model)
- **Backend**: Python with uv package manager, Lambda functions, App Runner
- **Frontend**: NextJS (Pages Router) with Clerk authentication
- **Infrastructure**: Terraform with independent state per component
- **Database**: Aurora Serverless v2 PostgreSQL with Data API
- **Vector Store**: S3 Vectors with SageMaker embeddings
- **Orchestration**: SQS queue-based agent coordination

## Critical Development Rules

### Python & uv
- **ALWAYS use `uv run` for all Python execution** - Never use bare `python` command
- **ALWAYS use `uv add` for dependencies** - Never use `pip install`
- Examples: `uv run script.py`, `uv add package-name`, `uv sync`
- Each backend subdirectory is a uv project (nested projects are intentional)
- uv may warn about nested projects - this is expected and can be ignored

### Terraform
- Each terraform directory (2_sagemaker, 3_ingestion, etc.) is **independent** with its own state
- **MUST copy `terraform.tfvars.example` to `terraform.tfvars`** before applying
- No shared state, no remote backends - all local state files
- Apply/destroy order matters: deploy 2→8, destroy 8→2

### Docker
- Lambda packaging requires Docker Desktop running (for linux/amd64 builds)
- Script: `backend/package_docker.py` - runs packaging for all agents
- Per-agent: `cd backend/{agent} && uv run package_docker.py`

## Development Commands

### Local Development
```bash
# Run frontend + backend locally
uv run scripts/run_local.py

# Requires:
# - .env file in root (backend config)
# - frontend/.env.local (Clerk config)
```

### Testing Agents

Each agent directory has standardized test files:

```bash
# Local testing with mocks (fast, no AWS calls)
cd backend/{agent}
uv run test_simple.py

# Full AWS integration testing (requires deployed infrastructure)
cd backend/{agent}
uv run test_full.py

# Backend-wide integration tests
cd backend
uv run test_simple.py  # With MOCK_LAMBDAS=true
uv run test_full.py    # Against real Lambda functions
```

### Lambda Packaging & Deployment

```bash
# Package all Lambda functions (requires Docker running)
cd backend
uv run package_docker.py

# Deploy all packaged Lambdas to AWS
cd backend
uv run deploy_all_lambdas.py

# Package single agent
cd backend/{agent}
uv run package_docker.py
```

### Terraform Deployment

Each guide has its own terraform directory with independent state:

```bash
# Generic pattern for each guide
cd terraform/{N_component}
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with required values
terraform init
terraform plan
terraform apply
terraform output  # Get ARNs/values for next steps

# Cleanup (reverse order recommended)
terraform destroy
```

### Database Operations

```bash
# Check database connectivity and schema
cd backend
uv run check_db.py

# Check job details
cd backend
uv run check_job_details.py
```

### Frontend Deployment

```bash
# Deploy to S3/CloudFront (requires Guide 7 infrastructure)
uv run scripts/deploy.py
```

### Monitoring

```bash
# Watch agent execution in real-time
cd backend
uv run watch_agents.py

# CloudWatch logs for specific Lambda
aws logs tail /aws/lambda/alex-{agent-name} --follow
```

## Architecture

### Multi-Agent System

Five specialized Lambda agents orchestrated by the Planner:

```
User Request → API Gateway → SQS Queue → Planner Agent (Orchestrator)
                                           ├─→ Tagger Agent (classify instruments)
                                           ├─→ Reporter Agent (portfolio analysis)
                                           ├─→ Charter Agent (visualizations)
                                           └─→ Retirement Agent (projections)
```

### Agent Code Structure

Each agent follows the same pattern:

```
backend/{agent}/
├── agent.py           # Agent logic, tool definitions
├── lambda_handler.py  # Lambda entry point, agent execution
├── templates.py       # Prompt templates
├── test_simple.py     # Local testing with mocks
├── test_full.py       # AWS integration testing
├── package_docker.py  # Docker-based packaging for Lambda
├── pyproject.toml     # uv project definition
└── uv.lock           # Dependency lock file
```

### OpenAI Agents SDK Patterns

**Standard Agent Creation:**
```python
from agents import Agent, Runner, trace, function_tool
from agents.extensions.models.litellm_model import LitellmModel

# Model setup
model = LitellmModel(model=f"bedrock/{model_id}")

# Agent execution
with trace("Agent Name"):
    agent = Agent(
        name="Agent Name",
        instructions=INSTRUCTIONS,
        model=model,
        tools=tools
    )

    result = await Runner.run(
        agent,
        input=task,
        max_turns=20
    )

    response = result.final_output
```

**Tool with Context (for database access):**
```python
from agents import function_tool, RunContextWrapper
from dataclasses import dataclass

@dataclass
class AgentContext:
    user_id: str
    job_id: str

@function_tool
async def tool_with_context(
    wrapper: RunContextWrapper[AgentContext],
    param: str
) -> str:
    context = wrapper.context
    # Access context.user_id, context.job_id
    return "result"

# When running agent:
result = await Runner.run(
    agent,
    input=task,
    context=context,  # Pass context object
    max_turns=10
)
```

**Important Limitations:**
- With Bedrock via LiteLLM: Agent cannot use BOTH Structured Outputs AND Tool calling
- Each agent uses either Structured Outputs OR Tools, never both
- LiteLLM requires: `os.environ["AWS_REGION_NAME"] = bedrock_region`

### Database Schema

Aurora Serverless v2 PostgreSQL accessed via Data API (no VPC needed):

**Core Tables:**
- `users` - Clerk user IDs with preferences
- `accounts` - User financial accounts
- `positions` - Holdings in accounts
- `instruments` - ETFs/stocks with allocation data
- `jobs` - Agent execution requests
- `results` - Agent outputs (reports, charts, retirement projections)
- `market_insights` - Research agent findings

**Database Access Pattern:**
```python
from database.db_client import AuroraDatabase

db = AuroraDatabase(
    cluster_arn=os.getenv("AURORA_CLUSTER_ARN"),
    secret_arn=os.getenv("AURORA_SECRET_ARN"),
    database=os.getenv("AURORA_DATABASE", "alex")
)

# Each table has typed methods
users = db.users.find_by_clerk_id(clerk_user_id)
accounts = db.accounts.find_by_user(user_id)
positions = db.positions.find_by_account(account_id)
```

### Researcher Agent (App Runner)

Autonomous research agent that runs on App Runner with MCP server integration:

- Playwright MCP server for web browsing
- EventBridge scheduler for periodic execution (optional)
- Stores findings in `market_insights` table
- Deployed via Docker to ECR, then App Runner

## Environment Variables

Root `.env` file structure (values populated guide-by-guide):

```bash
# Guide 1-2: Foundation
AWS_ACCOUNT_ID=
DEFAULT_AWS_REGION=
SAGEMAKER_ENDPOINT=alex-embedding-endpoint

# Guide 3: Ingestion
VECTOR_BUCKET=
ALEX_API_ENDPOINT=
ALEX_API_KEY=

# Guide 4: Researcher
OPENAI_API_KEY=

# Guide 5: Database
AURORA_CLUSTER_ARN=
AURORA_SECRET_ARN=
AURORA_DATABASE=alex

# Guide 6: Agents
BEDROCK_MODEL_ID=us.amazon.nova-pro-v1:0
BEDROCK_REGION=us-west-2
POLYGON_API_KEY=
POLYGON_PLAN=free

# Guide 7: Frontend
CLERK_JWKS_URL=
CLERK_ISSUER=
SQS_QUEUE_URL=
FRONTEND_URL=http://localhost:3000
CLOUDFRONT_URL=

# Guide 8: Observability (optional)
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=
```

Frontend requires separate `frontend/.env.local`:
```bash
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
NEXT_PUBLIC_API_URL=http://localhost:8000
```

## Common Issues & Solutions

### Docker Packaging Fails
- **Check**: Is Docker Desktop running? Run `docker ps` to verify
- **Check**: Docker file sharing enabled for working directory
- **Not the issue**: uv nested project warnings (these are harmless)

### Bedrock Model Access Denied
- **Check**: Model access granted in Bedrock console for the region
- **Check**: Nova Pro (`us.amazon.nova-pro-v1:0`) is recommended (Claude has rate limits)
- **Check**: Cross-region inference profiles need multi-region access
- **Check**: `AWS_REGION_NAME` environment variable set (LiteLLM requires this specific name)
- **Debug**: Add logging to confirm which region/model is being used

### Terraform Apply Fails
- **Check**: `terraform.tfvars` exists and is populated
- **Check**: Values from previous guide outputs (use `terraform output` in earlier directories)
- **Check**: ARNs/endpoints from previous guides are correct

### Lambda Import Errors
- **Check**: Package built with Docker (linux/amd64 architecture)
- **Check**: Dependencies in `pyproject.toml` are correct
- **Re-package**: `cd backend/{agent} && uv run package_docker.py`
- **Check CloudWatch logs**: `aws logs tail /aws/lambda/alex-{agent} --follow`

### Database Connection Issues
- **Check**: Cluster is "available" status: `aws rds describe-db-clusters`
- **Check**: Data API is enabled (`EnableHttpEndpoint: true`)
- **Check**: ARNs in environment match actual resources
- **Wait**: Database initialization takes 10-15 minutes after first deploy

## Project Context

This is an educational project for the "AI in Production" course by Ed Donner. Users are students learning to deploy production AI systems. The guides (in `guides/` directory) provide step-by-step instructions for deploying each component.

When helping users:
1. Ask which guide they're working on (1-8) to understand what's deployed
2. Check actual error messages and CloudWatch logs before diagnosing
3. Verify prerequisites (Docker running, terraform.tfvars configured, model access granted)
4. Make minimal, targeted changes - avoid over-engineering solutions
5. Help them understand the problem, not just fix it

## File Locations

**Configuration:**
- `.env` - Root environment variables
- `frontend/.env.local` - Frontend Clerk configuration
- `terraform/*/terraform.tfvars` - Per-component infrastructure variables

**Guides (step-by-step instructions):**
- `guides/1_permissions.md` - IAM setup
- `guides/2_sagemaker.md` - Embedding endpoint
- `guides/3_ingest.md` - S3 Vectors & ingestion Lambda
- `guides/4_researcher.md` - App Runner research agent
- `guides/5_database.md` - Aurora Serverless v2
- `guides/6_agents.md` - Multi-agent Lambda deployment
- `guides/7_frontend.md` - NextJS frontend & API
- `guides/8_enterprise.md` - Monitoring & observability

**Backend Agents:**
- `backend/planner/` - Orchestrator (decides which agents to invoke)
- `backend/tagger/` - Classifies instruments (ETF/Stock/etc)
- `backend/reporter/` - Portfolio analysis reports
- `backend/charter/` - Chart/visualization generation
- `backend/retirement/` - Retirement projections
- `backend/researcher/` - Market research (App Runner)
- `backend/ingest/` - Document ingestion Lambda
- `backend/database/` - Shared database client library
- `backend/api/` - FastAPI backend for frontend

**Infrastructure:**
- `terraform/2_sagemaker/` - SageMaker endpoint
- `terraform/3_ingestion/` - S3 Vectors, ingest Lambda, API Gateway
- `terraform/4_researcher/` - App Runner, ECR, EventBridge
- `terraform/5_database/` - Aurora cluster, Secrets Manager
- `terraform/6_agents/` - 5 Lambda agents, SQS queue, S3 packages
- `terraform/7_frontend/` - S3, CloudFront, API Gateway, API Lambda
- `terraform/8_enterprise/` - CloudWatch dashboards, alarms

**Deployment Scripts:**
- `scripts/run_local.py` - Local development (frontend + backend)
- `scripts/deploy.py` - Frontend deployment to S3/CloudFront
- `scripts/destroy.py` - Cleanup infrastructure
- `backend/package_docker.py` - Package all Lambdas
- `backend/deploy_all_lambdas.py` - Deploy all Lambdas to AWS

## Cost Management

Primary costs: Aurora Serverless v2 (runs continuously)

**Cost Optimization:**
- Destroy Aurora when not actively working: `cd terraform/5_database && terraform destroy`
- Monitor: AWS Cost Explorer, billing alerts
- Each terraform component can be destroyed independently
