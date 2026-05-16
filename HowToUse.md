# How to Use the FCA Handbook Compliance Agent Harness

This document guides you through deploying and using the FCA Handbook Compliance Agent Harness for regulatory compliance analysis.

## What This Harness Does

The harness answers: **"Which FCA Handbook rules apply to [product/service], and are we compliant?"**

It works by:
1. Extracting key features from your product/service description
2. Mapping your terminology to FCA Handbook canonical terms (glossary lookup)
3. Searching the FCA Handbook for relevant rules using semantic search + regulatory weighting
4. Using Claude to reason over retrieved rules and produce a compliance analysis
5. Presenting the analysis for human review and approval

## Before You Start

### Prerequisites

You'll need:

1. **FCA_Handbook_Text_And_Embeddings.json** — A JSON file containing FCA Handbook rules with pre-computed embeddings
   - The template (FCA_Handbook_Template_PRIN.json) is included; you need the full handbook
   - Embeddings must be computed using either Voyage AI (voyage-3-large) or OpenAI (text-embedding-3-large)
   - See EmbeddingModel.md for embedding model selection rationale

2. **API Credentials** (set as environment variables):
   - `ANTHROPIC_API_KEY` — For Claude API (reasoning layer)
   - `VOYAGE_API_KEY` OR `OPENAI_API_KEY` — For embedding API (semantic search)

3. **Python 3.9+** with dependencies:
   ```bash
   pip install anthropic requests numpy
   ```

### Implementation Status

This reference implementation is ~65% complete. Before production use, you must:

1. **Implement glossary lookup** — See PythonImplementation.md line 590; specify how to query your FCA Glossary data
2. **Implement approval gate UI/flow** — See PythonImplementation.md line 1087; adapt to your approval system (web UI, CLI, async approval)
3. **Configure audit logging** — Specify where interaction logs are stored (file, database, or event stream)

See HowToUse.md § Implementation Checklist below for details.

## Getting Started: Your First Compliance Analysis

### Step 1: Configure the Harness

Copy `harness.yaml` and adapt it for your environment:

```yaml
harness:
  data_sources:
    fca_handbook:
      artifact: "path/to/FCA_Handbook_Text_And_Embeddings.json"
      version: "2026-Q1"
      model: "voyage-3-large"  # or "text-embedding-3-large"
```

### Step 2: Prepare Your Prompt Template

The included `fca-compliance-analyst.md` is a Jinja2 template that Claude uses for reasoning. Customize if needed:

```markdown
# FCA Handbook Compliance Analysis Prompt

You are an expert financial regulatory compliance analyst...
[Your instructions to Claude]

## Entity Information
{{ entity_features }}

## Retrieved FCA Handbook Rules
{{ ranked_rules }}
```

Variables available:
- `{{ entity_features }}` — Product/service being analyzed (name, type, features)
- `{{ ranked_rules }}` — Top FCA Handbook rules retrieved via semantic search

### Step 3: Run a Query

Invoke the harness with a product description:

```python
from harness import ComplianceHarness

harness = ComplianceHarness('harness.yaml')

result = harness.execute_workflow(
    workflow='general_enquiry',
    entity_description="""
    # Robo-Advisor Platform
    
    - Entity Type: Investment Management Platform
    - Features:
      - Automated portfolio recommendation engine
      - Discretionary management on behalf of clients
      - Real-time rebalancing
      - Fee-based advisory (0.5% annually)
    """,
    question="Which COBS rules apply to our discretionary management service?"
)

print(result['answer'])
print(f"Confidence: {result['reasoning_log']['confidence_score']}")
for citation in result['citations']:
    print(f"  - {citation['rule_id']}: {citation['cited_text']}")
```

**Output:**
```json
{
  "answer": "Your service is subject to COBS 2 (Conduct of business) as a discretionary investment manager...",
  "citations": [
    {
      "rule_id": "COBS 2.1.1R",
      "cited_text": "A firm must ensure that it is ready, willing and organized to execute orders...",
      "binding_level": "R"
    }
  ],
  "reasoning_log": {
    "reasoning_chain": "Rule X applies because...",
    "gaps_identified": ["Unclear whether Section Y applies to your use case"],
    "confidence_score": 0.92
  },
  "timestamp": "2026-05-16T10:30:00Z"
}
```

## Workflow: From Query to Approval

The harness executes this workflow:

```
1. extract_features
   Input: Product description (markdown)
   Output: Structured features (name, type, features, use_cases)

2. check_terminology
   Input: Features + question
   Process: Map user terminology to FCA Handbook canonical terms
   Output: Mapped terms + original terms
   
3. retrieve_rules
   Input: Features + mapped terminology + question
   Process: Semantic search + regulatory weighting
   Output: Top 20 FCA Handbook rules ranked by relevance
   
4. analyze_compliance
   Input: Features + rules
   Process: Claude reasons over rules and produces analysis
   Output: Compliance analysis with citations and confidence score
   
5. human_review
   Input: Compliance analysis
   Process: Present for human approval
   Output: Approval decision + timestamp
```

**Each step is independently testable.** You can mock steps 3–4 to test approval flow, or mock step 5 to test reasoning without waiting for human approval.

## Customization Points

### Changing the Embedding Model

In `harness.yaml`, change the `model` field:

```yaml
model: "voyage-3-large"        # Voyage AI (legal domain optimized)
# or
model: "text-embedding-3-large"  # OpenAI (general purpose)
```

The harness detects the model at runtime and calls the appropriate API. No code changes needed.

**Important:** The model must match the embeddings in your FCA_Handbook_Text_And_Embeddings.json file.

### Adjusting Search Results

In `harness.yaml`, set `Param_top_k`:

```yaml
semantic_search:
  config:
    Param_top_k: 20  # Return top 20 rules (default)
```

Range: 1–100. More results = more context for Claude but slower analysis.

### Tuning Regulatory Weights

See StructuredSearch.md § Hardcoded Weights for the weighting algorithm. Weights are currently hardcoded in PythonImplementation.md line ~700. To adjust:

1. Edit `HandbookIndex.__init__()` weights configuration
2. Update commit message with rationale (audit trail)
3. Restart harness (no re-embedding needed)

**Examples:**
- Conservative compliance: Reduce GUIDANCE weight from 0.8 to 0.6 (stricter)
- Heightened scrutiny: Increase importance multipliers for high-risk rules

### Adapting for Other Regulatory Domains

To use this harness for non-FCA regulations:

1. **Prepare your regulatory data**: Create a JSON file with your codified rules + embeddings
   - Schema: Same as FCA_Handbook_Text_And_Embeddings (see template)
   - Embeddings: Use same embedding model as harness configuration

2. **Adjust regulatory weights** (StructuredSearch.md § Weighting Factors):
   - Rule type weights: Adjust based on your regulatory hierarchy (rules vs. guidance vs. examples)
   - Importance multipliers: Calibrate for your domain's critical areas
   - Piece weights: Update for your document structure

3. **Customize the prompt** (fca-compliance-analyst.md):
   - Replace FCA-specific guidance with your regulatory domain guidance
   - Adjust tone and emphasis for your compliance context

4. **Adapt approval workflow** (PythonImplementation.md § ApprovalGateAction):
   - Connect to your approval system
   - Customize role-based access (who can approve)

## Implementation Checklist

Before deploying to production, complete these tasks:

- [ ] **Data**: Prepare FCA_Handbook_Text_And_Embeddings.json with embeddings
- [ ] **Config**: Copy harness.yaml, set artifact path and embedding model
- [ ] **Secrets**: Set ANTHROPIC_API_KEY, VOYAGE_API_KEY (or OPENAI_API_KEY)
- [ ] **Glossary**: Implement `GlossaryLookupAction._lookup_glossary()` to query your FCA Glossary records
- [ ] **Approval UI**: Implement `ApprovalGateAction` approval flow (web UI, CLI, or async service)
- [ ] **Audit Logging**: Configure where interaction logs are persisted (file, database, or event stream)
- [ ] **Testing**: Run end-to-end test with real FCA Handbook and real API calls
- [ ] **Error Scenarios**: Test failure cases (missing rules, API timeouts, malformed LLM responses)
- [ ] **Performance**: Verify response times meet your SLA (typical: <30 seconds per analysis)
- [ ] **Validation**: Audit compliance against GeneralEnquirySpecs.md and StructuredSearch.md

## Common Issues

### "Embedding service unavailable"

**Problem**: VOYAGE_API_KEY or OPENAI_API_KEY not set

**Fix**:
```bash
export VOYAGE_API_KEY="your-api-key"  # If using Voyage AI
# or
export OPENAI_API_KEY="your-api-key"  # If using OpenAI
```

### "FCA Handbook data not found"

**Problem**: FCA_Handbook_Text_And_Embeddings.json missing

**Fix**: Check path in harness.yaml; ensure file exists and contains embeddings

### "Citation rule_id not found in retrieved rules"

**Problem**: Claude cited a rule that wasn't in the semantic search results

**Fix**: This is logged as a warning but doesn't block the analysis. Review the citation manually; may indicate your top_k is too low. Try increasing `Param_top_k` in harness.yaml.

### "Glossary term not found"

**Problem**: User terminology doesn't match FCA Glossary

**Expected behavior**: Glossary lookup returns the original term unmapped; semantic search proceeds with the user's phrasing. This is graceful degradation, not an error.

## Further Reading

- **Understand the architecture**: README.md
- **Understand the specifications**: GeneralEnquirySpecs.md, StructuredSearch.md
- **Understand the code**: PythonImplementation.md
- **Choose embedding model**: EmbeddingModel.md
- **See UI/demo design**: UIStrategy.md
