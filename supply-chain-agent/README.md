# Supply Chain Agentic AI - Cloud Run Runtime

Production-oriented Python/FastAPI reference implementation for the **online Agentic AI runtime** of a supply-chain investigation application.

## Flow

```text
Browser
  |
  v
Microsoft Entra ID
  |
  | OAuth/OIDC token
  v
GCP Load Balancer / Cloud Armor
  |
  v
Apigee
  | JWT validation, quota, rate limiting, API governance
  v
Cloud Run / FastAPI
  |
  +--> Input Guardrails
  |
  +--> Agent Orchestrator
  |       |
  |       +--> Knowledge Agent --> Vertex AI Vector Search
  |       |
  |       +--> Inventory Agent --> Inventory API
  |       |
  |       +--> Order Agent --> Purchase Order / Transfer Order APIs
  |       |
  |       +--> Investigation Agent --> Gemini
  |
  +--> Output Guardrails
  |
  +--> Deterministic JSON Schema Validation
  |
  v
Browser
```

## Important production boundary

Apigee should perform gateway-level JWT validation. The application should still establish the authorization context and never trust arbitrary client-supplied role headers. In production, configure Apigee and the identity architecture so the backend receives a verifiable identity context.

The example enterprise tools in this repository are **read-only adapters**. Replace the mock implementations with real APIs.

## Local run

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload --port 8080
```

Open:

```text
http://localhost:8080/docs
```

Example request:

```bash
curl -X POST http://localhost:8080/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "question": "Why did inventory for ABC123 drop by 35% in Plant 4 during the last seven days?"
  }'
```

## GCP configuration

Set:

- `GOOGLE_CLOUD_PROJECT`
- `VERTEX_AI_LOCATION`
- `VERTEX_AI_EMBEDDING_MODEL`
- `VERTEX_AI_MODEL`
- `VECTOR_SEARCH_INDEX_ENDPOINT`
- `VECTOR_SEARCH_DEPLOYED_INDEX_ID`

The Vector Search adapter is intentionally isolated in `app/rag/vector_search.py` so the exact deployed-index implementation can be configured without changing the agent layer.

## Cloud Run

```bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/supply-chain-agent

gcloud run deploy supply-chain-agent \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/supply-chain-agent \
  --region us-central1 \
  --platform managed \
  --no-allow-unauthenticated
```

For a production deployment, prefer Artifact Registry, Secret Manager, least-privilege service accounts, VPC egress controls, centralized logging, tracing, and a private service path from Apigee to Cloud Run where applicable.
