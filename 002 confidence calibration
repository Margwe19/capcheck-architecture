# ADR 002: Confidence Calibration

## Status

Accepted

## Context

During testing, we observed a problematic pattern: LLMs frequently output **exactly 60% confidence** when uncertain. This appears to be a learned behavior - 60% is high enough to seem useful but hedged enough to avoid being wrong.

The problem: a verdict with 60% confidence based on 5 high-quality sources should be treated very differently than 60% confidence based on speculation.

Example failure mode:
- Claim: "NASA announced discovery of alien life"
- Evidence: 5 Tier-1 sources (.gov, AP, Reuters) all say FALSE
- LLM output: "FALSE with 60% confidence"

The 60% is meaningless. The actual confidence should be 90%+ given source quality.

## Decision

Implement a confidence calibration system that:

1. **Detects default values** - Flags any confidence within 1% of 60%
2. **Recalculates from evidence** - Uses source count, quality, and consensus
3. **Applies adjustments** - Boosts for strong evidence, penalizes for conflicts
4. **Never returns 60%** - Final safety check ensures we don't output the default

### Algorithm

```typescript
function calibrateConfidence(rawConfidence: number, signals: ConfidenceSignals): number {
  let adjusted = rawConfidence;
  
  // CRITICAL: Detect and replace default 60%
  if (Math.abs(rawConfidence - 0.6) < 0.01) {
    adjusted = calculateFromEvidence(signals);
  }
  
  // BOOST: Multiple high-credibility sources
  if (signals.sourceCount >= 3 && signals.avgSourceTier <= 2.0) {
    adjusted = Math.min(0.95, adjusted + 0.15);
  }
  
  // PENALTY: Conflicting evidence
  if (signals.hasConflictingEvidence) {
    adjusted = Math.max(0.35, adjusted - 0.1);
  }
  
  // Final safety: never return exactly 60%
  if (Math.abs(adjusted - 0.6) < 0.01) {
    adjusted = 0.62;
  }
  
  return adjusted;
}

function calculateFromEvidence(signals: ConfidenceSignals): number {
  let base = 0.5;
  
  // Source quantity (max +0.2)
  base += Math.min(signals.sourceCount / 5, 1) * 0.2;
  
  // Source quality (max +0.2)
  base += ((5 - signals.avgSourceTier) / 4) * 0.2;
  
  // Fact-checker agreement (max +0.15)
  base += signals.factCheckerAgreement * 0.15;
  
  return Math.max(0.25, Math.min(0.9, base));
}
```

### Source Credibility Tiers

| Tier | Examples | Weight |
|------|----------|--------|
| 1 | .gov, .edu, peer-reviewed journals, AP, Reuters | Highest |
| 2 | Major newspapers, established news orgs | High |
| 3 | Regional news, trade publications | Medium |
| 4 | Blogs with editorial standards | Low |
| 5 | Social media, forums, unknown sources | Minimal |

## Consequences

### Positive

- **Meaningful confidence scores** - 85% actually means strong evidence
- **Calibrated to evidence quality** - High-tier sources boost confidence appropriately
- **Transparent factors** - Every adjustment is logged with reason
- **No more "60% everything"** - Eliminates the default value problem

### Negative

- **Added complexity** - Another system to maintain and tune
- **Potential for miscalibration** - Our weights might not be optimal
- **Disagreement with LLM** - Sometimes we override the model's judgment

### Monitoring

We log all calibration adjustments:

```json
{
  "rawConfidence": 0.60,
  "adjustedConfidence": 0.82,
  "factors": [
    {"factor": "default_override", "impact": +0.22, "reason": "Replaced default 60%"},
    {"factor": "high_credibility_sources", "impact": +0.15, "reason": "4 sources, avg tier 1.5"}
  ]
}
```

This allows us to audit calibration decisions and tune weights over time.

## Alternatives Considered

### 1. Ask LLM to self-calibrate

Rejected. The LLM is the source of the problem. Asking it to fix its own confidence doesn't work.

### 2. Train a separate calibration model

Considered for future. Would require labeled dataset of "true" confidence values. Current heuristic approach is sufficient for MVP.

### 3. Remove confidence scores entirely

Rejected. Users need to know how certain we are. Binary TRUE/FALSE is too limiting.

## References

- [Calibration of Language Models](https://arxiv.org/abs/2207.05221) - Academic research on LLM confidence
- Internal testing data showing 60% clustering
