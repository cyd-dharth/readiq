# ReadIQ — AI-Powered Book Intelligence Platform

> Thoughtful summaries, honest critiques, and cross-book connections for books that matter.

**Live site:** [readiq.app](https://readiq.app)

| Service | Repository | Deploy |
|---|---|---|
| Backend (FastAPI) | [github.com/cyd-dharth/readiq-backend](https://github.com/cyd-dharth/readiq-backend) | GCP Cloud Run |
| AI Agents Pipeline | [github.com/cyd-dharth/readiq-agents](https://github.com/cyd-dharth/readiq-agents) | GCP Cloud Run |
| Frontend (Astro) | [github.com/cyd-dharth/readiq-frontend](https://github.com/cyd-dharth/readiq-frontend) | Vercel |

---

## What it does

ReadIQ generates structured book intelligence from a title, PDF, or EPUB:

- **Hierarchical summaries** — a one-paragraph TL;DR, a full narrative summary, and per-chapter breakdowns with rolling context continuity
- **Sourced critique and support** — books and articles that intellectually oppose or reinforce the author's arguments, each with a paraphrased insight and a verified reference URL
- **RAG-powered chat** — a contextual Q&A grounded strictly on derived content, with hybrid keyword and vector retrieval
- **Reader comments** — a moderated comment section with honeypot spam protection

---

## Architecture

```
Browser
   |
   v
Astro SSR Frontend (Vercel)
   |
   | HTTPS
   v
FastAPI Backend (GCP Cloud Run) ──── Postgres + pgvector (Supabase)
   |                    |
   |                    | Redis job
   |                    v
   |           Agents Service (GCP Cloud Run)
   |                    |
   |                    |── Ingest (PDF / EPUB)
   |                    |── Summarize (LLM)
   |                    |── Research (Tavily + LLM)
   |                    |── Embed (pgvector)
   |                    |── Chat (RAG + LLM)
   v
Cloudflare DNS + Proxy
```

Three independently deployed services with a clean separation of concerns:

- **Backend** owns the public API, user auth, job dispatch, and rate limiting. All secrets stay here.
- **Agents service** owns all AI logic: LLM calls, embeddings, web search, PDF and EPUB parsing. No public routes except through the backend.
- **Frontend** owns the user experience. Reads from the backend API. Upload UI gated by an environment variable.

---

## Technical highlights

### Agentic pipeline with concurrent sub-agents

The generation pipeline runs as an orchestrator driving multiple specialised stages. Chapter summarization runs all chapters concurrently via `asyncio.gather` with a two-sentence rolling context handoff between chapters, maintaining narrative continuity without growing the context window. The research stage spawns two parallel sub-agents (critique and support) each running four targeted web searches, deduplicating by URL, scoring results on relevance and quality, and retrying with reformulated queries if the minimum threshold is not met.

### Four-tier PDF chapter detection

Rather than calling the LLM for everything, the ingest pipeline uses the cheapest reliable signal first:

1. **TOC page numbers** — extract page numbers from the table of contents, calibrate the offset against actual PDF page indices (books restart numbering after front matter), verify each boundary with fuzzy title matching
2. **TOC titles without page numbers** — extract chapter titles from the TOC, search each page's first lines for a fuzzy match
3. **Generic heuristic patterns** — detect headings by length, casing, and common chapter keywords
4. **LLM full-page analysis** — send compact page-start lines to the LLM only as a last resort

Each tier falls through gracefully. If all tiers fail the whole book becomes one chapter rather than failing the job.

### Provider-agnostic LLM and embedding clients

A common async interface (`complete(system, user, max_tokens) -> str`) with lazy-imported implementations for Anthropic Claude, OpenAI GPT, and Google Gemini. Switching providers is a single environment variable change with zero code changes. The same pattern applies to embeddings: Gemini `text-embedding-004` and OpenAI `text-embedding-3-small` both output 1536 dimensions matching the pgvector column, switchable via `EMBEDDING_PROVIDER`.

### Hybrid RAG retrieval

Chat requests are routed before hitting the vector store. Keyword matching classifies the question type (critique, support, summary, or default) and fetches the appropriate context: sourced items from the sources table, the full summary directly, or the top-N nearest chapter embeddings via pgvector cosine similarity. The LLM answer is grounded strictly on derived content, never on general model knowledge about the book, which keeps the platform copyright-safe.

### EPUB ingestion without LLM

EPUB files expose chapter structure explicitly in their spine and NCX table of contents. The ingest pipeline reads the spine order, maps TOC titles to spine items, extracts clean text via BeautifulSoup, merges short spine items into the preceding chapter, and filters non-chapter content (preface, footnotes, license pages) by title pattern matching. No LLM call needed for well-formed EPUBs.

### Idempotent pipeline stages

Every stage deletes and recreates derived rows before inserting. Re-running a book at any stage is safe and produces no duplicates. Failed stages do not block subsequent ones: embed failure does not stop research, research failure does not stop publication.

---

## Stack

| Layer | Technology |
|---|---|
| Backend API | Python, FastAPI, asyncpg, Pydantic v2 |
| AI pipeline | Anthropic Claude, Google Gemini, OpenAI |
| Embeddings | Gemini text-embedding-004, pgvector |
| Web search | Tavily |
| Document parsing | pypdf, ebooklib, BeautifulSoup |
| Database | PostgreSQL, pgvector (Supabase) |
| Job queue | Redis (local), direct HTTP (production) |
| Auth | JWT, bcrypt |
| Frontend | Astro 5, TypeScript, Tailwind CSS v4 |
| Infrastructure | GCP Cloud Run, Vercel, Cloudflare |
| CI/CD | GitHub Actions, Docker, Artifact Registry |

---

## Services

### Backend — `readiq-backend`

Public REST API and job controller. Handles user registration and login, book creation, generation dispatch, chat session management with daily rate limiting, source retrieval, and reader comments with honeypot spam protection. All AI provider keys live in the agents service; the backend holds only JWT secrets and database credentials.

Key endpoints:

```
POST   /api/auth/register
POST   /api/auth/login
GET    /api/books
GET    /api/books/{slug}
POST   /api/books
POST   /api/books/{id}/generate
POST   /api/books/{id}/chat
GET    /api/books/{id}/chat/history
GET    /api/books/{id}/sources
GET    /api/books/{id}/comments
POST   /api/books/{id}/comments
GET    /health
```

### Agents — `readiq-agents`

The AI pipeline and internal chat API. Runs as two processes from the same codebase: a Redis worker for background generation jobs and an internal FastAPI server for synchronous chat requests. The service is locked to internal GCP ingress and never receives public traffic directly.

Pipeline stages in order:

```
1. Ingest     PDF four-tier chapter detection or EPUB spine extraction
2. Summarize  Hierarchical LLM summarization with rolling context
3. Embed      Chapter summary embeddings written to pgvector
4. Research   Concurrent critique and support sub-agents via Tavily
```

### Frontend — `readiq-frontend`

Server-rendered Astro site with Tailwind CSS v4. All pages fetch fresh data on each request. Interactive features (auth, chat, comment submission) use vanilla TypeScript script blocks with no JavaScript framework. The upload form is gated by `PUBLIC_ENABLE_UPLOAD` and disabled in production. Future features (chat, comments, sources) are visible as disabled placeholders and activated progressively.

---

## Database schema

```
books           source metadata, generated summaries, publish gate
chapters        per-chapter summaries and pgvector embeddings
sources         sourced critiques and supporting works with URLs
users           email and bcrypt-hashed password
chat_sessions   one session per user per book
chat_messages   conversation history with role and content
comments        reader responses with moderation status
```

---

## Deployment

All three services deploy automatically on push to main via GitHub Actions.

```
Push to main
   |
   v
GitHub Actions: build Docker image
   |
   v
Push to GCP Artifact Registry
   |
   v
Deploy to Cloud Run (zero-downtime revision swap)
```

Frontend deploys to Vercel via the Vercel GitHub integration.

Infrastructure:

```
readiq.app          Vercel (Astro SSR, global CDN)
api.readiq.app      GCP Cloud Run via Cloudflare proxy
Agents service      GCP Cloud Run (internal ingress only)
Database            Supabase (PostgreSQL + pgvector)
DNS                 Cloudflare
```

---

## Why not LangChain

LangChain was deliberately excluded. The native Anthropic, OpenAI, and Google SDKs handle everything this pipeline needs. Building the provider abstraction, agentic orchestration, and retrieval from primitives demonstrates understanding of what the frameworks do internally rather than treating them as black boxes. It also produces cleaner, more debuggable code with explicit control over every LLM call.

---

## Local development

Each service runs independently. Start infrastructure first:

```bash
# From readiq-backend
docker compose up -d
psql -d readiq -f db/schema_website_v1.sql
```

Then run each service:

```bash
# Backend
cd readiq-backend && python run.py

# Agents worker
cd readiq-agents && python -m app.worker

# Agents chat server
cd readiq-agents && python run_api.py

# Frontend
cd readiq-frontend && npm install && npm run dev
```

Each service has a `.env.example` with all required variables documented.

---

## What is next

- Observability layer with per-pipeline-run tracing, token cost, and quality metrics via Langfuse
- Model evaluation harness with RAGAS for faithfulness and hallucination scoring
- Admin moderation UI for comments and source verification
- YouTube video generation pipeline (audio via ElevenLabs voice clone, thumbnail generation, YouTube Data API upload)

---

## Author

Built by Siddhartha — AI/ML/API Engineer and Team Lead at GIST Impact, MS(R) Sustainable Energy Engineering from IIT Kanpur (CPI 9.0, Best Thesis Award 2024).

[LinkedIn](https://linkedin.com/in/siddharth789) · [readiq.app](https://readiq.app)
