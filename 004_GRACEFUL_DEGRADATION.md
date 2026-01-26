# ADR 004: Graceful Degradation

## Status

Accepted

## Context

CapCheck depends on multiple external services:
- Anthropic API (Claude)
- Perplexity API
- OpenAI API (Whisper)
- Google APIs (Vision, Fact Check)

Any of these can fail due to:
- Rate limiting
- Service outages
- Network issues
- API changes

A fact-checking system cannot simply fail when one service is down. Users expect a result, even if degraded.

## Decision

Implement a multi-level fallback chain with automatic degradation:

### Level 1: V2 Pipeline (Full Quality)

```
LangGraph orchestration → Perplexity fact-check → Claude synthesis
```

### Level 2: V1 Pipeline (Reduced Quality)

```
Sequential processing → Brave Search + Claude → Claude synthesis
```

### Level 3: Minimal Pipeline (Emergency)

```
Claude-only → Training knowledge → UNCERTAIN verdict
```

### Implementation

```typescript
// backend/services/verification/router.ts

export async function verify(input: VerificationInput): Promise<VerificationResult> {
  // Try V2 pipeline first
  try {
    return await runVerificationV2(input);
  } catch (error) {
    logDegradation('V2 failed', error);
    
    if (!FALLBACK_ENABLED) throw error;
  }
  
  // Fall back to V1
  try {
    return await runVerificationV1(input);
  } catch (error) {
    logDegradation('V1 failed', error);
  }
  
  // Emergency: return UNCERTAIN
  return {
    verdict: 'UNCERTAIN',
    confidence: 0.0,
    summary: 'Unable to verify due to service issues. Please try again.',
    degraded: true,
    degradationReason: 'All verification pipelines failed',
  };
}
```

### Service-Level Fallbacks

| Primary Service | Fallback | Quality Impact |
|-----------------|----------|----------------|
| Perplexity Sonar Pro | Brave Search + Claude | Lower factual accuracy |
| Claude Sonnet | Claude Haiku | Less nuanced verdicts |
| Claude Haiku | GPT-4o-mini | Different model behavior |
| OpenAI Whisper | AssemblyAI | Comparable quality |
| AI Detection (ViT) | Skip with flag | No AI detection |
| Google Fact Check | Skip with flag | No prior fact-checks |

### Circuit Breaker Pattern

Prevent cascade failures by tracking service health:

```typescript
interface ServiceHealth {
  service: string;
  consecutiveFailures: number;
  lastFailure: Date | null;
  isOpen: boolean; // true = skip this service
}

function shouldUseService(health: ServiceHealth): boolean {
  if (!health.isOpen) return true;
  
  // Half-open: try again after 60 seconds
  if (health.lastFailure && 
      Date.now() - health.lastFailure.getTime() > 60_000) {
    return true;
  }
  
  return false;
}
```

## Consequences

### Positive

- **Always returns a result** - Users never see "Service Unavailable"
- **Automatic recovery** - No manual intervention needed when services return
- **Transparent degradation** - Response includes `degraded: true` flag
- **Cost savings** - Fallbacks are often cheaper (Brave vs Perplexity)

### Negative

- **Quality variance** - Degraded results are less accurate
- **Complexity** - Multiple code paths to maintain
- **Testing burden** - Must test all fallback combinations

### User Communication

When degraded, the response includes:

```json
{
  "verdict": "LIKELY_FALSE",
  "confidence": 0.72,
  "degraded": true,
  "degradationReason": "Primary fact-check service unavailable",
  "summary": "Based on available sources... (Note: This verification used backup services)"
}
```

## Monitoring

Alert thresholds:

| Metric | Warning | Critical |
|--------|---------|----------|
| V2 failure rate | > 5% | > 20% |
| Perplexity errors | > 10/hour | > 50/hour |
| Degraded responses | > 10% | > 30% |

## Alternatives Considered

### 1. Fail fast with error

Rejected. Users expect a result. Showing an error page destroys trust.

### 2. Queue and retry later

Considered for future. Would need async notification system. Good for non-real-time use cases.

### 3. Cache recent verifications

Implemented separately. If we've verified the same image/URL recently, return cached result.

## Related Decisions

- [002-confidence-calibration.md](002-confidence-calibration.md) - Degraded results have lower confidence
- [003-model-selection.md](003-model-selection.md) - Fallback models are explicitly chosen
