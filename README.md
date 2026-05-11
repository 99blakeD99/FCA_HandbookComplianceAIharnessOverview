# Compliance Agent Harness

## Overview

This document outlines two complementary components:

1. **FCA_Handbook_AI_Enquiry_Harness** — The deterministic orchestration framework.

2. **FCA_Handbook_AI_Enquiry_Tool** — The non-deterministic AI-callable Tool that the Harness exposes. 

These components are put forward as a model for Compliance AI, enabling modification to suit other codified requirements.

## Document Organisation

This documentation is organized into six documents:

1. **README.md** (this document) — Conceptual overview of the Harness pattern, design principles, and how it addresses compliance requirements. Implementation-agnostic; suitable for regulators, compliance teams, and architects.

2. **EmbeddingModel.md** — Outline of considerations surrounding choice of embedding model.

3. **FirstImplementation.md** — One reference implementation. Uses YAML for workflow specification, Python for execution, and Anthropic Claude for reasoning. Shows YAML-to-Python bridge, ACTION_REGISTRY pattern, prompt caching, and workflow execution. For engineers and implementers. Other implementations could use alternative specification languages or LLM providers.

4. **StructuredSearch.md** — How to rank regulatory search results appropriately. Explains multi-factor weighting (rule type, hierarchy, importance, section authority) and weights.yaml configuration. Foundational to the semantic_search action. For architects, compliance teams, and implementers designing search strategies.

5. **ActionSpecifications.md** — Detailed specifications for each action type (parse_markdown, semantic_search, claude_reasoning, approval_gate). Provides the contract that implementations must conform to, independent of how the Harness is specified or which LLM is used. For engineers and auditors.

6. **PythonImplementation.md** — Detailed Python implementation patterns (action registry, node execution loop, data flow between nodes, error handling). For engineers building implementations.

7. **UIStrategy.md** — Application design strategy for the demonstration implementation. Covers minimal feature set, user flows, approval gate UX, technology stack (Streamlit), and deployment considerations. For product, UX, and engineering teams building the demo application.

## FCA_Handbook_Text_And_Embeddings

This Harness and Tool depend on **`FCA_Handbook_Text_And_Embeddings`**, a composite artifact combining FCA regulatory text with vector embeddings.

## Harness as Complement to LLM

The LLM provides reasoning (intelligence); the Harness provides reliability.

**What the Harness Does** 

| Task | Comment |
|------|-----|
| Retrieve FCA rules via semantic search using embeddings | Deterministic; auditable |
| Apply regulatory weighting based on Handbook hierarchy | Enforces Handbook structure, not reasoning |
| Return top N weighted results | Deterministic; N configurable per design |
| Format citations as rule + reference | Standardized; prevents paraphrasing |
| Validate retrieved citations against source | Confirm accuracy |
| Log the reasoning chain | Audit trail |

Note: There are AI-like components in these functions. *Machine Learning* models are used to compute *embeddings*, which are used by applying *cosine similarity* (using the well-proven scipy library). These are deterministic, and will produce reproducible results: the same inputs will produce the same outputs. It is *not* "generative AI".

**What the LLM Does** 

| Task | Why |
|------|-----|
| Classify the question to determine workflow type | Understands intent; routes to new_product_review, regulatory_change_analysis, etc. |
| Reason which of the N rules apply to a specific situation | Genuine reasoning—requires interpretation |
| Explain *why* a rule applies | Interpretation and nuance; LLM's strength |
| Identify gaps or ambiguities | Flagging edge cases; LLM's strength |
| Quote the retrieved citations and provide context | LLM can point to specific rule text |

Note 1: These functions are "generative AI", and non-deterministic. The LLM will use whatever it thinks relevant at the time, subject to the constraints of the Harness.

Note 2: The LLM can be forced to be deterministic by setting "argmax" and temperature=0. But this removes the LLM's ability to reason. In this implementation the setting is "sampling" and temperature=0.5.

## Design

The guiding design principles are:

1. **Auditability** — Every decision must be traceable: question → retrieval → reasoning → citation → approval
2. **Deterministic Workflows** — Python executes the harness exactly as specified
3. **Graceful Failure** — No silent defaults to web search. Explicit errors when data unavailable.
4. **LLM Agnosticism at Core** — The core architecture is LLM-agnostic.
5. **Embedding Model Agnosticism** - New embedding models can easily be accommodated.

## Harness Invocation

The compliance Harness appears as a **named tool** with a focused description that enables the LLM to route compliance questions appropriately.

**Routing and invocation:**

The LLM reads the Tool description and makes a routing decision. If the question involves FCA, the LLM invokes the Tool. The Tool then validates scope and executes the Harness workflow. Implementation overview is set out in FirstImplementation.md.

