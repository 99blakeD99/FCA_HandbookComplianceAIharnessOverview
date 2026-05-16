# First Harness Implementation

## Overview

Following widespread practice, this implementation has the following components:

- YAML describes *what* should happen, using a structure that when a suitable LLM is prompted to develop the code it can understand what to do
- Action Specifications describe *the inputs, outputs, and behavior* of each action, again using a structure that the LLM can understand
- Python makes it work. 

Anyone reading the YAML can look up the corresponding Action Specification to understand exactly what will occur. 

For now, `general_enquiry` is the only workflow covered, as a proof of concept; other types (regulatory_change_analysis, incident_investigation...) will be covered.

## Use of FCA_Handbook_Text_And_Embeddings

This implementation will be based on a FCA_Handbook_Text_And_Embeddings artifact following the structure of FCA_Handbook_Template_PRIN.json.

Initially voyage-3-large will be used as the embedding model.

The data will be ingested directly into Python. A database is not needed, for the following reasons:

- No real-time transactional updates — Handbook changes quarterly at most
- Read-only access — Harness never modifies, only queries
- Manageable scale — 10,438 records with 3072-dim vectors fit easily in memory
- Embeddings are static — Computed once, don't change
- Simple distribution — Just ship the JSON file

## YAML

### Why YAML for Compliance Workflows?

Compliance workflows are audit-intensive. Python code is executable but opaque to auditors. YAML brings key information to the surface. 

YAML is human-readable, version-controlled, and deterministic. This makes it more accessible to compliance teams. 

### Example: general_enquiry Workflow

For a concrete YAML workflow definition and complete action specifications for general_enquiry, see **GeneralEnquirySpecs.md**.

The crucial point is the **nodes** structure: each node has a `name`, `action` type, `input` declarations, `output` type, and `config` containing action-specific parameters.

Each node follows the same structure: `name`, `action`, `input` (if applicable), and `config` (containing `Param_*` parameters). Action-specific fields are normalized under `config`, making the structure consistent for the Python execution loop.

Future workflows will follow the same pattern. Examples: regulatory_change_analysis, incident_investigation, periodic_audit.

**Naming convention:** Fields prefixed with `Param_` are configurable parameters defined in action specifications. Changing these values affects behavior (if implemented in Python). Fields without the prefix are structural (node name, action type, input/output declarations).

## Workflow 

Workflow execution is as set out in the YAML and Action Specifications in GeneralEnquirySpecs.md.

## Audit Trail Example

When regulators ask "How did you decide rules A, B, C apply?", this implementation provides a complete trace:

1. **YAML node 1 (extract_features)**: "We extracted these features from your product description"
2. **YAML node 2 (retrieve_rules)**: "We queried FCA Handbook v2026Q1 from `FCA_Handbook_Text_And_Embeddings` for matching rules (Param_top_k=20)"
3. **YAML node 3 (analyze_compliance)**: "Claude reasoned that rules A, B, C apply: [reasoning log with timestamp]"
4. **YAML node 4 (human_review)**: "Compliance officer [name] reviewed and approved on [timestamp] with actions [approved, identity_verified, snapshot_recorded]"
   - Both `timestamp` and `actions_carried_out` are declared as `required_fields` in the YAML, ensuring they are always captured

Every decision is traceable to a specific YAML node, configuration version, and explicit audit metadata (timestamps, approver identity, actions taken).

## Usage Records

Every interaction with the Harness is recorded for auditability and process improvement.

### Record Structure

Each interaction is appended to `interactions.json`:

```json
{
  "timestamp": "ISO-8601",
  "product_description": "string",
  "question": "string",
  "rules_retrieved": [{"id": int, "header": "string", "rank": int, "similarity": float}],
  "claude_reasoning": "string",
  "approval_decision": "approved|rejected",
  "approver_comment": "string",
  "session_id": "uuid"
}
```

### Purposes

**a. Compliance**: Complete audit trail of decisions, rules applied, and approvals. Answers "How was this decision made?"

**b. Feedback**: Analyze interaction patterns (rejected approvals, follow-up questions, rule frequencies) to identify process gaps and embedding quality issues.

### Future Migration

As usage patterns emerge, may migrate to a structured database (Supabase, etc.) for richer querying. JSON format allows straightforward migration without code changes.

---

## Claude Functionality

### Departure from LLM Agnosticism, but not at core

This implementation departs from LLM agnosticism to leverage useful Claude features:

- **Prompt caching**: Stable context (regulatory hierarchy, citation discipline, standard prompts) is cached on first use.
- **Thinking tokens** (Later Claude LLMs needed): Enabled via cached prompts to reason through compliance decisions.

### LLM Used in First Implementation

The harness architecture is LLM-agnostic (see README.md § Design), but **this first implementation uses Claude** for the reasoning layer:

- **External invocation**: Can be Claude, any other LLM, or direct Python API call (architecture is LLM-agnostic)
- **Internal reasoning** (this implementation): Uses Claude (claude-opus-4-7) via the `claude_reasoning` action
- **Requirement**: `ANTHROPIC_API_KEY` environment variable must be set when harness runs

Other implementations could substitute a different LLM for the reasoning layer without changing the harness architecture. The deterministic retrieval and weighting layers are truly LLM-agnostic; only the reasoning layer is implementation-specific.

### Cached Harness Configuration for Repeated Questions

When the LLM calls the Tool, the full Harness configuration is cached on first use, as implemented in Python:

```python
cached_harness = [
    {
        "type": "text",
        "text": harness_config_yaml + weights_config_yaml + compliance_analyst_prompt,
        "cache_control": {"type": "ephemeral"}
    }
]
```

**Token efficiency for repeated compliance questions**:
- First compliance request: Claude reads Harness (~1200 tokens), cached
- Subsequent requests: Harness reused (~0 new tokens)
- **Savings: ~98% reduction on repeated questions**

For a firm handling multiple product reviews and regulatory changes, this efficiency is designed for your workload.
