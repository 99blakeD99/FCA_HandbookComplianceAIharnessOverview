# General Enquiry Workflow Specification

This document specifies the **general_enquiry** workflow, which is the first implementation of the Compliance Agent Harness. It includes the YAML workflow definition and detailed specifications for each action type used in this workflow.

The general_enquiry workflow handles: product/service description → feature extraction → rule retrieval → compliance reasoning → human approval.

Other workflows (regulatory_change_analysis, incident_investigation, etc.) will have similar specifications files, following this template.

## Workflow Definition

```yaml
harness:
  data_sources:
    fca_handbook:
      artifact: "FCA_Handbook_Text_And_Embeddings"
      version: "2026-Q1"
      model: "text-embedding-3-large"
      weighting_config: "weights.yaml (schema in StructuredSearch.md)"
  
  workflows:
    general_enquiry:
      nodes:
        - name: extract_features
          action: parse_markdown
          input: entity_description
          output: EntityFeatures
          config: {}
        
        - name: retrieve_rules
          action: semantic_search
          input:
            entity_features: extract_features
            question: question
          output: RankedRules
          config:
            Param_top_k: 20
            Param_source_version: "2026-Q1"
        
        - name: analyze_compliance
          action: claude_reasoning
          input:
            entity_features: extract_features
            rules: retrieve_rules
          output: ComplianceAnalysis
          config:
            Param_tools:
              - citation_formatter
              - audit_logger
            Param_prompt_template: "fca-compliance-analyst.md"
        
        - name: human_review
          action: approval_gate
          input:
            analysis: analyze_compliance
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

**Naming convention:** Fields prefixed with `Param_` are configurable parameters defined in the Action Specification below. Changing these values will affect behavior (if implemented in Python). Fields without the prefix are structural (node name, action type, input/output declarations).

Each node follows the same structure: `name`, `action`, `input` (if applicable), and `config` (containing `Param_*` parameters). Action-specific fields are normalized under `config`, making the structure consistent for the Python execution loop.

---

## Action Specifications

Each action type is implemented by a Python class that conforms to an exact specification. This ensures auditability: any auditor or implementer can read the spec and verify that the Python code matches it.

### parse_markdown

**Purpose**: Extract and structure entity information from markdown-formatted entity description.

**Input**: 
- `entity_description` (string): Markdown-formatted text describing an entity (product, service, business line, etc.)

**Output**: 
- `EntityFeatures` (object) with fields:
  - `entity_name` (string): Name of the entity
  - `entity_type` (string): Category or classification (e.g., "advisory platform", "robo-advisor", "trading system")
  - `features` (array of strings): Key capabilities and features
  - `use_cases` (array of strings): Intended use and client segments
  - `client_interaction` (string): How clients interact (manual, automated, hybrid)
  - `data_handled` (array of strings): Types of sensitive data (e.g., "client portfolios", "trading instructions")
  - `decision_authority` (string): Who makes final decisions (algorithm, advisor, client)

**Process**:
1. Parse markdown structure (headings, sections)
2. Extract entity name from first heading (h1)
3. Search for standard sections: Features, Use Cases, Architecture, Data Handling, Client Interaction
4. Normalize extracted text (trim whitespace, standardize formatting)
5. Validate required fields present (entity_name, features, use_cases, decision_authority)
6. Return structured object

**Validation**:
- `entity_name`: Non-empty string, max 200 chars
- `features`: Non-empty array, min 1 item, max 50 items, each item max 500 chars
- `use_cases`: Non-empty array, min 1 item, max 20 items
- `decision_authority`: One of: "algorithm", "advisor", "client", "hybrid"
- If any required field missing: raise error with field name

**Error Handling**:
- Invalid markdown: Return error "Unable to parse product description structure"
- Missing required fields: Return error "Missing required field: {field_name}"
- Field validation fails: Return error "Invalid value for {field_name}: {reason}"

### semantic_search

**Purpose**: Query `FCA_Handbook_Text_And_Embeddings` for regulatory rules matching entity features and user question. Apply regulatory weighting to rank results by binding authority and relevance.

**Input**:
- `entity_features` (EntityFeatures object): Structured output from parse_markdown
- `question` (string): User's compliance question (e.g., "Which COBS rules apply?")

**Configuration** (from YAML node config):
- `Param_top_k` (integer): Number of results to return (default 20, range 1–100)

**Implementation Constants** (hardcoded, not parameterized):
- Data source: `FCA_Handbook_Text_And_Embeddings` (from harness configuration)
- Weighting: Always enabled (regulatory weighting always applied)

**Output**:
- `RankedRules` (array of objects) with fields:
  - `rule_id` (string): FCA rule identifier (e.g., "COBS 2.1.1R")
  - `text` (string): Verbatim rule text from source
  - `base_similarity` (number, 0–1): Cosine similarity from embeddings
  - `weight_factors` (object):
    - `rule_type_weight` (number): Binding authority multiplier
    - `hierarchy_multiplier` (number): Position in handbook structure
    - `importance_multiplier` (number): Regulatory criticality
    - `piece_weight` (number): Section weighting
  - `final_score` (number): base_similarity × rule_type_weight × hierarchy_multiplier × importance_multiplier × piece_weight
  - `source_version` (string): Version of FCA_Handbook_Text_And_Embeddings used (e.g., "2026-Q1")

**Process**:
1. Concatenate entity_features (name, type, features, use_cases) with user question as plain text
2. Embed the combined text using the same embedding model (text-embedding-3-large)
3. Compute cosine similarity between query embedding and rule embeddings from the pre-loaded FCA_Handbook_Text_And_Embeddings (loaded once at harness initialization). See StructuredSearch.md for how similarity scores are then weighted by rule type, hierarchy, importance, and piece authority.
5. Sort by cosine similarity, select top 50 candidates (filtering step before weighting)
6. Apply regulatory weighting algorithm to rank candidates by final_score (see StructuredSearch.md for factor definitions and weights.yaml (schema in StructuredSearch.md) configuration)
7. Sort by final_score descending
8. Return top_k results with full weight_factors breakdown for auditability

**Validation**:
- `entity_features`: Must contain entity_type and features (validate EntityFeatures structure)
- `question`: Non-empty string, min 5 chars, max 1000 chars
- `Param_top_k`: Integer in range [1, 100], default 20
- Harness configuration: weights.yaml (schema in StructuredSearch.md) must be present and valid; if missing or invalid: raise error with config path

**Error Handling**:
- Data source unavailable: Return error "FCA Handbook data (FCA_Handbook_Text_And_Embeddings) is temporarily unavailable. Escalate to compliance_team@company.com"
- Embedding API failure: Return error "Embedding service unavailable. Unable to query FCA Handbook."
- Invalid weights.yaml (schema in StructuredSearch.md): Return error "Regulatory weights configuration invalid at {path}: {reason}"
- No results found: Return empty array with informational log (not an error—some queries legitimately have no matches)

### claude_reasoning

**Purpose**: Invoke Claude to reason over retrieved rules and product features, producing a compliance analysis with citations and reasoning logs.

**Input**:
- `entity_features` (EntityFeatures object): Structured product information
- `rules` (RankedRules array): Top-ranked FCA rules from semantic_search

**Configuration** (from YAML node config):
- `Param_tools` (array of strings): Internal tool names available to Claude (e.g., ["citation_formatter", "audit_logger"])
- `Param_prompt_template` (string): Path to prompt template file (e.g., "fca-compliance-analyst.md")

**Output**:
- `ComplianceAnalysis` (object) with fields:
  - `answer` (string): Claude's reasoning and conclusions
  - `citations` (array of objects):
    - `rule_id` (string): FCA rule cited (e.g., "COBS 2.1.1R")
    - `cited_text` (string): Exact verbatim text from FCA Handbook
    - `context` (string): Sentence or paragraph from rule showing why it applies
    - `binding_level` (string): "R", "G", "E", or "D" (from rule_id suffix)
  - `reasoning_log` (object):
    - `reasoning_chain` (string): Claude's step-by-step logic ("rule X applies because feature Y matches criterion Z")
    - `gaps_identified` (array of strings): Edge cases or areas of uncertainty
    - `confidence_score` (number, 0–1): Model's confidence in the analysis
  - `timestamp` (string): ISO 8601 timestamp of analysis

**Process**:
1. Load Jinja2 prompt template from file (e.g., `harness/prompts/fca-compliance-analyst.md`)
2. Construct system prompt by rendering template with entity_features and rules. Rules include rule_id (authoritative identifier) for each RankedRule. Template syntax: `{{ entity_features }}`, `{{ ranked_rules }}`
3. Enable prompt caching for stable context (regulatory hierarchy, citation discipline, standard instructions)
4. Call Claude API with:
   - System prompt (cached) — includes RankedRules with rule_id fields
   - User message: question + available internal tools description
   - Internal tools: citation_formatter, audit_logger (to enforce structured output and logging; citation_formatter must extract rule_id)
   - Model: claude model with thinking tokens
   - Temperature: 0.5 (deterministic but thoughtful)
   - Max tokens: 2000
   - Extended thinking: enabled (for complex multi-rule scenarios)
5. Parse Claude's response to extract reasoning_log, citations (by rule_id), and answer text
6. Validate each citation by looking up rule_id in the RankedRules array (not regex)
7. Construct ComplianceAnalysis object with full traceability

**Validation**:
- `entity_features`: Must match EntityFeatures schema
- `rules`: Non-empty array of RankedRules
- `Param_prompt_template`: File must exist, be valid markdown, use valid Jinja2 syntax (`{{ variable }}`)
- `Param_tools`: Array of valid tool names (must be recognized by Claude API)
- Each citation in output: rule_id must exist in the RankedRules array; cited_text must match the corresponding rule's text field exactly
- Citation lookup: Use rule_id to find the rule in RankedRules; do NOT use regex to extract or validate rule_id
- If citation validation fails: flag in output with error "Citation {rule_id} not found in retrieved rules" or "Citation {rule_id} text mismatch: expected '[actual_text]' but model cited '[cited_text]'"

**Error Handling**:
- Prompt template missing: Return error "Prompt template not found at {path}"
- Claude API unavailable: Return error "LLM service unavailable. Unable to reason over rules."
- Citation validation fails: Include warning in ComplianceAnalysis.reasoning_log; do not halt (compliance analyst should review)
- Empty reasoning output: Return error "LLM produced no usable reasoning output"

**Citation Validation** (sub-process):
1. For each citation in Claude's response:
   - Extract rule_id and cited_text
   - Find rule in source rules by rule_id
   - Check if cited_text appears verbatim in rule.text (allow for whitespace normalization)
   - If mismatch: log warning with both texts for compliance review

### approval_gate

**Purpose**: Present compliance analysis to user for formal review. Request explicit approval or rejection with approver identity. Log formal approval decision and audit trail.

**Input**:
- `analysis` (ComplianceAnalysis object): Output from claude_reasoning

**Configuration** (from YAML node config):
- `Param_required_approval.Param_role` (string): Required role for approval (e.g., "compliance_officer")
- `Param_required_approval.Param_record_identity` (boolean): Whether to capture approver name and email (required for compliance audit)

**Output**:
- `ApprovalDecision` (object) with fields:
  - `approved` (boolean): true if approved, false if rejected
  - `approver_name` (string): Name of approving officer (required if record_identity=true)
  - `approver_email` (string): Email of approving officer (required if record_identity=true)
  - `approver_role` (string): Role confirmation (e.g., "Compliance Officer")
  - `timestamp` (string): ISO 8601 timestamp of decision (UTC) — **required**, must be captured at decision time
  - `actions_carried_out` (array of strings): List of actions executed during approval (e.g., ["approved", "identity_verified", "snapshot_recorded"]) — **required**, documents what occurred
  - `comments` (string): Optional notes from approver
  - `analysis_snapshot` (object): Full ComplianceAnalysis at time of approval (for immutability)

**Process**:
1. Format compliance analysis for human review (readable summary + full citations + reasoning log)
2. Present analysis to user with formal approval request: "Do you formally approve this compliance analysis? (Approve / Reject)"
3. User provides formal decision:
   - Choice: Approve or Reject
   - If record_identity=true: User provides name and email for formal signature
   - Optional comments field for feedback on analysis quality
4. Validate approver identity:
   - If record_identity=true: approver_name and approver_email required (email domain validated)
   - If record_identity=false: use system/conversation metadata (not recommended for regulated use)
5. Record decision timestamp (in UTC) at moment of formal approval
6. Create immutable snapshot: store copy of analysis as it was at approval time
7. Log formal approval decision (decision, approver identity, timestamp, comments, analysis snapshot) for audit trail
8. Return ApprovalDecision object

**Validation**:
- `analysis`: Must be valid ComplianceAnalysis object with non-empty citations and reasoning_log
- `Param_required_approval.Param_role`: Must be in configured roles list (e.g., ["compliance_officer", "senior_analyst"])
- If `Param_required_approval.Param_record_identity` is true: approver_name and approver_email are required before approval is final
- approver_email: Must match domain whitelist (e.g., @company.com) to prevent external approvals
- `timestamp`: Must be captured in UTC, non-empty ISO 8601 format
- `actions_carried_out`: Must be non-empty array documenting all executed actions

**Error Handling**:
- Analysis missing required fields: Return error "ComplianceAnalysis incomplete: missing {field_name}"
- No approver available: Return error "No {role} available for approval. Escalate to {escalation_contact}"
- Approval timeout (48h): Return error "Approval pending for >48h. Escalate to {escalation_contact}. Auto-reject recommended."
- approver_email fails validation: Return error "Approver email invalid or not authorized: {email}"
- Approver rejects analysis: Store rejection with comments; return ApprovalDecision.approved=false; halt workflow
