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

**Independent Directory Architecture:**

Each terraform directory (2_sagemaker, 3_ingestion, etc.) is **independent** with:
- Its own local state file (`terraform.tfstate`)
- Its own `terraform.tfvars` configuration
- No dependencies on other terraform directories

**This is intentional** for educational purposes:
- Students can deploy incrementally, guide by guide
- State files are local (simpler than remote S3 state)
- Each part can be destroyed independently
- No complex state bucket setup needed
- Infrastructure can be destroyed step by step

**Critical Requirements:**
- ⚠️ **MUST copy `terraform.tfvars.example` to `terraform.tfvars`** before applying
- Common pattern: Use Cursor File Explorer to copy and edit in each directory
- If `terraform.tfvars` is missing or misconfigured:
  - Terraform will use default values (often wrong)
  - Resources may fail to create with cryptic errors
  - Cross-service connections will break

**State Management:**
- State files are `.gitignored` automatically
- Local state means no S3 bucket needed
- Students can `terraform destroy` each directory independently
- If a student loses state, they may need to import existing resources or recreate

**Deployment Order:**
- Apply order matters: deploy 2→8
- Destroy order recommended: destroy 8→2 (reverse)

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

## Common Issues & Troubleshooting

**The most common issues relate to AWS region choices! Check environment variables, terraform settings (everything should propagate from tfvars).**

### Issue 1: `package_docker.py` Fails

**Symptoms**: Script fails with uv warning about nested projects and perhaps an error message

**Root Cause (common)**: Docker Desktop is not running or a Docker mounts denied issue

**Diagnosis**:
1. Ask: "Is Docker Desktop running?"
2. Check: Can they run `docker ps` successfully?
3. Recent fix: The script now gives better error messages, but older versions were misleading

**Solution**: Start Docker Desktop, wait for it to fully initialize, then retry

**If the issue is a Mounts Denied error**: It fails to mount the /tmp directory into Docker as it doesn't have access to it. Going to Docker Desktop app, and adding the directory mentioned in the error to the shared paths (Settings → Resources → File Sharing) solved the problem for a student.

**Not the solution**: Changing uv project configurations (this is a red herring)

### Issue 2: Region Issues and Bedrock Model Access Denied

**Symptoms**: "Access denied" or "Model not found" errors when running agents

**Root Cause**: Model access not granted in Bedrock, or wrong region

**Diagnosis**:
1. Which model are they trying to use?
2. Which region is their code running in?
3. Have they requested model access in Bedrock console?
4. For inference profiles: Do they have permissions for multiple regions?
5. Are the right environment variables being set? LiteLLM needs `AWS_REGION_NAME`. Check that nothing is being hardcoded in the code, and that tfvars are set right. Add logging to confirm which region is being used.

**Solution**:
1. Go to Bedrock console in the correct region
2. Click "Model access"
3. Request access to Nova Pro
4. For cross-region: Set up inference profiles with multi-region permissions

### Issue 3: Terraform Apply Fails

**Symptoms**: Resources fail to create, dependency errors, ARN not found

**Root Cause**: `terraform.tfvars` not configured, or values from previous guides not set

**Diagnosis**:
1. Does `terraform.tfvars` exist in this directory?
2. Are all required variables set (check `terraform.tfvars.example`)?
3. For later guides: Do they have output values from earlier guides?
4. Run `terraform output` in previous directories to get required ARNs

**Solution**:
1. Copy `terraform.tfvars.example` to `terraform.tfvars`
2. Fill in all required values
3. Get ARNs from previous terraform outputs: `cd terraform/X_previous && terraform output`
4. Update `.env` file with values for Python scripts

### Issue 4: Lambda Function Failures

**Symptoms**: 500 errors, timeout errors, "Module not found" errors

**Root Cause**: Package not built correctly, environment variables missing, or IAM permissions

**Diagnosis**:
1. Check CloudWatch logs: `aws logs tail /aws/lambda/alex-{agent-name} --follow`
2. Check Lambda environment variables in AWS Console
3. Check IAM role has required permissions
4. Was the Lambda package built with Docker for linux/amd64?

**Solution**:
1. For packaging: Re-run `package_docker.py` with Docker running
2. For env vars: Verify in Lambda console or `terraform.tfvars`
3. For permissions: Check IAM role policy in terraform

### Issue 5: Aurora Database Connection Fails

**Symptoms**: "Cluster not found", "Secret not found", Data API errors

**Root Cause**: Database not fully initialized, wrong ARNs, or Data API not enabled

**Diagnosis**:
1. Check cluster status: `aws rds describe-db-clusters`
2. Verify Data API is enabled (should show `EnableHttpEndpoint: true`)
3. Check ARNs in environment variables match actual resources
4. Database may still be initializing (takes 10-15 minutes)

**Solution**:
1. Wait for cluster to reach "available" status
2. Verify Data API is enabled in RDS console
3. Run `terraform output` in `5_database` to get correct ARNs
4. Update environment variables with actual ARNs

## Project Context

This is an educational project for the "AI in Production" course by Ed Donner (Udemy). Users are students learning to deploy production AI systems. Students work in Cursor (VS Code fork) on Windows/Mac/Linux. They're familiar with AWS services and have been introduced to Terraform, uv, NextJS, and Docker.

**Student Setup:**
- AWS root user + IAM user "aiengineer" with permissions
- AWS CLI configured with aiengineer user
- Budget alerts set (but should regularly check billing)

### Working with Students: Core Principles

**IMPORTANT: Before starting, read all guides (1-8) in the guides folder for full background.**

#### 1. Always Establish Context First

When a student asks for help:
1. **Ask which guide/day they're on** - Critical for understanding what infrastructure is deployed
2. **Ask what they're trying to accomplish** - Understand the goal before diving into code
3. **Ask what error or behavior they're seeing** - Get actual error message, not interpretation

#### 2. Diagnose Before Fixing ⚠️ MOST IMPORTANT

**DO NOT jump to conclusions and write lots of code before the problem is truly understood.**

Common mistakes to avoid:
- Writing defensive code with `isinstance()` checks before understanding root cause
- Adding try/except blocks that hide the real error
- Creating workarounds that mask the actual problem
- Making multiple changes at once (makes debugging impossible)

**Instead, follow this process:**
1. **Reproduce the issue** - Ask for exact error messages, logs, commands
2. **Identify root cause** - Use CloudWatch logs, AWS Console, error traces
3. **Verify understanding** - Explain what you think is happening and confirm with student
4. **Propose minimal fix** - Change one thing at a time
5. **Test and verify** - Confirm the fix works before moving on

#### 3. Common Root Causes (Check These First)

**Docker Desktop Not Running** (Most common with `package_docker.py`)
- Script fails with generic uv warning about nested projects
- Real issue is Docker isn't running
- Students often get distracted by the uv warning
- **Always ask**: "Is Docker Desktop running?"
- **Mounts Denied error**: Add directory to Docker file sharing (Settings → Resources → File Sharing)

**AWS Permissions Issues** (Most common overall)
- Missing IAM policies for specific AWS services
- Region-specific permissions (especially for Bedrock inference profiles)
- Inference profiles require permissions for MULTIPLE regions
- **Check**: IAM policies, AWS region settings, Bedrock model access

**Terraform Variables Not Set**
- Each terraform directory needs `terraform.tfvars` configured
- Missing or incorrect variables cause cryptic errors
- **Check**: Does `terraform.tfvars` exist? Are all required variables set?

**AWS Region Mismatches**
- Bedrock models may only be available in specific regions
- Nova Pro requires inference profiles
- Cross-region resource access may need models approved in Bedrock in multiple regions
- **Check**: Region consistency across configuration files

**Model Access Not Granted**
- AWS Bedrock requires explicit model access requests
- Nova Pro is recommended (Claude Sonnet has strict rate limits)
- Access is per-region; inference profiles may require multiple regions
- **Check**: Bedrock console → Model access

#### 4. Current Model Strategy

**Use Nova Pro, not Claude Sonnet**
- Nova Pro (`us.amazon.nova-pro-v1:0` or `eu.amazon.nova-pro-v1:0`) is recommended
- Requires inference profiles for cross-region access
- Claude Sonnet has too strict rate limits for this project
- Students need to request access in AWS Bedrock console, potentially for multiple regions

#### 5. Help Students Help Themselves

Encourage students to:
- Read error messages carefully (especially CloudWatch logs)
- Check AWS Console to verify resources exist
- Use `terraform output` to see deployed resource details
- Test incrementally (don't deploy everything at once)
- Keep AWS costs in mind (remind them to destroy when not actively working)

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

**Cleanup Process (reverse order recommended):**
```bash
cd terraform/8_enterprise && terraform destroy
cd terraform/7_frontend && terraform destroy
cd terraform/6_agents && terraform destroy
cd terraform/5_database && terraform destroy  # Biggest cost savings
cd terraform/4_researcher && terraform destroy
cd terraform/3_ingestion && terraform destroy
cd terraform/2_sagemaker && terraform destroy
```

---

## Course Structure: Week 3 & 4

### Week 3: Research Infrastructure

**Day 3 - Foundations**
- Guide 1: AWS Permissions (IAM setup, AlexAccess group)
- Guide 2: SageMaker Deployment (HuggingFace embeddings endpoint)

**Day 4 - Vector Storage**
- Guide 3: Ingestion Pipeline (S3 Vectors, Lambda, API Gateway)

**Day 5 - Research Agent**
- Guide 4: Researcher Agent (App Runner, Bedrock Nova Pro, Playwright MCP)

### Week 4: Portfolio Management Platform

**Day 1 - Database**
- Guide 5: Database & Infrastructure (Aurora Serverless v2, Data API, seed data)

**Day 2 - Agent Orchestra**
- Guide 6: AI Agent Orchestra (5 Lambda agents, SQS orchestration, parallel processing)

**Day 3 - Frontend**
- Guide 7: Frontend & API (Clerk auth, NextJS, FastAPI backend, CloudFront)

**Day 4 - Enterprise Features**
- Guide 8: Enterprise Grade (CloudWatch, WAF, GuardDuty, LangFuse observability)

---

## Getting Help

### For Students

If you're stuck:
1. **Check the guide carefully** - Most steps have troubleshooting sections
2. **Review error messages** - Look at CloudWatch logs, not just terminal output
3. **Verify prerequisites** - Is Docker running? Are permissions set? Is terraform.tfvars configured?
4. **Contact the instructor**:
   - Post a question in Udemy (include guide number, error message, what you've tried)
   - Email Ed Donner: ed@edwarddonner.com

When asking for help, include:
- Which guide/day you're on
- Exact error message (copy/paste, don't paraphrase)
- What command you ran
- Relevant CloudWatch logs if available
- What you've already tried

### For AI Assistants (Claude Code)

When helping students:
0. **Prepare** - Read all the guides to be fully briefed
1. **Establish context** - Which guide? What's the goal?
2. **Get error details** - Actual messages, logs, console output
3. **Diagnose first** - Don't write code before understanding the problem
4. **Think incrementally** - One change at a time
5. **Verify understanding** - Explain what you think is wrong before fixing
6. **Keep it simple** - Avoid over-engineering solutions

**Remember**: Students are learning. The goal is to help them understand what went wrong and how to fix it, not just to make the error go away.

---

**Course**: AI in Production
**Instructor**: Ed Donner
**Platform**: Udemy
**Project**: Alex - Capstone for Weeks 3-4
**Last Updated**: January 2025
