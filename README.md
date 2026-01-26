# CapCheck AI Technical Architecture

> Public documentation for the CapCheck AI fact-checking platform. This repo contains architecture decisions, design patterns, and technical deep-dives without proprietary code.

## Overview

CapCheck is a fact-checking platform that uses a **LangGraph-based multi-agent pipeline** to verify claims in real-time. The system combines:

- **Claude** (Sonnet for reasoning, Haiku for classification/OCR)
- **Perplexity Sonar Pro** for fact-checking (#1 on SimpleQA benchmark, 0.858 F-score)
- **Vision Transformer** for AI-generated image detection
- **OpenAI Whisper** for audio transcription

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CapCheck V2 Verification Pipeline                         │
└─────────────────────────────────────────────────────────────────────────────┘

  Request                                                              Response
     │                                                                     ▲
     ▼                                                                     │
┌─────────┐   ┌──────────┐   ┌──────────┐   ┌───────────────┐   ┌──────────┐
│ INTAKE  │──▶│ CLASSIFY │──▶│ STRATEGY │──▶│  SPECIALISTS  │──▶│SYNTHESIS │
│         │   │          │   │          │   │   (parallel)  │   │          │
│ • OCR   │   │ • Satire │   │ • Agent  │   │ ┌───────────┐ │   │ • Verdict│
│ • Audio │   │ • Stakes │   │   select │   │ │web_search │ │   │ • Conf.  │
│ • URL   │   │ • AI rel │   │ • Thresh │   │ │fact_check │ │   │ • Review │
└─────────┘   └────┬─────┘   └──────────┘   │ │ai_detect  │ │   └──────────┘
                   │                        │ │claim_extr │ │
                   │ satire?                │ └───────────┘ │
                   ▼                        └───────────────┘
              ┌──────────┐
              │  OUTPUT  │  ◀─── Short-circuit for satire
              │ (format) │
              └──────────┘
```

## Key Technical Patterns

### 1. Evidence-Only Synthesis

All verdicts are determined from retrieved evidence only. The LLM is explicitly constrained from using training knowledge to prevent hallucinations.

→ See [decisions/001_EVIDENCE_ONLY_SYNTHESIS.md](decisions/001_EVIDENCE_ONLY_SYNTHESIS.md)

### 2. Confidence Calibration

The system detects when LLMs output default confidence values (like 60%) and recalculates based on source quality, quantity, and consensus.

→ See [decisions/002_CONFIDENCE_CALIBRATION.md)](decisions/002_CONFIDENCE_CALIBRATION.md)

### 3. Model Selection Strategy

Different models for different tasks based on latency, cost, and capability requirements.

→ See [decisions/003_MODEL_SELECTION.md](decisions/003_MODEL_SELECTION.md)

### 4. Graceful Degradation

Automatic fallback chain when services fail: V2 → V1, Perplexity → Brave Search + Claude.

→ See [decisions/04_GRACEFUL_DEGRADATION.md](decisions/04_GRACEFUL_DEGRADATION.md)

## Documentation

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Full system architecture and data flow |
| [LLM_PATTERNS.md](LLM_PATTERNS.md) | LLM integration patterns and prompts |
| [decisions/](decisions/) | Architecture Decision Records (ADRs) |

## Tech Stack

**Backend:** TypeScript, Node.js, Fastify, LangGraph, PostgreSQL, Hasura GraphQL

**AI/ML:** Claude Sonnet 4, Claude Haiku 3.5, Perplexity Sonar Pro, OpenAI Whisper, Vision Transformer

**Frontend:** iOS (Swift/SwiftUI), Web (Svelte 5/SvelteKit), Chrome Extension (planned)

**Infrastructure:** Docker, Fly.io, Cloudflare

## Performance

| Metric | Value |
|--------|-------|
| Cost per verification | $0.025-0.045 |
| Latency (P95) | 40-60s full pipeline |
| Caching | Database-backed with perceptual image hashing |

## Why This Architecture?

Fact-checking requires **accuracy over speed** and **transparency over black-box verdicts**. Key design principles:

1. **Evidence-grounded** - Never trust LLM training data for factual claims
2. **Auditable** - Every verdict has traceable source citations
3. **Calibrated** - Confidence scores reflect actual evidence quality
4. **Resilient** - Graceful degradation when services fail

## Contact

**Aaron Kantrowitz**  
[aaronkantrowitz.com](https://aaronkantrowitz.com) | [LinkedIn](https://linkedin.com/in/aaronkantrowitz)

---

*This is a portfolio documentation repo. The CapCheck codebase is private.*
