# Structured Search: Regulatory Weighting for Compliance Queries

## Problem: Why Standard Embeddings Aren't Enough

Vector embeddings excel at semantic similarity, but they treat all matches equally. For regulatory compliance, this creates a critical problem:

**Example:**
- Query: "Which rules apply to robo-advisor discretionary management?"
- Embedding returns: 
  - "COBS 2.1.1R" (binding Rule) — similarity: 0.92
  - "COBS 2.1.1G" (Guidance) — similarity: 0.91
  - "Glossary definition of 'robo-advisor'" — similarity: 0.89

**Standard search ranks these by similarity alone.** But regulatory significance is obfuscated:
- The Rule is binding law (must comply)
- The Guidance is interpretive (should consider)
- The Glossary definition is just definitional (reference)

**Result:** Compliance analysis based on technically relevant but legally misleading results.

## Solution: Multi-Factor Regulatory Ranking

Structured search combines semantic similarity (from embeddings) with regulatory hierarchy weighting. Each result is scored by:

1. **Base Similarity** — Cosine similarity from embedding model (0–1)
2. **Rule Type Weight** — Regulatory binding authority
3. **Hierarchy Multiplier** — Document structure position
4. **Importance Multiplier** — Regulatory criticality
5. **Piece Weight** — Section/source authority

**Final Score = Base Similarity × All Weight Factors**

## Weighting Factors

### Rule Type (Regulatory Hierarchy)

The FCA Handbook encodes rule type in the rule_id suffix:

| Type | Code | Weight | Meaning |
|------|------|--------|---------|
| Rule | R | 1.0 | Explicitly defined as binding regulatory requirement |
| Guidance | G | 0.8 | Explicitly defined as interpretive guidance (should follow) |
| Evidential | E | 0.6 | Explicitly defined as example/case law (informational) |
| Unclassified | U | 0.8 | No explicit type found (e.g., annexes, definitions, forms) |
| Deleted | D | 0.1 | Explicitly marked as obsolete (historical only) |

**Extracting from data:** In `FCA_Handbook_Text_And_Embeddings`, the rule type appears as a single letter in the `regulatory_content` field, immediately following the rule ID number:
- Example: `"PRIN 1.1.1 G 01/01/2021 RP..."` → Extract `G` (Guidance, weight 0.8)
- Example: `"COBS 2.1.1R 31/07/2023 RP..."` → Extract `R` (Rule, weight 1.0)
- Example: `"PRIN 1 Annex 1 Non-designated..."` → No suffix found → Assign `U` (Unclassified, weight 0.8)

Implementation: Extract the first occurrence of [RGED] from `regulatory_content`. If not found, assign U (Unclassified). Then map to weights: `{R: 1.0, G: 0.8, E: 0.6, U: 0.8, D: 0.1}`

### Importance Multiplier (Regulatory Criticality)

Each rule has a regulatory_score (1–12) indicating criticality:

| Score Range | Level | Multiplier | Meaning |
|-------------|-------|------------|---------|
| 7–12 | HIGH | 1.3 | Critical areas (consumer protection, market integrity) |
| 4–6 | MEDIUM | 1.1 | Standard regulatory content |
| 1–3 | LOW | 0.7 | Peripheral or supporting content |

### Hierarchy Multiplier (Document Structure)

Position within the Handbook's nested structure:

| Position | Multiplier | Example |
|----------|-----------|---------|
| Primary rule (top-level) | 1.2 | COBS 2.1.1R |
| Sub-rule | 1.0 | COBS 2.1.1A (indented) |
| Guidance under rule | 0.9 | COBS 2.1.1G (indented) |

**Computing from data:** In `FCA_Handbook_Text_And_Embeddings`, each record includes a `level` field indicating document depth:

| Level | Interpretation | Multiplier |
|-------|-----------------|-----------|
| 1 | Top-level section (e.g., "COBS 2") | 1.2 |
| 2 | Subsection (e.g., "COBS 2.1") | 1.0 |
| 3+ | Nested items, guidance, sub-guidance | 0.9 |

Implementation: `hierarchy_multiplier = {1: 1.2, 2: 1.0}.get(level, 0.9)`

### Piece Weight (Source Authority)

Different sections of the Handbook have different authority:

| Piece | Weight | Authority |
|-------|--------|-----------|
| Main Handbook | 1.2 | Primary regulatory rules |
| Glossary | 0.95 | Definitions and terms |
| Instruments | 1.0 | Regulatory instruments |
| Forms | 0.85 | Templates and forms |
| Technical Standards | 0.9 | Technical guidance |
| Level 3 Materials | 0.85 | Supporting materials |

## Hardcoded Weights (MVP Implementation)

In the MVP implementation, weights are hardcoded in `HandbookIndex.__init__()` for auditability and determinism. This reference section documents the values and rationale.

**Current weights:**

```python
rule_type_weights:
  RULES:         1.0    # Binding regulatory requirements (explicitly marked)
  GUIDANCE:      0.8    # Interpretive regulatory guidance (explicitly marked)
  EVIDENTIAL:    0.6    # Evidential provisions (explicitly marked)
  UNCLASSIFIED:  0.8    # No explicit type (annexes, definitions, forms)
  DELETED:       0.1    # Deleted sections (minimal relevance)

importance_multipliers:
  high:    1.3       # Scores 7-12: Critical regulatory areas
  medium:  1.1       # Scores 4-6: Standard regulatory content
  low:     0.7       # Scores 1-3: Peripheral or supporting content

piece_base_weights:
  "Main Handbook":      1.2
  "Glossary":           0.95
  "Instruments":        1.0
  "Forms":              0.85
  "Technical Standards": 0.9
  "Level3Materials":    0.85
```

### Future: Tuning Weights for Different Scenarios

These examples show how weights could be tuned if migrated to external configuration:

**Conservative approach** (strict compliance focus):
- RULES: 1.0, GUIDANCE: 0.6

**Regulatory change scenario** (heightened scrutiny):
- importance_multipliers: high=1.5, medium=1.3

**Consumer protection focus**:
- importance_multipliers: high=1.5

## Integration with semantic_search Action

The `semantic_search` action (FirstImplementation.md) implements structured search:

```
INPUT: Product features + Compliance question
  ↓
1. Embed question using the same model used in FCA_Handbook_Text_And_Embeddings
2. Load FCA_Handbook_Text_And_Embeddings (in-memory JSON)
3. Compute cosine similarity against all rule embeddings
4. Select top 50 candidates (pre-filtering before weighting)
5. Apply hardcoded regulatory weights (defined in HandbookIndex.__init__)
6. For each candidate, compute:
   - rule_type_weight (from rule_id suffix)
   - hierarchy_multiplier (from metadata level)
   - importance_multiplier (from regulatory_score)
   - piece_weight (from piece/section)
   - final_score = similarity × all weights
7. Sort by final_score, return top_k (e.g., 20)
  ↓
OUTPUT: RankedRules (with full weight factor breakdown for auditability)
```

## Design Rationale

### Why Weights Are Hardcoded (MVP) and How to Migrate

**Current approach (MVP):** Weights are hardcoded in `HandbookIndex.__init__()` for simplicity and determinism.

**Auditability strategy:**
1. Weights are documented here in StructuredSearch.md — the authoritative reference
2. Git history shows exactly when and why weights changed (commit messages document regulatory rationale)
3. Each search result includes full weight factor breakdown for regulatory review
4. Code audit: compare HandbookIndex.__init__() against StructuredSearch.md to verify implementation matches spec

**Future improvement:** Move weights to external configuration (weights.yaml or environment variables) for:
- Non-technical tuning by compliance officers without code changes
- Dynamic weight switching per-query without restart
- Better separation of data (weights) from code

### Why This Matters for Compliance

Compliance decisions must be defensible:

> "Why did your system recommend Rule X over Rule Y?"

With structured search, you can answer:
- "Rule X is type R (binding), Rule Y is type G (advisory)"
- "Rule X scored 7/10 importance, Rule Y scored 3/10"
- "Rule X is in Main Handbook (weight 1.2), Rule Y is in Glossary (weight 0.95)"

**Every decision is traceable and auditable.**

## Notes

- Weights are version-controlled in git, enabling audit trails
- Weight changes take effect on next harness startup (no re-embedding needed)
- All weight changes should be documented in commit messages for compliance history
- For complex multi-rule scenarios, retrieved results include full weight factor breakdown
- The semantic_search action applies regulatory weighting automatically (enforced in code)
