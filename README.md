# DocuShield — AI Service

The **AI/RAG microservice** for DocuShield, an AI co-pilot that triages legal contract risk. This service extracts clauses from uploaded contracts, embeds them, and classifies risk using a rule-engine-first, LLM-fallback pipeline.

This is a **standalone Python service** — independently deployable from the [DocuShield Backend](#) (NestJS gateway). The two never call each other directly; all communication happens through a Redis job queue. This is a deliberate architectural boundary: the Gateway can stay fast and available even if this service is slow, retrying, or temporarily down.

## Tech Stack

- **Framework:** FastAPI
- **Extraction/Chunking:** LangChain (clause-boundary-aware chunking)
- **Embeddings:** Google Gemini
- **Classification:** Rule-based engine (deterministic, runs first) → Groq (Llama 3.x) for ambiguous clauses only
- **RAG:** pgvector similarity search over Supabase Postgres
- **Job Queue:** BullMQ (Python client), consuming jobs enqueued by the NestJS gateway via Redis
- **Testing:** pytest, load-tested with k6/Artillery from the gateway side

## Why This Pipeline Order

1. **Cache check first** (Redis) — a cache hit never touches an LLM
2. **Rule engine second** — deterministic pattern matching (liability caps, renewal windows, indemnification) handles the majority of clauses for free
3. **LLM only for ambiguous clauses** — this is the single biggest lever for staying inside Groq's free-tier limits
4. **Structured JSON output only** — every classification returns a strict schema (`risk_level`, `confidence`, `reasoning`, `category`), never free-text

## Security

- **Prompt-injection defense, two layers:** input sanitization *and* delimited-context prompting — contract text is attacker-influenceable by design (a counterparty could embed an instruction inside a clause), so both layers are required, not optional
- **Circuit breaker** on Groq/Gemini calls — if the provider is down or rate-limited, the system degrades to rule-engine-only scoring instead of failing the whole job

## Features

- 📄 PDF/DOCX extraction with clause-boundary-aware chunking
- 🧬 Gemini embeddings stored in pgvector for RAG
- ⚖️ Deterministic rule engine for common clause patterns
- 🤖 LLM classification (Groq) gated behind ambiguity + cache checks
- 🛡️ Prompt-injection sanitization + delimited-context prompting
- 🔄 BullMQ worker consuming jobs from the Gateway's Redis queue
- 🚦 Circuit breaker for graceful degradation when the LLM provider is unavailable

## Getting Started

```bash
git clone https://github.com/<org>/docushield-ai-service.git
cd docushield-ai-service
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
python -m app.main
```

## Environment Variables

| Variable | Description |
|---|---|
| `REDIS_URL` | Same Redis instance the Gateway uses for its job queue |
| `DATABASE_URL` | Supabase Postgres **pooled** connection string (same DB as the Gateway) |
| `GROQ_API_KEY` | Groq API key for LLM classification |
| `GEMINI_API_KEY` | Google Gemini API key for embeddings |

## Project Structure
app/
├── api/v1/ # internal endpoints (health, status)
├── core/ # config, correlation-id-aware logging
├── ingestion/ # loaders, chunking, embeddings
├── classification/ # rule_engine.py, llm_classifier.py, schemas.py
├── rag/ # retriever, prompt templates
├── workers/ # bullmq_consumer.py — the job queue worker
├── security/ # prompt_injection.py
└── main.py


## Running the Worker

```bash
python -m app.workers.bullmq_consumer
```

Consumes jobs from the `ingestion` queue — the same queue name the Gateway's producer enqueues to. Payload shape is a contract, verified by the producer-consumer contract test in the Gateway repo.

## Testing

```bash
pytest
```

## Team

Built by **Shanza, Zayyam, and Annas** — work split so every member touches backend, frontend, AI, and testing. See the [Team Work Division doc](#) for the full breakdown.
