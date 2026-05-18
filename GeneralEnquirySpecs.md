# General Enquiry Workflow Specification

This document specifies the **general_enquiry** workflow, which is the first implementation of the Compliance Agent Harness. It includes the YAML workflow definition and detailed specifications for each action type used in this workflow.

The general_enquiry workflow handles: product/service description → feature extraction → entry retrieval → compliance reasoning → human approval.

Other workflows (regulatory_change_analysis, incident_investigation, etc.) will have similar specifications files, following this template.

## Workflow Definition

```yaml
harness:
  data_sources:
    fca_handbook:
      artifact: "FCA_Handbook_Text_And_Embeddings"
      version: "2026-Q1"
      model: "voyage-3-large"
      weighting_config: "weights.yaml (schema in StructuredSearch.md)"
  
  workflows:
    general_enquiry:
      nodes:
        - name: validate_scope
          action: validate_scope
          input: question
          output: ScopeValidation
          config:
            Param_require_explicit_scope: true
        
        - name: extract_features
          action: parse_markdown
          input: entity_description
          output: EntityFeatures
          config: {}
        
        - name: check_terminology
          action: glossary_lookup
          input:
            entity_features: extract_features
            question: question
          output: TerminologyMapped
          config:
            Param_glossary_piece: "Glossary"
        
        - name: embed_question
          action: embed_text
          input: question
          output: QuestionEmbedding
          config: {}
        
        - name: retrieve_entries
          action: semantic_search
          input:
            entity_features: extract_features
            question_terms: check_terminology
            question_embedding: embed_question
          output: RankedEntries
          config:
            Param_top_k: 20
            Param_source_version: "2026-Q1"
        
        - name: analyze_compliance
          action: claude_reasoning
          input:
            entity_features: extract_features
            entries: retrieve_entries
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

**Naming convention:** 
- Fields prefixed with `Param_` are configurable parameters defined in the Action Specification below. Changing these values will affect behavior (if implemented in Python). 
- Fields without the prefix are structural: `name` (node identifier), `action` (action type), `input`/`output` (data flow).
- The implementation method (regex, embeddings, LLM, etc.) is specified explicitly in each action's **Process** section below for transparency and auditability.

Each node follows the same structure: `name`, `action`, `input` (if applicable), `output`, and `config` (containing `Param_*` parameters). Action-specific fields are normalized under `config`, making the structure consistent for the Python execution loop.

---

## Action Specifications

Each action type is implemented by a Python class that conforms to an exact specification. This ensures auditability: any auditor or implementer can read the spec and verify that the Python code matches it.

**Message Protocol**: Each action includes a **Messaging** section specifying what status updates the Harness sends to the user during execution. Messages serve three purposes: (1) feedback on progress, (2) debugging/transparency on workflow state, (3) audit trail documentation. All messages are logged to both user output and `interactions.json` for compliance review.

**Message Destinations**:
- **User output**: Printed to stdout (or streamed via websocket in service deployments). Includes symbol prefix (✓, ⚠, ℹ, ✗, →) based on message_type.
- **Audit trail (interactions.json)**: Unstyled text + timestamp + message_type. Always appended, never filtered.
- **Interactive prompts** (validate_scope only): Block execution, wait for user confirmation. Both action and user response logged.

**Note**: Message text below is unstyled; the implementation adds symbols (✓, ⚠, ℹ, ✗, →) based on `message_type` ("complete", "warning", "status", "error", "progress"). Pass text without symbols to `self.message(text, message_type)`.

### validate_scope

**Purpose**: Validate that the user question concerns FCA Handbook compliance specifically (not PRA Requirements, technical standards, or other FCA documents). Acts as gatekeeper before workflow execution.

**Input**:
- `question` (string): User's compliance question, with conversation context

**Output**:
- `ScopeValidation` (object) with fields:
  - `valid` (boolean): Whether question passes scope validation
  - `reason` (string): Explanation (e.g., "Scope is clear and valid" or "Question out of scope: asks about investment strategy, not FCA Handbook compliance")

**Configuration** (from YAML node config):
- `Param_require_explicit_scope` (boolean, default true): If true, ambiguous questions are rejected unless user explicitly confirms FCA Handbook scope. If false, ambiguous questions proceed.

**Process** (deterministic scope checking; no LLM):
1. Analyze question to determine scope
2. If scope is clear (mentions "FCA Handbook" explicitly): valid = true
3. If scope is ambiguous: 
   - If Param_require_explicit_scope is true: ask clarifying question "Do you mean compliance with the **FCA Handbook**?" and wait for confirmation
   - If false: valid = true (proceed with ambiguous scope)
4. If scope is out-of-bounds (e.g., "What's the best investment strategy?"): valid = false, reason explains what harness covers

**Validation**:
- `question`: Non-empty string, min 5 chars, max 1000 chars

**Error Handling**:
- Scope validation failure: Return ScopeValidation with valid=false and explanation; do not proceed to subsequent nodes
- Ambiguous scope + Param_require_explicit_scope=true: Prompt user for clarification; if no confirmation received, reject with valid=false

**Messaging** (user-facing updates):
- On start: "Validating compliance scope..."
- On scope valid: "FCA Handbook compliance question confirmed" (message_type: "complete")
- On scope ambiguous (requires clarification): "Your question scope is ambiguous. Do you mean FCA Handbook compliance?" (message_type: "status"; inline prompt)
- On scope invalid: "Question out of scope. This harness covers FCA Handbook compliance only: {reason}" (message_type: "error")
- Message destination: User (all types)

**Notes**:
- Scope validation may vary by workflow. Some workflows may accept broader scope; this specification applies to general_enquiry only.
- Deterministic and auditable; no LLM involved in scope checking.

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

**Process** (using regex for markdown parsing only; no LLM):
1. Parse markdown structure (headings, sections) using regex — extract h1, section headers, list items
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

**Messaging** (user-facing updates):
- On start: "Extracting product features from your description..." (message_type: "status")
- On complete: "Extracted {entity_name}: {feature_count} features, {use_case_count} use cases identified" (message_type: "complete")
- On error: "Could not parse product description. Please check markdown format." (message_type: "error")
- Message destination: User (all types)

### glossary_lookup

**Purpose**: Map user question terminology to FCA Handbook canonical terms via the FCA Glossary. Handles terminology mismatches where user's language differs from regulatory language (e.g., "money handling" → "cash", "deposit" → "advance payment"). Improves semantic search recall on user phrasing that does not directly match handbook terminology.

**Input**:
- `entity_features` (EntityFeatures object): Structured product information
- `question` (string): User's compliance question

**Configuration** (from YAML node config):
- `Param_glossary_piece` (string): FCA Handbook piece containing glossary definitions (fixed: "Glossary")

**Output**:
- `TerminologyMapped` (object) with fields:
  - `mapped_terms` (array of objects): For each term found in FCA Glossary:
    - `user_term` (string): Original term from user question or entity_features (e.g., "bitcoin")
    - `canonical_term` (string): FCA Glossary canonical entry (e.g., "Digital Currencies")
    - `glossary_entry` (string): Verbatim definition from FCA Glossary
  - `unmapped_terms` (array of strings): Terms not found in FCA Glossary (user's original phrasing)
  - `all_search_terms` (array of strings): Union of unmapped_terms and canonical_terms (used by semantic_search)

**Process** (deterministic FCA Glossary lookup; no LLM):
1. Extract key noun phrases from question and entity_features (capitalized terms, multi-word phrases)
2. For each term, search FCA Glossary (Glossary piece from FCA_Handbook_Text_And_Embeddings) using **hierarchical matching strategy**:
   - **Stage 1: Exact match** — Check for exact term match in glossary (case-insensitive)
   - **Stage 2: Prefix match** — If exact fails, check for glossary entries starting with the term
   - **Stage 3: Fuzzy semantic match** — If prefix fails, embed the term and compute cosine similarity against pre-computed glossary embeddings; return match if similarity > 0.7 threshold
3. If any stage finds a match: record canonical term and glossary definition; do not proceed to later stages
4. If **no match found in any stage**: add term to unmapped_terms (pass through unchanged); term not covered in FCA Glossary
5. Compile all_search_terms: both mapped canonical terms and unmapped user terms
6. Return TerminologyMapped object

**Fallback behavior when term not found in glossary:**
- No error is raised; unmapped term is included in all_search_terms
- semantic_search receives the original term as-is (no enrichment from glossary)
- Search proceeds with full original question + unmapped term (graceful degradation)
- This preserves coverage for regulatory language that may not be in the glossary

**Validation**:
- `entity_features`: Must match EntityFeatures schema
- `question`: Non-empty string, min 5 chars, max 1000 chars
- FCA Glossary: Must be available in FCA_Handbook_Text_And_Embeddings

**Error Handling**:
- FCA Glossary unavailable: Return error "FCA Handbook Glossary not available. Unable to perform terminology mapping."
- No terms extracted: Return TerminologyMapped with empty mapped_terms and all unmapped (not an error—semantic_search proceeds with original terms)
- Glossary lookup fails (data corruption): Return error "Error querying FCA Glossary. Check FCA_Handbook_Text_And_Embeddings integrity."

**Messaging** (user-facing updates):
- On start: "Mapping terminology to FCA Glossary..." (message_type: "status")
- On complete: "Mapped {mapped_count} terms to canonical FCA language ({unmapped_count} unmapped)" (message_type: "complete")
- If no terms found: "No glossary terms found; proceeding with original question terms" (message_type: "warning")
- On error: "Glossary lookup failed. Proceeding with original question terms." (message_type: "warning")
- Message destination: User (all types)

**Notes**:
- Unmapped terms (not found in FCA Glossary) are passed to semantic_search unchanged as fallback
- This node does not modify the user's question—it enriches it with canonical alternatives
- Glossary matching is deterministic and auditable; no LLM involved
- **Matching strategy rationale**: Exact/prefix are fast (O(n) lookup); fuzzy adds semantic understanding (e.g., "bitcoin" → "Digital Currencies") at cost of 1 embedding API call per unfound term
- **Threshold for fuzzy match**: 0.7 cosine similarity (tunable per deployment); matches must exceed threshold to be considered valid
- **API calls**: Fuzzy matching may call embedding API if exact/prefix fail; credentials (VOYAGE_API_KEY or OPENAI_API_KEY) must be configured
- **Glossary completeness assumption**: Glossary is expected to cover most FCA Handbook terminology; not being in glossary does not mean term is invalid, only that it requires semantic search enrichment

### embed_text

**Purpose**: Embed the user's compliance question using the same embedding model as FCA_Handbook_Text_And_Embeddings. This is foundational: embeddings from different models are incompatible and will produce meaningless search results.

**Input**:
- `question` (string): User's compliance question

**Output**:
- `QuestionEmbedding` (vector): Embedded question vector (dimensionality matches handbook embeddings)

**Configuration** (from YAML node config):
- None (uses embedding model from harness data_sources.fca_handbook.model)

**Process** (deterministic embedding only; no LLM):
1. Detect embedding model from harness configuration (voyag-3-large or text-embedding-3-large)
2. Call embedding API with question text
3. Return embedding vector
4. Embedding vector is passed to semantic_search for cosine similarity computation

**Validation**:
- `question`: Non-empty string, max 1000 chars
- Embedding model configured and available in harness

**Error Handling**:
- Embedding API unavailable: Return error "Embedding service unavailable. Check API credentials (VOYAGE_API_KEY or OPENAI_API_KEY) and network connectivity."
- Embedding API rate limit: Return error "Embedding service rate limited. Retry after delay."
- Model mismatch: If embedding model differs from handbook embeddings, results will be meaningless but no error raised (caught by search result quality checks downstream)

**Messaging** (user-facing updates):
- On start: "Embedding your question..." (message_type: "status")
- On complete: "Question embedded ({embedding_model}, {embedding_dimensions} dimensions)" (message_type: "complete")
- On error: "Embedding service unavailable. Check API credentials and network connectivity." (message_type: "error")
- Message destination: User (all types)

**Notes**:
- This node must use the same embedding model as FCA_Handbook_Text_And_Embeddings (configured at harness startup)
- Embedding happens per question (every user query)
- Handbook embedding happens once at startup (see EmbeddingModel.md)
- If model is switched after harness startup, handbook and question embeddings become incompatible; harness must restart to reload handbook

### semantic_search

**Purpose**: Query `FCA_Handbook_Text_And_Embeddings` for regulatory entries matching entity features and question terminology. Apply regulatory weighting to rank results by binding authority and relevance. Uses FCA Glossary-mapped terminology (from glossary_lookup) alongside user's original question to improve recall on terminology mismatches.

**Input**:
- `entity_features` (EntityFeatures object): Structured output from parse_markdown
- `question_embedding` (vector): Embedded question from embed_text node
- `question_terms` (TerminologyMapped object): Mapped FCA Glossary terms from glossary_lookup node

**Configuration** (from YAML node config):
- `Param_top_k` (integer): Number of results to return (default 20, range 1–100)

**Implementation Constants** (hardcoded, not parameterized):
- Data source: `FCA_Handbook_Text_And_Embeddings` (from harness configuration)
- Weighting: Always enabled (regulatory weighting always applied)

**Output**:
- `RankedEntries` (array of objects) with fields:
  - `entry_id` (string): FCA entry identifier (e.g., "COBS 2.1.1R")
  - `text` (string): Verbatim entry text from source
  - `base_similarity` (number, 0–1): Cosine similarity from embeddings
  - `weight_factors` (object):
    - `entry_type_weight` (number): Binding authority multiplier
    - `hierarchy_multiplier` (number): Position in handbook structure
    - `importance_multiplier` (number): Regulatory criticality
    - `piece_weight` (number): Section weighting
  - `final_score` (number): base_similarity × entry_type_weight × hierarchy_multiplier × importance_multiplier × piece_weight
  - `source_version` (string): Version of FCA_Handbook_Text_And_Embeddings used (e.g., "2026-Q1")

**Process** (deterministic weighting only; embedding delegated to embed_text node; no LLM):
1. Extract search terms: Combine all_search_terms from question_terms (mapped FCA Glossary terms + unmapped fallback terms)
2. Receive pre-computed question_embedding from embed_text node (ensures question and handbook use same embedding model)
3. Compute cosine similarity (numpy dot product) between question_embedding and entry embeddings from pre-loaded FCA_Handbook_Text_And_Embeddings
4. Sort by cosine similarity, select top 50 candidates (deterministic filtering before weighting)
5. Apply regulatory weighting algorithm (see StructuredSearch.md) to rank candidates by final_score: base_similarity × entry_type_weight × hierarchy_multiplier × importance_multiplier × piece_weight
6. Sort by final_score descending (deterministic ranking)
7. Return top_k results with full weight_factors breakdown for auditability

**Validation**:
- `entity_features`: Must contain entity_type and features (validate EntityFeatures structure)
- `question_embedding`: Must be a vector of correct dimensionality (matches handbook embeddings)
- `question_terms`: Must be valid TerminologyMapped object
- `Param_top_k`: Integer in range [1, 100], default 20
- Harness configuration: weights.yaml (schema in StructuredSearch.md) must be present and valid; if missing or invalid: raise error with config path

**Error Handling**:
- Data source unavailable: Return error "FCA Handbook data (FCA_Handbook_Text_And_Embeddings) is temporarily unavailable. Escalate to compliance_team@company.com"
- Invalid weights.yaml (schema in StructuredSearch.md): Return error "Regulatory weights configuration invalid at {path}: {reason}"
- No results found: Return empty array with informational log (not an error—some queries legitimately have no matches)

**Messaging** (user-facing updates):
- On start: "Searching FCA Handbook for relevant entries..." (message_type: "status")
- On progress (after candidate filtering): "Retrieved {candidate_count} candidates from {total_entries} entries" (message_type: "progress")
- On progress (during weighting): "Applying regulatory weights to rank results..." (message_type: "progress")
- On complete: "Retrieved {top_k} ranked entries ({top_1_score:.2f} similarity, {top_1_weight:.1f}x multiplier)" (message_type: "complete")
- If no results: "No matching entries found for your question" (message_type: "warning")
- On error: "Search failed. Check FCA Handbook data and weights configuration." (message_type: "error")
- Message destination: User (all types; progressive updates)

### claude_reasoning

**Purpose**: Invoke Claude to reason over retrieved entries and product features, producing a compliance analysis with citations and reasoning logs.

**Input**:
- `entity_features` (EntityFeatures object): Structured product information
- `entries` (RankedEntries array): Top-ranked FCA entries from semantic_search

**Configuration** (from YAML node config):
- `Param_tools` (array of strings): Internal tool names available to Claude (e.g., ["citation_formatter", "audit_logger"])
- `Param_prompt_template` (string): Path to prompt template file (e.g., "fca-compliance-analyst.md")

**Output**:
- `ComplianceAnalysis` (object) with fields:
  - `answer` (string): Claude's reasoning and conclusions
  - `citations` (array of objects):
    - `entry_id` (string): FCA entry cited (e.g., "COBS 2.1.1R")
    - `cited_text` (string): Exact verbatim text from FCA Handbook
    - `context` (string): Sentence or paragraph from entry showing why it applies
    - `binding_level` (string): "R", "G", "E", or "D" (from entry_id suffix)
  - `reasoning_log` (object):
    - `reasoning_chain` (string): Claude's step-by-step logic ("entry X applies because feature Y matches criterion Z")
    - `gaps_identified` (array of strings): Edge cases or areas of uncertainty
    - `confidence_score` (number, 0–1): Model's confidence in the analysis
  - `timestamp` (string): ISO 8601 timestamp of analysis

**Process** (LLM reasoning with deterministic citation validation):
1. Load Jinja2 prompt template from file (e.g., `harness/prompts/fca-compliance-analyst.md`)
2. Construct system prompt by rendering template with entity_features and entries (deterministic). Entries include entry_id (authoritative identifier) for each RankedEntry. Template syntax: `{{ entity_features }}`, `{{ ranked_entries }}`
3. Enable prompt caching for stable context (deterministic optimization)
4. Call Claude API (LLM reasoning only — do not modify retrieved entries):
   - System prompt (cached) — includes RankedEntries with entry_id fields
   - User message: question + available internal tools description
   - Internal tools: citation_formatter, audit_logger (structured output enforcement; citation_formatter must extract entry_id from tool call)
   - Model: claude-opus with thinking tokens enabled
   - Temperature: 0.5 (deterministic but thoughtful)
   - Max tokens: 2000
   - Extended thinking: enabled
5. Parse Claude's response to extract reasoning_log, citations (by entry_id from tool_use), and answer text
6. Validate each citation using deterministic entry_id lookup in RankedEntries array (not regex, not LLM-based)
7. Construct ComplianceAnalysis object with full traceability

**Validation**:
- `entity_features`: Must match EntityFeatures schema
- `entries`: Non-empty array of RankedEntries
- `Param_prompt_template`: File must exist, be valid markdown, use valid Jinja2 syntax (`{{ variable }}`)
- `Param_tools`: Array of valid tool names (must be recognized by Claude API)
- Each citation in output: entry_id must exist in the RankedEntries array; cited_text must match the corresponding entry's text field exactly
- Citation lookup: Use entry_id to find the entry in RankedEntries; do NOT use regex to extract or validate entry_id
- If citation validation fails: flag in output with error "Citation {entry_id} not found in retrieved entries" or "Citation {entry_id} text mismatch: expected '[actual_text]' but model cited '[cited_text]'"

**Error Handling**:
- Prompt template missing: Return error "Prompt template not found at {path}"
- Claude API unavailable: Return error "LLM service unavailable. Unable to reason over entries."
- Citation validation fails: Include warning in ComplianceAnalysis.reasoning_log; do not halt (compliance analyst should review)
- Empty reasoning output: Return error "LLM produced no usable reasoning output"

**Messaging** (user-facing updates):
- On start: "Analyzing compliance requirements with Claude..." (message_type: "status")
- On Claude thinking: "Claude is reasoning (thinking tokens enabled)..." (message_type: "progress")
- On complete: "Analysis complete. {citation_count} entries identified, confidence: {confidence_score:.0%}" (message_type: "complete")
- On citation warning: "Citation validation warning: {warning_message}" (message_type: "warning")
- On error: "LLM analysis failed. Please try again." (message_type: "error")
- Message destination: User (all types; progressive transparency)

**Citation Validation** (sub-process):
1. For each citation in Claude's response:
   - Extract entry_id and cited_text
   - Find entry in source entries by entry_id
   - Check if cited_text appears verbatim in entry.text (allow for whitespace normalization)
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

**Process** (human formal approval only; no LLM):
1. Format compliance analysis for human review (readable summary + full citations + reasoning log)
2. Present analysis to user with formal approval request: "Do you formally approve this compliance analysis? (Approve / Reject)"
3. Capture human decision (deterministic):
   - Choice: Approve or Reject (binary)
   - If record_identity=true: Capture name and email from user (formal signature)
   - Optional comments field for feedback
4. Validate approver identity (deterministic):
   - If record_identity=true: Require approver_name and approver_email; validate email domain against whitelist
   - If record_identity=false: Use system/conversation metadata (not recommended for regulated use)
5. Record decision timestamp in UTC at moment of approval (deterministic)
6. Create immutable snapshot: store copy of analysis as it was at approval time (deterministic)
7. Log formal approval decision for audit trail (deterministic)
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

**Messaging** (user-facing updates):
- On start: "Analysis ready for compliance review. Awaiting approval from {required_role}..." (message_type: "status")
- On approval: "Approved by {approver_name} ({approver_role}) at {timestamp}" (message_type: "complete")
- On rejection: "Rejected by {approver_name} ({approver_role}). Reason: {comments}" (message_type: "error")
- On timeout (48h): "Approval request timed out after 48 hours. Escalate to compliance_team@company.com" (message_type: "error")
- Message destination: User (all types); audit trail includes messages PLUS ApprovalDecision object with identity, timestamp, and actions_carried_out

**Audit Trail Integration**:
The approval_gate action returns ApprovalDecision with complete audit metadata. The harness logs both:
1. Messages (as with all actions): unstyled text + timestamp + type
2. ApprovalDecision object: approver_name, approver_email, approver_role, timestamp (decision time), actions_carried_out

Together these provide full audit trail: what the system asked, who approved/rejected, when, and what actions were taken. An auditor reviewing `interactions.json` will find both the message progression and the formal ApprovalDecision metadata.
