# Contoso Creative Writer - AI Agent Instructions

## Architecture Overview

This is a **multi-agent AI system** that creates well-researched, product-specific articles through a coordinated workflow of 4 specialized agents:

- **Research Agent**: Uses Azure AI Agent Service with Bing Grounding Tool for topic research
- **Product Agent**: Uses Azure AI Search for semantic similarity search of product vector store
- **Writer Agent**: Combines research and product data into coherent articles
- **Editor Agent**: Provides feedback and quality control with retry logic

The system uses **FastAPI** (Python) backend with **React/TypeScript** frontend, deployed on **Azure Container Apps**.

## Critical Development Patterns

### Agent Orchestration Flow
All agents are orchestrated through `src/api/orchestrator.py` using a **streaming generator pattern**:
```python
@trace
def create(research_context, product_context, assignment_context):
    yield start_message("researcher")
    research_result = researcher.research(research_context, feedback)
    yield complete_message("researcher", research_result)
    # ... continues for each agent
```

### Prompty Framework Integration
- All agents use **Prompty** for prompt management (`.prompty` files in each agent directory)
- Tracing is enabled with `@trace` decorators for debugging
- Configuration uses `model_config` dict for Azure OpenAI deployment names

### Authentication Pattern
All Azure services use **DefaultAzureCredential** with managed identity:
```python
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()
```

## Key Integration Points

### Azure AI Agent Service (Research Agent)
- Uses connection string format: `"{location}.api.azureml.ms;{sub_id};{resource_group};{project_name}"`
- Requires BingGroundingTool with proper TPM quota (80+ for gpt-4o)
- Located in: `src/api/agents/researcher/researcher.py`

### Vector Search (Product Agent)
- Uses Azure AI Search with `VectorizedQuery` for semantic search
- Embeddings generated via Azure OpenAI `text-embedding-ada-002`
- Index name: `contoso-products` (configurable)
- Located in: `src/api/agents/product/product.py`

### Streaming Response Pattern
API uses `PromptyStream` for real-time agent workflow updates:
```python
return StreamingResponse(
    PromptyStream("create_article", create(task.research, task.products, task.assignment)),
    media_type="text/event-stream"
)
```

## Essential Commands

### Development Setup
```bash
# Backend (API)
cd src/api
python -m venv .venv
source .venv/bin/activate  # or .venv\Scripts\activate on Windows
pip install -r requirements.txt

# Frontend (Web)
cd src/web
npm install
npm run dev
```

### Azure Deployment
```bash
# Deploy complete infrastructure and applications
azd up

# Deploy infrastructure only
azd provision

# Deploy applications only
azd deploy
```

### Data Pipeline (Product Vector Store)
```bash
# Create search index from products.csv
cd data
python create-azure-search.py
```

## Configuration Management

### Environment Variables
Core variables defined in `azure.yaml` pipeline section:
- `AZURE_OPENAI_ENDPOINT` / `AZURE_OPENAI_NAME`
- `AZURE_OPENAI_DEPLOYMENT_NAME` (gpt-4o)
- `AZURE_OPENAI_4_EVAL_DEPLOYMENT_NAME` (gpt-4o-mini)
- `AZURE_SEARCH_ENDPOINT` / `AZURE_SEARCH_NAME`
- `AZURE_AI_PROJECT_NAME` (for Agent Service)
- `APPINSIGHTS_CONNECTIONSTRING`

### Deployment Models
- **Development**: Uses azd environments in `.azure/` directory
- **Container Apps**: Both API and Web deployed as containerized services
- **Infrastructure**: Terraform templates in `infra/` directory

## Evaluation & Monitoring

### Built-in Evaluators
Located in `src/api/evaluate/evaluators.py`:
- `RelevanceEvaluator`, `GroundednessEvaluator`, `FluencyEvaluator`, `CoherenceEvaluator`
- `ContentSafetyEvaluator` for harmful content detection
- Custom `FriendlinessEvaluator` using Prompty

### Telemetry Integration
- OpenTelemetry with FastAPI instrumentation
- Prompty tracing for agent workflow debugging
- Application Insights for production monitoring

## Common Debugging Patterns

### Agent Workflow Issues
- Use `@trace` decorator and check Prompty traces
- Verify Azure OpenAI deployment names match environment variables
- Check BingGroundingTool quota limits for research agent

### Vector Search Problems
- Verify `contoso-products` index exists in Azure AI Search
- Check embedding model deployment (`text-embedding-ada-002`)
- Validate search endpoint and credentials

### CORS Configuration
CORS origins auto-configured for Codespaces and Container Apps:
```python
if code_space: 
    origins = [f"https://{code_space}-8000.app.github.dev", ...]
```

## Testing Strategies

### Local Testing
```python
# Test orchestrator directly
from orchestrator import test_create_article
test_create_article(research_context, product_context, assignment_context)
```

### Evaluation Pipeline
Background evaluation runs automatically after article creation:
```python
evaluate_article_in_background(
    research_context, product_context, assignment_context,
    research, products, article
)
```
