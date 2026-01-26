# LLM Integration Patterns

Patterns and strategies for integrating LLMs into the CapCheck verification pipeline.

## Model Selection

Different tasks require different models optimized for cost, latency, and capability.

| Task | Model | Rationale |
|------|-------|-----------|
| OCR / Image description | Claude Haiku 3.5 | Fast, cheap, sufficient quality |
| Content classification | Claude Haiku 3.5 | Structured output, low latency |
| Claim extraction | Claude Haiku 3.5 | Pattern matching, not deep reasoning |
| Synthesis / Verdict | Claude Sonnet 4 | Requires nuanced judgment |
| Fact-checking | Perplexity Sonar Pro | #1 on SimpleQA benchmark (0.858 F-score) |
| Audio transcription | OpenAI Whisper | Best-in-class for speech |
| AI image detection | Vision Transformer | Specialized classifier |

### Cost Impact

Using task-appropriate models reduces costs by ~47% compared to using Sonnet for everything.

## Prompt Patterns

### 1. Evidence-Only Synthesis

**Problem:** LLMs use training data that may be outdated or incorrect.

**Solution:** Explicitly constrain the model to use only provided evidence.

```
CRITICAL CONSTRAINT: Evidence-Only Synthesis

You must determine the verdict based ONLY on the evidence provided below.
Do NOT use your training knowledge to override or supplement this evidence.

If the evidence says a claim is false, your verdict must reflect that,
even if your training data suggests otherwise.

If evidence is insufficient, return UNCERTAIN or UNVERIFIABLE.
```

### 2. Verdict Validation Checklist

**Problem:** LLMs can output contradictory verdicts and summaries.

**Solution:** Force explicit validation before finalizing.

```
Before finalizing your verdict, verify:

1. DIRECTION CHECK: Does your verdict direction (true/false) match the majority of evidence?
2. CONFIDENCE CHECK: Is your confidence based on actual evidence quality, not a default value?
3. EVIDENCE CHECK: Is your verdict based only on retrieved evidence, not training data?
4. COHERENCE CHECK: Does your summary support your verdict, not contradict it?
5. SATIRE CHECK: Did you check if the source is a known satire site?
```

### 3. Structured Output with Zod

**Problem:** LLM outputs can be inconsistent and hard to parse.

**Solution:** Define schema and validate responses.

```typescript
const ClassificationSchema = z.object({
  contentType: z.enum(['factual_claim', 'opinion', 'satire', 'advertisement']),
  mediaType: z.enum(['screenshot', 'photo', 'video', 'social_media_post']),
  stakes: z.enum(['low', 'medium', 'high', 'critical']),
  aiDetectionRelevance: z.enum(['none', 'low', 'medium', 'high', 'critical']),
  origin: z.object({
    satireSource: z.boolean(),
    knownSource: z.string().nullable(),
  }),
});

// Parse and validate LLM response
const classification = ClassificationSchema.parse(JSON.parse(llmResponse));
```

### 4. Claim Extraction

**Problem:** Complex content contains multiple claims that must be verified independently.

**Solution:** Decompose into atomic, verifiable sub-claims.

```
Extract verifiable factual claims from this content.

For each claim:
- claim_text: The specific assertion (not opinion)
- type: 'statistical' | 'quote' | 'event' | 'scientific' | 'other'
- priority: 1-5 (5 = most important to verify)
- verifiable: true/false

Rules:
- Extract ONLY factual claims, not opinions or predictions
- Each claim should be independently verifiable
- Preserve exact quotes and numbers
- Flag claims that reference specific dates or statistics
```

### 5. Source Credibility Assessment

**Problem:** Not all sources are equally reliable.

**Solution:** Tier-based credibility scoring.

```
Assess source credibility using these tiers:

Tier 1 (Highest): .gov, .edu, peer-reviewed journals, wire services (AP, Reuters, AFP)
Tier 2: Major newspapers (NYT, WSJ, WaPo), established broadcast (BBC, NPR)
Tier 3: Regional newspapers, trade publications, established digital-native news
Tier 4: Blogs with editorial standards, expert personal sites
Tier 5 (Lowest): Social media, forums, unknown sources

For each source, provide:
- tier: 1-5
- reasoning: Why this tier was assigned
- potential_bias: Any known bias or perspective
```

## Anti-Patterns to Avoid

### 1. Trusting Default Confidence

**Problem:** LLMs output 60% confidence when uncertain.

**Wrong:**
```typescript
const confidence = llmResponse.confidence; // Often exactly 0.60
```

**Right:**
```typescript
const confidence = calibrateConfidence(llmResponse.confidence, evidenceSignals);
// See decisions/002-confidence-calibration.md
```

### 2. Single-Shot Verification

**Problem:** One LLM call can miss nuance or hallucinate.

**Wrong:**
```typescript
const verdict = await claude.verify(claim); // Single call
```

**Right:**
```typescript
// Multi-stage pipeline with evidence gathering
const evidence = await gatherEvidence(claim);
const verdict = await synthesizeFromEvidence(evidence);
```

### 3. Unbounded Context

**Problem:** Dumping everything into context degrades quality.

**Wrong:**
```typescript
const response = await claude.complete({
  prompt: `Here's everything we know: ${allData}. What's the verdict?`
});
```

**Right:**
```typescript
// Summarize and prioritize before synthesis
const relevantEvidence = prioritizeEvidence(allEvidence, claim);
const response = await claude.complete({
  prompt: `Based on these ${relevantEvidence.length} sources: ${formatEvidence(relevantEvidence)}`
});
```

## Perplexity Integration

### Structured Fact-Check Request

```typescript
const response = await perplexity.chat.completions.create({
  model: 'sonar-pro',
  messages: [
    {
      role: 'system',
      content: `You are a fact-checker. Return JSON only:
        {
          "verdict": "TRUE" | "FALSE" | "MISLEADING" | "UNVERIFIABLE",
          "confidence": 0.0-1.0,
          "summary": "One paragraph explanation",
          "sources": [{"url": "...", "title": "...", "excerpt": "..."}]
        }`
    },
    {
      role: 'user',
      content: `Fact-check this claim: "${claim}"`
    }
  ],
  response_format: { type: 'json_object' }
});
```

### Why Perplexity for Fact-Checking?

Perplexity Sonar Pro achieved **0.858 F-score on SimpleQA**, significantly outperforming:
- GPT-4: 0.38
- Claude 3 Opus: 0.47
- Gemini 1.5: 0.42

For factual accuracy specifically, Perplexity is the best available option.

## Error Handling

### Retry with Exponential Backoff

```typescript
async function callWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await sleep(Math.pow(2, i) * 1000); // 1s, 2s, 4s
    }
  }
  throw new Error('Unreachable');
}
```

### Graceful Degradation

```typescript
async function factCheck(claim: string): Promise<FactCheckResult> {
  // Try primary: Perplexity
  try {
    return await perplexityFactCheck(claim);
  } catch (error) {
    logger.warn('Perplexity failed, falling back to Brave + Claude');
  }
  
  // Fallback: Brave Search + Claude synthesis
  try {
    const searchResults = await braveSearch(claim);
    return await claudeSynthesize(claim, searchResults);
  } catch (error) {
    logger.error('All fact-check methods failed');
    return { verdict: 'UNCERTAIN', confidence: 0, degraded: true };
  }
}
```

## Monitoring

### Key Metrics to Track

```typescript
interface LLMCallMetrics {
  model: string;
  task: string;
  latencyMs: number;
  inputTokens: number;
  outputTokens: number;
  cost: number;
  success: boolean;
  retries: number;
}

// Log every LLM call
function logLLMCall(metrics: LLMCallMetrics) {
  logger.info('llm_call', metrics);
  
  // Alert on anomalies
  if (metrics.latencyMs > 30000) {
    alertSlack(`Slow LLM call: ${metrics.model} took ${metrics.latencyMs}ms`);
  }
}
```

### Cost Tracking

```typescript
const MODEL_COSTS = {
  'claude-sonnet-4': { input: 0.003, output: 0.015 }, // per 1K tokens
  'claude-3-5-haiku': { input: 0.00025, output: 0.00125 },
  'sonar-pro': { perRequest: 0.005 },
};

function calculateCost(model: string, inputTokens: number, outputTokens: number): number {
  const costs = MODEL_COSTS[model];
  if ('perRequest' in costs) return costs.perRequest;
  return (inputTokens / 1000) * costs.input + (outputTokens / 1000) * costs.output;
}
```
