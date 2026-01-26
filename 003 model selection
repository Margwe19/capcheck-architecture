# ADR 003: Model Selection Strategy

## Status

Accepted

## Context

Different tasks in the verification pipeline have different requirements:

| Task | Latency Need | Reasoning Depth | Cost Sensitivity |
|------|--------------|-----------------|------------------|
| OCR / Text extraction | Fast | Low | High |
| Content classification | Fast | Medium | High |
| Claim extraction | Medium | Medium | Medium |
| Synthesis / Verdict | Slow OK | High | Low |
| Fact-checking search | Medium | N/A (API) | Medium |

Using a single model (e.g., Claude Sonnet for everything) would be:
- **Too expensive** - Sonnet costs ~15x more than Haiku
- **Too slow** - Sonnet latency adds up across multiple calls
- **Overkill** - OCR doesn't need advanced reasoning

## Decision

Route tasks to the optimal model based on requirements:

| Task | Model | Rationale |
|------|-------|-----------|
| OCR / Image description | Claude Haiku 3.5 | Fast, cheap, sufficient quality |
| Content classification | Claude Haiku 3.5 | Structured output, low latency |
| Claim extraction | Claude Haiku 3.5 | Pattern matching, not deep reasoning |
| Synthesis / Verdict | Claude Sonnet 4 | Requires nuanced judgment |
| Fact-checking | Perplexity Sonar Pro | Specialized for factual queries, #1 on SimpleQA |
| Audio transcription | OpenAI Whisper | Best-in-class for speech |
| AI image detection | Vision Transformer (ViT) | Self-hosted, specialized classifier |

### Cost Analysis

**Before optimization (all Sonnet):**
```
OCR: $0.003 × 1 call = $0.003
Classification: $0.003 × 1 call = $0.003
Claim extraction: $0.003 × 1 call = $0.003
Synthesis: $0.003 × 1 call = $0.003
Perplexity: $0.005 × 1 call = $0.005
Total: ~$0.017 per simple verification
```

**After optimization (mixed models):**
```
OCR (Haiku): $0.00025 × 1 call = $0.00025
Classification (Haiku): $0.00025 × 1 call = $0.00025
Claim extraction (Haiku): $0.00025 × 1 call = $0.00025
Synthesis (Sonnet): $0.003 × 1 call = $0.003
Perplexity: $0.005 × 1 call = $0.005
Total: ~$0.009 per simple verification
```

**Savings: ~47% cost reduction** while maintaining verdict quality.

### Implementation

```typescript
// backend/services/verification/v2/config.ts

export const MODELS = {
  // Fast, cheap tasks
  ocr: 'claude-3-5-haiku-20241022',
  classification: 'claude-3-5-haiku-20241022',
  claimExtraction: 'claude-3-5-haiku-20241022',
  
  // Complex reasoning
  synthesis: 'claude-sonnet-4-20250514',
  
  // Specialized APIs
  factCheck: 'sonar-pro', // Perplexity
  transcription: 'whisper-1', // OpenAI
} as const;
```

### Why Perplexity Sonar Pro for Fact-Checking?

Perplexity's Sonar Pro model achieved **0.858 F-score on SimpleQA**, the leading benchmark for factual accuracy. This outperforms:
- GPT-4: 0.38
- Claude 3 Opus: 0.47
- Gemini 1.5: 0.42

For a fact-checking system, using the most accurate factual model is worth the API cost.

## Consequences

### Positive

- **47% cost reduction** on typical verifications
- **Faster pipeline** - Haiku responds in ~200ms vs Sonnet's ~2s
- **Quality maintained** - Verdict quality unchanged (Sonnet still does synthesis)
- **Specialized tools** - Best model for each task

### Negative

- **More complexity** - Multiple API clients to maintain
- **Version management** - Must track model deprecations across providers
- **Debugging harder** - Issues could be in any model

### Monitoring

Track per-model metrics:

```typescript
interface ModelMetrics {
  model: string;
  task: string;
  latencyMs: number;
  tokenCount: number;
  cost: number;
  success: boolean;
}
```

Weekly review of cost distribution to catch regressions.

## Alternatives Considered

### 1. All Sonnet

Rejected. Too expensive at scale. OCR doesn't benefit from advanced reasoning.

### 2. All Haiku

Rejected. Synthesis quality suffered in testing. Verdicts were less nuanced.

### 3. Fine-tuned single model

Considered for future. Would require training data and ongoing maintenance. Current approach is more flexible.

### 4. Open-source models (Llama, Mistral)

Considered. Self-hosting would reduce costs further but adds infrastructure complexity. May revisit at scale.

## Future Considerations

- **Claude Haiku 4** - When released, evaluate for claim extraction upgrade
- **Perplexity Pro** - Monitor for improved fact-checking models
- **Self-hosted ViT** - Already deployed on Fly.io for AI detection
- **Batch processing** - Could reduce costs further for non-real-time verifications
