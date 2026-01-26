# ADR 001: Evidence-Only Synthesis

## Status

Accepted

## Context

LLMs have training data that may contain outdated or incorrect information. When asked to verify a claim, an LLM might use its training knowledge instead of the evidence we've gathered, leading to:

1. **Hallucinated verdicts** - LLM "remembers" something that contradicts current evidence
2. **Stale information** - Training data cutoff means recent events are unknown
3. **Confidence without basis** - LLM sounds certain even when evidence is weak

For a fact-checking system, this is unacceptable. Users trust our verdicts because they're evidence-based.

## Decision

We explicitly constrain the synthesis LLM to use only retrieved evidence, never training knowledge.

### Implementation

The system prompt includes a hard constraint:

```
CRITICAL CONSTRAINT: Evidence-Only Synthesis

You must determine the verdict based ONLY on the evidence provided below.
Do NOT use your training knowledge to override or supplement this evidence.

If the evidence says a claim is false, your verdict must reflect that,
even if your training data suggests otherwise.

If evidence is insufficient, return UNCERTAIN or UNVERIFIABLE.
```

Additionally, the LLM must complete a validation checklist before finalizing:

```
Before finalizing your verdict, verify:

1. DIRECTION CHECK: Does your verdict direction (true/false) match the majority of evidence?
2. CONFIDENCE CHECK: Is your confidence based on actual evidence quality, not a default value?
3. EVIDENCE CHECK: Is your verdict based only on retrieved evidence, not training data?
4. COHERENCE CHECK: Does your summary support your verdict, not contradict it?
5. SATIRE CHECK: Did you check if the source is a known satire site?
```

## Consequences

### Positive

- **Auditable verdicts** - Every claim can be traced to source citations
- **No hallucinations** - LLM cannot invent facts not in evidence
- **Transparent uncertainty** - Returns UNCERTAIN when evidence is insufficient rather than guessing
- **Updatable** - New evidence immediately affects verdicts without retraining

### Negative

- **Slower** - Must always gather evidence even for "obvious" claims
- **More expensive** - Requires Perplexity/search calls for every verification
- **Occasional frustration** - Returns UNCERTAIN for claims the LLM "knows" but cannot verify

### Tradeoffs Accepted

We accept higher cost and latency in exchange for accuracy and auditability. For a fact-checking system, a wrong confident answer is worse than an uncertain correct one.

## Alternatives Considered

### 1. Hybrid approach (evidence + training data)

Rejected. Too easy for training data to override weak evidence. No clear boundary for when to trust which source.

### 2. Fine-tuned fact-checking model

Rejected. Training data becomes stale. Cannot verify claims about events after training cutoff. Expensive to maintain.

### 3. RAG with vector database of trusted sources

Partially adopted. We use real-time search (Perplexity) rather than a static vector DB because facts change. Could add vector DB for historical context in future.
