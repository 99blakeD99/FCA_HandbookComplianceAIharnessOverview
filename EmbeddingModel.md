# Embedding Model 

## Action

The first step in preparing for this Harness is to ensure that the JSON file containing the codified requirements (in this case the FCA Handbook) is modified to include embeddings (as illustrated in FCA_Handbook_Template_PRIN.json). This can readily be done by instructing a suitable LLM.

## Background: Data Characteristics of FCA Handbook

The FCA Handbook presents unique embedding challenges due to its structure and diversity:

### Content Distribution
- Main Handbook: 3,670 records, avg 5,251 chars (complex regulatory text)
- Glossary: 3,640 records, avg 323 chars (definitions and terms)
- Instruments: 1,933 records, avg 81 chars (short regulatory instruments)
- Forms: 812 records, avg 66 chars (brief form descriptions)
- Technical Standards: 197 records, avg 134 chars (technical guidance)
- Level 3 Materials: 186 records, avg 104 chars (supporting materials)

### Key Insights
1. **Extreme Length Variance:** 6 chars to 337,850 chars (Main Handbook entries) — requires context window sufficient for complete rules
2. **Highly Structured:** 85 modules with consistent regulatory patterns — benefits from models trained on legal/regulatory text
3. **Hierarchical:** 5 levels with clear regulatory importance scores (1-12) — semantic search must preserve this hierarchy (handled via weighting in StructuredSearch.md)
4. **Domain-Specific:** Financial services regulatory language — requires embedding model optimized for legal/regulatory domain

These characteristics inform the embedding model choice: a model with longer context windows, legal domain optimization, and proven regulatory text performance is essential.

---

## Current Choice

**Model**: Voyage AI `voyage-3-large`

### Why Voyage 3-Large

1. **Legal Domain Optimization**: MLEB-tested (score: 85.71, #2 ranking globally) specifically for legal/regulatory document retrieval. This matters because FCA rules are regulatory text with domain-specific terminology that general-purpose models may miss.

2. **Proven in Production**: Released and battle-tested. Second-highest performer on MLEB (first is the newer Kanon 2). Low migration risk.

3. **API Compatibility**: Drop-in replacement for OpenAI's API. Minimal code changes required.

4. **Longer Context**: 32,000 tokens vs OpenAI's 8,191. Allows storing complete rules without truncation (~90 pages vs ~23 pages).

5. **Free Tier Available**: 200M tokens free per account for all Voyage models, reducing embedding costs for testing/development.

6. **Cost-Effective**: $0.18/1M tokens vs $0.13 for OpenAI = minimal cost increase (~$0.05 more for full handbook) for substantially better legal domain performance.

### Previous Model

**OpenAI `text-embedding-3-large`** was chosen historically without domain-specific evaluation. It performs well for general-purpose retrieval but is not optimized for legal/regulatory text. Recent research (2025-2026) revealed legal domain-specific models significantly outperform general-purpose models on compliance document retrieval.

## Research Findings

### Discovery: Legal Domain Optimization Matters

A new benchmark emerged in 2025: **MLEB (Massive Legal Embedding Benchmark)** that tests embedding models specifically on legal document retrieval. Key finding:

> The qualities that make an embedding model perform well at general multilingual information retrieval tasks are not necessarily the same as those that make a model perform well at legal information retrieval.

This matters for FCA Handbook because:
- FCA rules are regulatory/legal text with domain-specific terminology
- General-purpose models may miss nuances critical for compliance
- Legal-optimized models score higher on legal retrieval benchmarks

### Top Models for Regulatory Text (2026)

**General-Purpose (High Performers)**:
- OpenAI text-embedding-3-large
- Qwen3-Embedding-8B (open-source alternative)
- BGE-M3 (pre-trained on legal/finance data)
- Voyage AI family (optimized for long documents)

**Legal-Domain-Optimized**:
- Kanon 2 Embedder (MLEB rank: #1)
- Voyage 3.5 (MLEB rank: #3)
- LegalBERT (older, but domain-specific)

## Comparison: Cost & Latency

| Model | Cost/1M tokens | Latency | Context Length | Legal MLEB Score | Best For |
|-------|----------------|---------|-----------------|-----------------|----------|
| **OpenAI text-embedding-3-large** | $0.13 | ~200ms | 8,191 tokens | N/A | Current; widely integrated |
| **Voyage 4-lite** | $0.02 | ~100ms | 32,000 tokens | N/A (general) | Cost-conscious; general use |
| **Voyage 3-large** | $0.18 | ~150ms | 32,000 tokens | 85.71 | Legal-optimized; proven; balanced |
| **Kanon 2 Embedder** | Custom* | ~60ms | 16,384 tokens | 86.03 🏆 | Fastest; best legal score; self-hosted |

*Kanon 2 pricing requires contacting Isaacus; discounts negotiable.

## Key Metrics Explained

### MLEB Score (Massive Legal Embedding Benchmark)

Score range: 0-100 (NDCG@10 metric). Measures retrieval quality on legal documents.

- Kanon 2 Embedder: 86.03 (1st place)
- Voyage 3-large: 85.71 (2nd place)
- OpenAI text-embedding-3-large: Not benchmarked on MLEB

### Latency

Embedding time per request:

- Kanon 2 Embedder: ~60ms (340% faster than Voyage 3-large)
- Voyage 4-lite: ~100ms (fastest general-purpose)
- OpenAI text-embedding-3-large: ~200ms

### Context Length

Maximum tokens per document before truncation:

- Kanon 2 Embedder: 16,384 tokens (~46 pages legal text)
- Voyage models: 32,000 tokens (~90 pages)
- OpenAI text-embedding-3-large: 8,191 tokens (~23 pages)

Matters for FCA rules that span multiple sections.

## Comparative Context: Other Models

For reference, other models considered:

| Model | Legal MLEB Score | Context | Cost | Notes |
|-------|------------------|---------|------|-------|
| **Voyage 3-large** | **85.71** 🏆 | **32K tokens** | **$0.18/1M** | **Chosen** |
| OpenAI text-embedding-3-large | N/A (not benchmarked) | 8K tokens | $0.13/1M | Previous choice; not legal-optimized |
| Kanon 2 Embedder | 86.03 (highest) | 16K tokens | Custom | Fastest; best scores; newer; requires API integration |
| Voyage 4-lite | N/A | 32K tokens | $0.02/1M | Cheapest; for general use, not legal-optimized |

## Sources

- [Voyage AI Pricing](https://docs.voyageai.com/docs/pricing)
- [Embedding Models Pricing April 2026](https://awesomeagents.ai/pricing/embedding-models-pricing/)
- [Massive Legal Embedding Benchmark (MLEB)](https://huggingface.co/blog/isaacus/introducing-mleb)
- [Kanon 2 Embedder: Australian LLM beats OpenAI at legal retrieval](https://huggingface.co/blog/isaacus/kanon-2-embedder)
- [Kanon 2 Embedder AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-qoz2rxyqhtewu)
- [Best Embedding Models 2026 - MTEB Benchmarks](https://blog.premai.io/best-embedding-models-for-rag-2026-ranked-by-mteb-score-cost-and-self-hosting/)
- [Text Embedding Models Compared 2026](https://pecollective.com/tools/text-embedding-models-compared/)

## Rationale Summary

The shift from OpenAI to Voyage 3-Large reflects a strategic decision to optimize for the specific domain—legal/regulatory text retrieval—rather than general-purpose embeddings. The MLEB benchmark (introduced 2025) definitively shows that legal-domain optimization matters: Voyage 3-Large scores 85.71, while OpenAI's general-purpose model is not benchmarked on legal retrieval at all.

For FCA compliance analysis, the ability to accurately retrieve nuanced regulatory rules is more important than minimal cost savings. Voyage 3-Large provides this with:
- Proven legal domain performance (MLEB #2 globally)
- API compatibility (minimal code disruption)
- Longer context windows (complete rule storage)
- Production-ready stability

## Notes

- All pricing current as of May 2026
- MLEB benchmarks reflect testing as of October 2025
- Model performance on FCA Handbook specifically should be tested before full migration
- Free tier considerations:
  - Voyage 4 models: 200M free tokens/account
  - OpenAI: No free embedding tier
  - Kanon 2: Contact Isaacus for trial
