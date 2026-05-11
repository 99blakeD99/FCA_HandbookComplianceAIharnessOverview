# First Harness Implementation

## Overview

Following widespread practice, this implementation has the following components:

- YAML declares *what* should happen
- Action Specifications define *the contract*
- Python makes it work. 

Anyone reading the YAML can look up the corresponding Action Specification to understand exactly what will occur. 

An auditor wishing to verify that this chain has integrity can simply ask a suitable LLM to verify.

For now, `new_product_review` is the only workflow covered, as a proof of concept; other types (regulatory_change_analysis, incident_investigation...) will be covered.

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

### Example YAML configuration: new_product_review Workflow

The following specifies information for the workflow: the crucial point to understand is the **nodes**.

```yaml
harness:
  data_sources:
    fca_handbook:
      artifact: "FCA_Handbook_Text_And_Embeddings"
      version: "2026-Q1"
      model: "text-embedding-3-large"
      weighting_config: "weights.yaml"
  
  workflows:
    new_product_review:
      nodes:
        - name: extract_features
          action: parse_markdown
          input: product_description
          output: ProductFeatures
          config: {}
        
        - name: retrieve_rules
          action: semantic_search
          input:
            product_features: ProductFeatures
            question: question
          output: RankedRules
          config:
            Param_top_k: 20
        
        - name: analyze_compliance
          action: claude_reasoning
          input:
            product_features: ProductFeatures
            rules: RankedRules
          output: ComplianceAnalysis
          config:
            Param_tools:
              - citation_formatter
              - audit_logger
            Param_prompt_template: "fca-compliance-analyst.md"
        
        - name: human_review
          action: approval_gate
          input:
            analysis: ComplianceAnalysis
          output:
            ApprovalDecision:
              required_fields:
                - timestamp
                - actions_carried_out
          config:
            Param_required_approval:
              Param_role: compliance_officer
              Param_record_identity: true
```

**Naming convention:** Fields prefixed with `Param_` are configurable parameters defined in the Action Specification. Changing these values will affect behavior (if implemented in Python). Fields without the prefix are structural (node name, action type, input/output declarations).

Each node follows the same structure: `name`, `action`, `input` (if applicable), and `config` (containing `Param_*` parameters). Action-specific fields are normalized under `config`, making the structure consistent for the Python execution loop.

Other workflows will in due course include: regulatory_change_analysis, incident_investigation, periodic_audit, etc.

## Workflow 

### Tool: Scope Validation and Harness Invocation

The Tool acts as the entry point and gatekeeper. It performs scope validation, then invokes the Harness where appropriate.

**Scope Validation**

Before executing the workflow, the Tool validates that the question concerns the **FCA Handbook specifically** (not PRA Requirements, technical standards, or other FCA documents).

**Validation logic:**
- If the question clearly targets FCA Handbook compliance: Harness executes immediately.
- If scope is ambiguous: Tool asks "Do you mean compliance with the **FCA Handbook**?" and only invokes the Harness if the answer is yes.
- If question is out of scope: Tool clarifies what it handles and does not invoke the Harness.

**Example:**
- User: "What are the FCA requirements for algorithmic advice?"
- LLM recognizes "FCA" and invokes the Tool
- Tool validates: scope is ambiguous, asks for clarification
- User: "Yes, FCA Handbook rules for algorithmic advice."
- Validation passes; Tool invokes Harness: retrieves rules, invokes LLM reasoning, returns auditable citations.

**Harness Invocation**

When the LLM invokes the Tool, it provides:
- `product_description` (string): Product details in markdown format
- `question` (string): The compliance question (e.g., "Which FCA rules apply?")

After scope validation passes, the Tool invokes the Harness, which carries out the following steps:

1. **Determines workflow type**: The LLM's question is classified to select the appropriate workflow. For example, "Attached is a summary of a new product we are creating, can you review it against the FCA Handbook?" triggers `new_product_review`. 

2. **Loads YAML workflow** and **Maps inputs**: The workflow definition is loaded from a YAML file. User inputs (`product_description`, `question`) are connected to workflow nodes according to the YAML configuration. In the example workflow, this means:
   - `product_description` → NODE 1 input
   - NODE 1 output (`ProductFeatures`) → NODE 2 input
   - `question` → NODE 2 input

3. **Initializes execution context** and **Executes workflow**: Prepares data sources, action registry, and audit logging, then runs nodes sequentially.

This keeps the Tool invocation simple (product + question) while allowing YAML to define the Harness orchestration. 

### Workflow Execution

The nodes set out in the YAML configuration are executed in sequence, with actions as set out in Action Specifications.md.

## Audit Trail Example

When regulators ask "How did you decide rules A, B, C apply?", this implementation provides a complete trace:

1. **YAML node 1 (extract_features)**: "We extracted these features from your product description"
2. **YAML node 2 (retrieve_rules)**: "We queried FCA Handbook v2026Q1 from `FCA_Handbook_Text_And_Embeddings` for matching rules (Param_top_k=20)"
3. **YAML node 3 (analyze_compliance)**: "Claude reasoned that rules A, B, C apply: [reasoning log with timestamp]"
4. **YAML node 4 (human_review)**: "Compliance officer [name] reviewed and approved on [timestamp] with actions [approved, identity_verified, snapshot_recorded]"
   - Both `timestamp` and `actions_carried_out` are declared as `required_fields` in the YAML, ensuring they are always captured

Every decision is traceable to a specific YAML node, configuration version, and explicit audit metadata (timestamps, approver identity, actions taken).

---

## Claude Functionality

### Departure from LLM Agnosticism, but not at core

This implementation departs from LLM agnosticism to leverage useful Claude features:

- **Prompt caching**: Stable context (regulatory hierarchy, citation discipline, standard prompts) is cached on first use.
- **Thinking tokens** (Later Claude LLMs needed): Enabled via cached prompts to reason through compliance decisions.

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
