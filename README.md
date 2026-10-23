# Alex

**Agentic Learning Equities eXplainer** is a multi-agent SaaS for portfolio analysis: reports, charts, retirement projections, and market research on AWS.

## Credit

This project is based on **[Alex](https://github.com/ed-donner/alex)** by **[Ed Donner](https://github.com/ed-donner)**, the capstone for Weeks 3 and 4 of the Udemy course [AI in Production](https://www.udemy.com/). Original copyright is Ed Donner's (see [LICENSE](LICENSE)). This repository is a student snapshot of that work, with local deployment changes (including the researcher running as a Lambda container image with a Function URL).

Course questions and contributions belong with Ed: [ed-donner/alex](https://github.com/ed-donner/alex) or ed@edwarddonner.com.

## What it does

Signed-in users manage holdings in a Next.js app (Clerk). The FastAPI backend enqueues an analysis job. A **Planner** Lambda orchestrates **Tagger**, **Reporter**, **Charter**, and **Retirement** agents (OpenAI Agents SDK, Bedrock). A separate **Researcher** agent browses the web on a schedule, embeds documents, and stores vectors so later analyses can pull market context.

Infrastructure is split into independent Terraform directories (`terraform/2_sagemaker` through `terraform/8_enterprise`) with local state. Step-by-step deploy notes are in `guides/`.

## Architecture components

C4 container view of production. Internals sit inside the Alex boundary; people and vendors sit outside.

```mermaid
---
config:
  theme: default
  themeVariables:
    background: "#ffffff"
---
flowchart TB
    user[Investor]
    clerk[Clerk<br/>auth]
    bedrock[AWS Bedrock<br/>Nova Pro or gpt-oss]
    polygon[Polygon.io<br/>market prices]
    web[Public web<br/>Playwright MCP]

    subgraph alex["Alex"]
        cf["CloudFront + S3<br/>Next.js static UI"]
        apigw["API Gateway HTTP<br/>FastAPI Lambda"]
        sqs["SQS<br/>analysis jobs"]
        planner["Planner Lambda<br/>orchestrator"]
        specialists["Tagger Reporter Charter Retirement<br/>Lambda agents"]
        researcher["Researcher Lambda<br/>Function URL + Playwright"]
        ingest["Ingest Lambda<br/>API Gateway REST"]
        sm["SageMaker serverless<br/>all-MiniLM-L6-v2 embeddings"]
        vectors["S3 Vectors<br/>research index"]
        aurora["Aurora Serverless v2<br/>PostgreSQL Data API"]
        scheduler["EventBridge Scheduler<br/>optional research trigger"]
    end

    user -->|HTTPS| cf
    user -->|sign in| clerk
    cf -->|JWT + REST| apigw
    apigw -->|Data API| aurora
    apigw -->|enqueue job| sqs
    sqs -->|invoke| planner
    planner -->|invoke| specialists
    planner -->|query vectors| vectors
    planner -->|prices| polygon
    specialists -->|reports charts tags| aurora
    specialists -->|Bedrock via LiteLLM| bedrock
    planner -->|Bedrock via LiteLLM| bedrock
    scheduler -->|HTTP /research/auto| researcher
    researcher -->|browse| web
    researcher -->|generate copy| bedrock
    researcher -->|POST /ingest| ingest
    ingest -->|embed| sm
    ingest -->|put vectors| vectors
```

## Request path

Portfolio analysis from the browser through the agent orchestra.

```mermaid
---
config:
  theme: default
  themeVariables:
    background: "#ffffff"
---
sequenceDiagram
    autonumber
    actor User
    participant UI as CloudFront Next.js
    participant Clerk
    participant API as FastAPI Lambda
    participant DB as Aurora Data API
    participant SQS as Analysis SQS
    participant Planner
    participant Agents as Specialist Lambdas
    participant Vec as S3 Vectors
    participant Bedrock

    User->>UI: Open app
    UI->>Clerk: Authenticate
    User->>UI: Request analysis
    UI->>API: POST job (Clerk JWT)
    API->>DB: Load portfolio and preferences
    API->>SQS: Send analysis job
    SQS->>Planner: Invoke orchestrator
    Planner->>DB: Read holdings
    Planner->>Vec: Retrieve research context
    Planner->>Agents: Invoke tagger reporter charter retirement
    Agents->>Bedrock: LLM tools or structured output
    Agents->>DB: Store reports and charts
    Planner->>DB: Mark job complete
    User->>API: Poll job status
    API->>DB: Return results
    API->>UI: Reports and chart JSON
```

## Deployment

Each `terraform/*` folder is applied on its own (local `terraform.tfstate`, no shared remote backend). Lambda zip/image packages are built with Docker; the frontend is a Next.js static export uploaded to S3 and served by CloudFront.

```mermaid
---
config:
  theme: default
  themeVariables:
    background: "#ffffff"
---
flowchart LR
    repo["Git repo<br/>guides backend frontend terraform"]
    uv["uv<br/>Python agents and API"]
    docker["Docker<br/>Lambda linux/amd64 packages"]
    tf["Terraform apply<br/>2_sagemaker to 8_enterprise"]
    ecr["ECR<br/>researcher image"]
    aws["AWS account<br/>Lambda SQS API Gateway CloudFront"]
    s3fe["S3 + CloudFront<br/>static UI"]
    data["Aurora + S3 Vectors<br/>SageMaker Bedrock"]

    repo --> uv
    repo --> docker
    repo --> tf
    docker --> ecr
    ecr --> aws
    tf --> aws
    tf --> s3fe
    aws --> data
    uv -->|scripts/run_local.py| repo
```

Apply order matches the guides: SageMaker embeddings, ingest + S3 Vectors, researcher, Aurora, agent Lambdas + SQS, frontend + API, then optional CloudWatch/Langfuse in enterprise.

## Repository layout

| Path | Role |
|------|------|
| `guides/` | Deploy walkthroughs (start here) |
| `backend/` | Planner, tagger, reporter, charter, retirement, researcher, ingest, API, shared database lib |
| `frontend/` | Next.js Pages Router + Clerk |
| `terraform/` | Independent stacks per guide |
| `scripts/` | Local run, frontend deploy, destroy helpers |

## License

MIT. Copyright (c) 2025 Ed Donner. See [LICENSE](LICENSE).
