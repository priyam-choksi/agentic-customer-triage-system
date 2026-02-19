# Agentic Customer Triage System

An end-to-end multi-agent RAG system for intelligent customer support triage. The system routes incoming queries to specialized domain agents — travel bookings (flights, hotels, car rentals, excursions) and banking/financial services — each backed by their own knowledge base and retrieval pipeline.

Built with LangGraph, LangChain, FastAPI, and Claude (Anthropic), with a suggested production architecture on AWS.

---

## What This Does

Most customer support systems either dump everything into one LLM or rely on rigid rule-based routing. This system does neither. It:

1. **Triages** the incoming query to the right domain agent
2. **Reformulates** the question into an optimized search query
3. **Retrieves** relevant context from a domain-specific vector store (RAG)
4. **Generates** a grounded answer using a specialized assistant
5. **Validates** the answer with a confidence score before surfacing it

The result is a modular, observable, and extensible support system where each agent has a clearly scoped job.

---

## Architecture

### Agent Pipeline

```
User Query
    │
    ▼
Triage Agent  ──────────────────────────────────────────────
    │                                                       │
    ▼                                                       ▼
Travel Domain                                     Banking Domain
    │                                                       │
    ├── Reformulation Agent                     ├── Reformulation Agent
    ├── RAG Search Agent (Qdrant)               ├── RAG Search Agent (ChromaDB)
    ├── Specialized Sub-Agents                  └── Validation Agent
    │     ├── Flight Booking                             │
    │     ├── Hotel Booking                              ▼
    │     ├── Car Rental                         Confidence Score +
    │     └── Excursion                          Source Attribution
    └── Sensitive Tool Confirmation
```

### Travel Domain (LangGraph State Graph)

The travel side uses a **LangGraph state machine** with conditional routing between agents and tool nodes. Safe tools (read operations like search) run automatically; sensitive tools (bookings, modifications) pause and require explicit user confirmation before execution.

- **Primary Assistant** → entry point, routes to specialized sub-agents
- **Specialized Assistants** → Flight, Hotel, Car Rental, Excursion
- **Tool Nodes** → safe tools (auto-execute) + sensitive tools (require approval)
- **Memory** → conversation state persisted via LangGraph checkpointer
- **Observability** → LangSmith for tracing agent decisions and tool usage

### Banking Domain (3-Agent Sequential Pipeline)

The banking side uses a **linear 3-agent chain** optimized for accuracy and confidence tracking:

1. **Reformulation Agent** → rewrites the raw customer question into a clean search query
2. **Search Agent** → retrieves top-k chunks from ChromaDB, generates a grounded answer
3. **Validation Agent** → scores answer confidence (0–100) and flags low-confidence responses for human review

---

## Project Structure

```
agentic-customer-triage-system/
│
├── app/                            # Core application modules
├── customer_support_chat/          # Customer support chat agents and pipeline
├── docs/                           # Documentation
├── graphs/                         # LangGraph agent state graphs
├── images/                         # Architecture diagrams and screenshots
├── mcp_server/                     # MCP server setup
├── scripts/                        # Utility and setup scripts
├── tests/                          # Test suite
├── vectorizer/                     # Embedding generation for Qdrant
│
├── main.py                         # Application entrypoint
├── docker-compose.yml              # Docker Compose (Qdrant + services)
├── Dockerfile                      # Container definition
├── Makefile                        # Common dev commands
├── tools.yaml                      # Tools configuration
├── TOOLBOX.md                      # Tools reference documentation
├── pyproject.toml                  # Project dependencies (Poetry)
├── requirements.txt                # Pip dependencies
├── uv.lock                         # uv lockfile
├── poetry.lock                     # Poetry lockfile
├── .python-version                 # Python version pin
├── .dev.env                        # Environment variable template
└── README.md
```

---

## Tech Stack

| Layer | Travel Domain | Banking Domain |
|---|---|---|
| Agent Framework | LangGraph + LangChain | Direct Claude API |
| LLM | OpenAI GPT | Claude Sonnet 4.5 |
| Vector DB | Qdrant | ChromaDB |
| Embeddings | LangChain default | HuggingFace all-MiniLM-L6-v2 |
| Orchestration | LangGraph state graph | Sequential pipeline |
| Observability | LangSmith | SQLite analytics |
| Backend | FastAPI | FastAPI |

---

## Setup & Running Locally

### Prerequisites

- Python 3.12+
- Poetry
- Docker + Docker Compose
- OpenAI API Key (travel domain)
- Anthropic API Key (banking domain)
- LangSmith API Key (optional, for tracing)

### 1. Clone and configure environment

```bash
git clone https://github.com/priyam-choksi/agentic-customer-triage-system.git
cd agentic-customer-triage-system

cp .dev.env .env
```

Edit `.env`:

```
OPENAI_API_KEY="your_openai_api_key"
ANTHROPIC_API_KEY="your_anthropic_api_key"
LANGCHAIN_API_KEY="your_langsmith_api_key"   # optional
```

### 2. Install dependencies

```bash
poetry install
```

Or with pip:

```bash
pip install -r requirements.txt
```

### 3. Set up the travel domain

Generate embeddings and start Qdrant:

```bash
poetry run python vectorizer/main.py
docker compose up qdrant -d
```

Qdrant dashboard available at: `http://localhost:6333/dashboard`

### 4. Set up the banking domain

Add your banking PDF files to `knowledge_base/`, then initialize the vector store:

```bash
poetry run python scripts/init_rag.py
```

This extracts text from PDFs, chunks them, generates embeddings, and builds the ChromaDB store.

### 5. Run the application

```bash
poetry run python main.py
```

Or use the Makefile shortcuts:

```bash
make run       # start the app
make test      # run tests
make docker    # build and start with Docker Compose
```

---

## API Reference

### `POST /api/query`
Routes a query through triage → appropriate domain pipeline.

**Request:**
```json
{
  "question": "I need to change my flight to Miami",
  "user_name": "Rep Name",
  "domain": "auto"
}
```

**Response:**
```json
{
  "success": true,
  "domain": "travel",
  "original_question": "I need to change my flight to Miami",
  "reformulated_query": "flight modification Miami booking update",
  "answer": "...",
  "confidence_score": 91,
  "sources": [...],
  "response_time_ms": 1823
}
```

### `GET /api/stats`
Returns dashboard analytics — total queries, average confidence, per-rep breakdowns, and low-confidence flags.

---

## AWS Deployment Architecture

Designed for production scale with a medallion data lake, containerized services on EKS, and fully automated CI/CD.

```
                        ┌─────────────────────────────────────┐
                        │           API Gateway                │
                        └────────────────┬────────────────────┘
                                         │
                        ┌────────────────▼────────────────────┐
                        │     Customer Support Chat (EKS)      │
                        │   Triage → Travel/Banking Agents     │
                        └──────┬──────────────────┬───────────┘
                               │                  │
              ┌────────────────▼───┐    ┌─────────▼──────────────┐
              │  Qdrant Search API │    │   ChromaDB Search API   │
              └────────────────────┘    └────────────────────────┘
                               │                  │
              ┌────────────────▼──────────────────▼───────────────┐
              │                    Data Layer                      │
              │  S3 (Bronze/Silver/Gold)  │  Redshift  │  Athena  │
              └────────────────────────────────────────────────────┘
                               │
              ┌────────────────▼───────────────────────────────────┐
              │              Data Processing                        │
              │   Airflow on EKS  │  Embedding Jobs (EC2)          │
              │   SageMaker / Bedrock (model training)              │
              └────────────────────────────────────────────────────┘
                               │
              ┌────────────────▼───────────────────────────────────┐
              │           Monitoring & Security                     │
              │  CloudWatch  │  Grafana + Redshift  │  Secrets Mgr │
              └────────────────────────────────────────────────────┘
              ┌────────────────────────────────────────────────────┐
              │                 CI/CD                               │
              │     CodePipeline + CodeBuild + ECR                  │
              └────────────────────────────────────────────────────┘
```

**Key design decisions:**

- **S3 medallion architecture** (Bronze → Silver → Gold) keeps raw, processed, and embedding-ready data cleanly separated
- **Qdrant on EKS** for the travel domain's vector search; ChromaDB for banking (swappable to Qdrant in production for consistency)
- **ElastiCache (Redis)** for session state — agents reference conversation history across turns without re-reading the DB
- **Airflow on EKS** orchestrates scheduled embedding refresh jobs so the knowledge base stays current
- **LangSmith** traces every agent decision in dev; **CloudWatch + Grafana** handle production monitoring

---

## My Contributions

This project merges two open-source reference implementations into a unified triage system with an extended architecture:

- Designed and implemented the **top-level triage router** that classifies incoming queries and dispatches to the appropriate domain pipeline
- Unified two independent backends into a **single FastAPI service** with shared request/response contracts
- Extended the banking pipeline with **source attribution** and **confidence-based escalation** logic for the manager dashboard
- Adapted the LangGraph travel agent graph for **modular integration** alongside the banking pipeline without state conflicts
- Built out the **MCP server layer** for tool orchestration across both domains
- Designed the **AWS production architecture** with clear separation between data engineering, ML, and serving layers
- Integrated **LangSmith observability** across both domains for end-to-end request tracing

---

## Roadmap

- [ ] Adaptive RAG on travel search tools (smarter vector filtering)
- [ ] Corrective RAG fallback to web search when VecDB confidence is low
- [ ] Self-RAG grounding checks before final response generation
- [ ] Unified vector store (migrate ChromaDB → Qdrant) for consistency
- [ ] Redis session memory across both domains
- [ ] Auth layer on the FastAPI backend

---

## License

Educational / demonstration project.