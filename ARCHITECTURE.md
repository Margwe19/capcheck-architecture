# System Architecture

Full system architecture and data flow for the CapCheck verification platform.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CapCheck System Architecture                         │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   iOS App   │     │   Web App   │     │  Chrome Ext │
    │  (SwiftUI)  │     │  (Svelte 5) │     │  (planned)  │
    └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
           │                   │                   │
           └───────────────────┼───────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Hasura GraphQL    │
                    │   Engine + Auth     │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │PostgreSQL│    │ Fastify  │    │  Actions │
        │   Data   │    │  Server  │    │ Handlers │
        └──────────┘    └────┬─────┘    └──────────┘
                             │
                             ▼
                    ┌─────────────────────┐
                    │   V2 Orchestrator   │
                    │     (LangGraph)     │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   Claude    │     │  Perplexity │     │  External   │
    │ Sonnet/Haiku│     │  Sonar Pro  │     │   APIs      │
    └─────────────┘     └─────────────┘     └─────────────┘
                                                   │
                                    ┌──────────────┼──────────────┐
                                    │              │              │
                                    ▼              ▼              ▼
                              ┌─────────┐   ┌─────────┐   ┌─────────┐
                              │ Whisper │   │   ViT   │   │ Google  │
                              │ (Audio) │   │(AI Det) │   │  APIs   │
                              └─────────┘   └─────────┘   └─────────┘
```

## Component Responsibilities

| Component | Purpose | Technology |
|-----------|---------|------------|
| **iOS App** | Native mobile client | Swift, SwiftUI |
| **Web App** | Browser-based client | Svelte 5, SvelteKit 2, Vite |
| **Hasura** | GraphQL API, auth, permissions, subscriptions | Hasura GraphQL Engine |
| **PostgreSQL** | User data, verification history, caching | PostgreSQL 15 |
| **Fastify** | Action handlers, business logic | Node.js + Fastify |
| **V2 Orchestrator** | Multi-agent verification pipeline | LangGraph StateGraph |
| **Claude** | OCR, classification, synthesis, reasoning | Anthropic API |
| **Perplexity** | Real-time fact-checking with citations | Perplexity API |
| **Whisper** | Audio transcription | OpenAI API |
| **ViT** | AI-generated image detection | Self-hosted on Fly.io |

## Data Flow

### Request Lifecycle

```
1. Client submits content (image/video/audio/URL)
      │
      ▼
2. Hasura validates auth, routes to Fastify action
      │
      ▼
3. V2 Orchestrator processes through LangGraph pipeline:
      │
      ├─► Intake: OCR, transcription, URL extraction
      ├─► Classify: Stakes, satire detection, AI relevance
      ├─► Strategy: Select specialists, set thresholds
      ├─► Specialists: Parallel evidence gathering
      ├─► Synthesis: Evidence-only verdict determination
      └─► Output: Format and return
      │
      ▼
4. Results stored in PostgreSQL via Hasura
      │
      ▼
5. Client receives structured verification result
```

### Verification Pipeline Detail

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         V2 Verification Pipeline                             │
└─────────────────────────────────────────────────────────────────────────────┘

  Input                                                                Output
    │                                                                     ▲
    ▼                                                                     │
┌─────────┐   ┌──────────┐   ┌──────────┐   ┌───────────────┐   ┌──────────┐
│ INTAKE  │──▶│ CLASSIFY │──▶│ STRATEGY │──▶│  SPECIALISTS  │──▶│SYNTHESIS │
│         │   │          │   │          │   │   (parallel)  │   │          │
│ • OCR   │   │ • Satire │   │ • Agent  │   │ ┌───────────┐ │   │ • Verdict│
│ • Audio │   │ • Stakes │   │   select │   │ │web_search │ │   │ • Conf.  │
│ • URL   │   │ • AI rel │   │ • Thresh │   │ │fact_check │ │   │ • Sources│
└─────────┘   └────┬─────┘   └──────────┘   │ │ai_detect  │ │   └──────────┘
                   │                        │ │claim_extr │ │
                   │                        │ └───────────┘ │
                   ▼                        └───────────────┘
              ┌──────────┐
              │  OUTPUT  │  ◄─── Short-circuit for satire
              └──────────┘
```

## Database Schema (Simplified)

```sql
-- Core verification result
CREATE TABLE verifications (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    input_type TEXT, -- 'image', 'video', 'audio', 'url'
    input_hash TEXT, -- perceptual hash for deduplication
    verdict TEXT, -- 'TRUE', 'FALSE', 'MISLEADING', 'UNCERTAIN', 'SATIRE'
    confidence DECIMAL,
    summary TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Evidence sources
CREATE TABLE sources (
    id UUID PRIMARY KEY,
    verification_id UUID REFERENCES verifications(id),
    url TEXT,
    title TEXT,
    credibility_tier INTEGER, -- 1-5
    excerpt TEXT
);

-- Extracted claims
CREATE TABLE claims (
    id UUID PRIMARY KEY,
    verification_id UUID REFERENCES verifications(id),
    claim_text TEXT,
    sub_verdict TEXT,
    sub_confidence DECIMAL
);
```

## Caching Strategy

### Perceptual Image Hashing

Same or similar images return cached results without re-verification:

```
1. Compute perceptual hash of input image
2. Query for existing verifications within hamming distance threshold
3. If match found and < 24 hours old, return cached result
4. Otherwise, run full verification pipeline
```

### Cache Invalidation

- Time-based: Results expire after 24 hours for fast-moving stories
- Manual: Admin can invalidate specific verifications
- Source-based: If a key source updates, related verifications are flagged

## Infrastructure

### Production Environment

| Service | Host | Purpose |
|---------|------|---------|
| Web App | Cloudflare Pages | Static site hosting |
| API | Cloudflare Workers | Edge compute |
| Database | Neon | Serverless PostgreSQL |
| Hasura | Hasura Cloud | GraphQL engine |
| ViT Model | Fly.io | GPU inference |
| File Storage | Cloudflare R2 | Image/video storage |

### Local Development

```bash
# Start all services
docker-compose up -d

# Services available:
# - PostgreSQL: localhost:5432
# - Hasura Console: localhost:8080
# - Fastify API: localhost:3000
# - Web App: localhost:5173
```

## Security Considerations

- **Auth**: Hasura JWT validation with Clerk
- **Rate Limiting**: Per-user limits on verification requests
- **Input Validation**: Max file sizes, allowed MIME types
- **API Keys**: Stored in environment, never in code
- **Audit Log**: All verifications logged with user context
