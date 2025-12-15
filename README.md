# Alex - the Agentic Learning Equities Explainer

## Multi-agent Enterprise-Grade SaaS Financial Planner

![Course Image](assets/alex.png)

_If you're looking at this in Cursor, please right click on the filename in the Explorer on the left, and select "Open preview", to view it in formatted glory._

### Welcome to The Capstone Project for Week 3 and Week 4!

#### The directories:

1. **guides** - this is where you will live - step by step guides to deploy to production
2. **backend** - the agent code, organized into subdirectories, each a uv project (as is the backend parent directory)
3. **frontend** - a NextJS React frontend integrated with Clerk
4. **terraform** - separate terraform subdirectories with state for each part
5. **scripts** - the final deployment script

#### Order of play:

##### Week 3

- On Week 3 Day 3, we will do 1_permissions and 2_sagemaker
- On Week 3 Day 4, we will do 3_ingest
- On Week 3 Day 5, we will do 4_researcher

##### Week 4

- On Week 4 Day 1, we will do 5_database
- On Week 4 Day 2, we will do 6_agents
- On Week 4 Day 3, we will do 7_frontend
- On Week 4 Day 4, we will do 8_enterprise

#### Keep in mind

- Please submit your community_contributions, including links to your repos, in the production repo community_contributions folder
- Regularly do a git pull to get the latest code
- Reach out in Udemy or email (ed@edwarddonner.com) if I can help! This is a gigantic project and I am here to help you deliver it!

---

## My Journey - Deployment Log

### Completed Deployment (December 2024)

Successfully deployed the full Alex Financial Planner stack on AWS following all 8 guides.

#### Infrastructure Deployed

| Guide | Component | AWS Services |
|-------|-----------|--------------|
| 1 | Permissions | IAM user, AlexAccess group, policies |
| 2 | SageMaker | Serverless embedding endpoint (all-MiniLM-L6-v2) |
| 3 | Ingestion | S3 Vectors bucket, Lambda, API Gateway |
| 4 | Researcher | App Runner, ECR, EventBridge scheduler |
| 5 | Database | Aurora Serverless v2 PostgreSQL with Data API |
| 6 | Agents | 5 Lambda agents (Planner, Tagger, Reporter, Charter, Retirement), SQS |
| 7 | Frontend | NextJS on S3/CloudFront, FastAPI Lambda, Clerk auth |
| 8 | Enterprise | CloudWatch dashboards, LangFuse observability |

#### Key Configurations

- **Region**: us-east-1 (Lambda, Aurora, etc.)
- **Bedrock Model**: us.amazon.nova-pro-v1:0 (Nova Pro via inference profile)
- **Bedrock Region**: us-west-2 (cross-region inference)
- **Observability**: LangFuse for agent tracing

#### Lessons Learned

1. **Bedrock Model Access**: Nova Pro requires requesting access in AWS Bedrock console. Claude Sonnet has strict rate limits - Nova Pro is recommended.

2. **Cross-Region Bedrock**: When using inference profiles (e.g., `us.amazon.nova-pro-v1:0`), ensure model access is granted in all regions the profile spans.

3. **LangFuse Integration**: The OpenAI Agents SDK tracing requires `OPENAI_API_KEY` even when using Bedrock models. The key doesn't need balance - it's only for trace export.

4. **Terraform State**: Each guide has independent local state. Deploy in order (2→8), destroy in reverse (8→2).

5. **Docker Requirement**: Lambda packaging requires Docker Desktop running for linux/amd64 builds.

6. **Aurora Costs**: Aurora Serverless v2 is the primary cost driver. Destroy when not actively working to save costs.

#### Teardown

All resources successfully destroyed on December 16, 2024:

```bash
# Destruction order (reverse of deployment)
cd terraform/8_enterprise && terraform destroy -auto-approve
cd terraform/7_frontend && terraform destroy -auto-approve
cd terraform/6_agents && terraform destroy -auto-approve
cd terraform/5_database && terraform destroy -auto-approve
cd terraform/4_researcher && terraform destroy -auto-approve
cd terraform/3_ingestion && terraform destroy -auto-approve
cd terraform/2_sagemaker && terraform destroy -auto-approve

# Cleanup orphaned CloudWatch log groups
aws logs delete-log-group --log-group-name /aws/lambda/alex-api
aws logs delete-log-group --log-group-name /aws/lambda/alex-researcher-scheduler
```

#### Known Issues Encountered

1. **ProxyTracerProvider Error**: LangFuse 3.3.4 has API incompatibility with Logfire 4.7.0's OpenTelemetry instrumentation. The `span.score()` call in reporter agent fails with `'ProxyTracerProvider' object has no attribute 'sampler'`. Non-critical - traces still work.

2. **S3 Bucket Deletion**: Frontend S3 bucket must be emptied before terraform destroy:
   ```bash
   aws s3 rm s3://alex-frontend-{account-id} --recursive
   ```

---

*Course: AI in Production by Ed Donner (Udemy)*