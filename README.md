# FCA Handbook Compliance Agent Harness

## Overview

This repo sets out specifications for the **FCA Handbook Compliance Agent Harness** — a deterministic orchestration framework capable of containing multiple Workflows. The Harness is invoked as a named tool by external LLMs. Upon invocation, an internal LLM is used: at the beginning to classify the input information and choose the appropriate Workflow; and again at the end for reasoning analysis.

The Harness is necessary in order to ensure that the internal LLM actually uses the FCA Handbook rather than the data used in its training (which is very likely to be based on secondary, out-of-date, or unreliable sources).

This architecture is put forward as a template for Compliance AI, enabling modification to suit other codified requirements. The starting point is a JSON file containing the codified requirements (to which as a prerequisite first step appropriate embeddings need to be added). Essentially this repo sets out a map of what to do with it.

These specifications have not been live-tested in a production environment. Implementation should be validated against real-world compliance workflows and regulatory scrutiny before deployment in regulated use.

The specifications are written using conventions which are human readable, but structured such as to enable giving to a suitable LLM with the instruction to create the requisite code. Similarly an auditor wishing to verify that the chain from specs to python has integrity can simply ask a suitable LLM. 

## Tool Description

This is the Tool Description that appears to external LLMs to enable them to route compliance questions appropriately:

> **FCA Handbook Compliance Enquiry**
>
> Use this tool to answer questions about FCA regulatory compliance. The tool:
> - Retrieves relevant FCA Handbook entries via semantic search with regulatory weighting
> - Reasons over retrieved entries to determine applicability to specific situations
> - Provides citations with exact entry text for audit and verification
> - Returns a formal compliance analysis with identified gaps and confidence scores
>
> Invoke when users ask: "Which FCA Handbook requirements apply to [product/service]?", "Is [situation] compliant?", or similar regulatory questions.
>
> Do NOT invoke for non-FCA questions, general financial advice, or questions outside UK financial services regulation.

**Deployment of this Tool Description:**

- **Direct integration:** Include the description in the LLM SDK tool definition (Anthropic, OpenAI, etc.) so the external LLM can invoke the tool
- **Service deployment:** Store the Tool Description in your service's configuration files when exposing the Harness as an API
- **Tool marketplace:** Register in the platform's tool definition interface (e.g., Anthropic tool marketplace, LLM-specific registries)

## Getting Started

To understand the Compliance Agent Harness architecture, read the documents in order:

1. **[README.md](README.md)** (this document). Conceptual overview of the Harness.

2. **[EmbeddingModel.md](EmbeddingModel.md)**. Outline of considerations surrounding choice of embedding model.

3. **[StructuredSearch.md](StructuredSearch.md)**. How to rank search results, having regard to the highly structured organisation of codified requirements. Foundational to the semantic_search action. 

4. **[FirstImplementation.md](FirstImplementation.md)**. One reference implementation. 

5. **[GeneralEnquirySpecs.md](GeneralEnquirySpecs.md)**. Workflow definition (YAML) and detailed specifications for each action type. Future workflows will have similar spec files.

6. **[PythonImplementation.md](PythonImplementation.md)**. Detailed Python implementation patterns.

7. **[UIStrategy.md](UIStrategy.md)**. Design outline for demo. The Harness will normally be called as part of a bigger system, but a standalone demo will be useful. Also see [UI_mockup.html](https://htmlpreview.github.io/?https://github.com/99blakeD99/FCA_HandbookComplianceAIharnessOverview/blob/main/UI_mockup.html) (open in browser).

8. **[FCA_Handbook_Template_PRIN.json](FCA_Handbook_Template_PRIN.json)**. A concrete unofficial subset of data with two embeddings (Voyage 3-Large and OpenAI text-embedding-3-large models).

9. **[Status.md](Status.md)**. Outstanding issues which need design decisions before coding. Some will be resolved as feedback emerges; others are decisions for those forking the repo for their own implementation.

## FCA_Handbook_Text_And_Embeddings

The Harness will depend on **`FCA_Handbook_Text_And_Embeddings`**, a composite artifact combining FCA regulatory text with vector embeddings (of which FCA_Handbook_Template_PRIN.json is an illustrative subset)

## Harness as Complement to LLM

The Harness provides reliability, the LLM provides reasoning (intelligence).

**What the Harness Does** 

| Task | Comment |
|------|-----|
| Retrieve FCA Handbook entries via semantic search using embeddings | Deterministic; auditable |
| Apply regulatory weighting based on Handbook hierarchy | Enforces Handbook structure, not reasoning |
| Return top N weighted results | Deterministic; N configurable per design |
| Format citations as entry + reference | Standardized; prevents paraphrasing |
| Validate retrieved citations against source | Confirm accuracy |
| Log the reasoning chain | Audit trail |

Note 1: There are AI-like components in these functions. *Machine Learning* models are used to compute *embeddings*, which are used by applying *cosine similarity* (using the well-proven numpy library). These are deterministic, and will produce reproducible results: the same inputs will produce the same outputs. It is *not* "generative AI".

Note 2. Generally the functionality in the Harness uses well-proven non-AI methods (regex, numpy for linear algebra, and embeddings HTTP client).

**What the LLM Does** 

| Task | Why |
|------|-----|
| Classify the question to determine workflow type | Understands intent; routes to general_enquiry, regulatory_change_analysis, etc. |
| Reason which of the N entries apply to a specific situation | Genuine reasoning—requires interpretation |
| Explain *why* an entry applies | Interpretation and nuance; LLM's strength |
| Identify gaps or ambiguities | Flagging edge cases; LLM's strength |
| Quote the retrieved citations and provide context | LLM can point to specific entry text |

Note 1: These functions are "generative AI", and non-deterministic. The LLM will use whatever it thinks relevant at the time, subject to the constraints of the Harness.

Note 2: The LLM can be forced to be deterministic by setting "argmax" and temperature=0. But this removes the LLM's ability to reason. In this implementation the setting is "sampling" and temperature=0.5.

## Design

The guiding design principles are:

1. **Auditability** — Every decision must be traceable: question → retrieval → reasoning → citation → approval
2. **Deterministic Workflows** — Python executes the harness exactly as specified
3. **Graceful Failure** — No silent defaults to web search. Explicit errors when data unavailable.
4. **LLM Agnosticism at Core** — The core architecture is LLM-agnostic.
5. **Embedding Model Agnosticism** - New embedding models can easily be accommodated.

## About this GitHub Repo

This repo contains the reference implementation of the Compliance Agent Harness. 

If you have questions, talking points, ideas, experiences, feedback: please open a GitHub Issue. 

We encourage you to **fork and adapt this implementation** for your specific regulatory requirements.

## License

This project is licensed under the MIT License. See [opensource.org/licenses/MIT](https://opensource.org/licenses/MIT) for details.

## Architecture & Design

Blake Dempster, [JBMD.co.uk](https://jbmd.co.uk)
