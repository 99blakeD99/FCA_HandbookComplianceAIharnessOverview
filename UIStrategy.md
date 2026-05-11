# UI Strategy: Minimal Demo Application

## Overview

This document captures the UI design, architecture, and deployment strategy for a minimal demo application that showcases the Compliance Agent Harness. The demo serves two purposes:

1. **Internal Testing**: Validate the harness implementation against real workflows, catch integration bugs, and refine process design
2. **FCA Demonstration**: Give the FCA hands-on experience with the system, gather feedback on usability and results, and support the case for FCA Handbook AI-accessibility

## Guiding Principles

1. **Minimal and Focused** — Demo only the core workflow. No admin panels, no database management UIs. Just product + question → analysis + approval.
2. **Working Code Over Specification** — The demo is how we build confidence. Issues found in running code matter more than perfect specs.
3. **Feedback Loop** — The approval gate is deliberately designed to capture approver feedback (via comments) so we can iterate on the analysis quality.
4. **Transparency** — Users see the full reasoning chain, citations, weights, and confidence. No black boxes.
5. **Professional but Honest** — Polished enough for FCA review, but clearly labeled as demo/MVP ("not authoritative").

## User Flows

### Internal Tester Flow

```
1. Open demo app (internal link, no auth)
2. Enter product description (markdown format)
3. Enter compliance question
4. Click "Analyze"
5. See workflow progress (parsing → searching → reasoning → approval)
6. Review compliance analysis:
   - Claude's answer
   - Cited rules (with rule IDs, binding level, relevance scores)
   - Reasoning log (chain of logic, gaps identified, confidence score)
7. Approval panel:
   - Review analysis
   - Choose: Approve / Reject
   - Optional: Add comments (used for process refinement)
   - Submit
8. See ApprovalDecision summary with approver identity and timestamp
9. Download full audit trail (JSON) for testing
```

### FCA User Flow

```
1. Receive link + password (controlled distribution)
2. Open demo, authenticate
3. [Same as internal flow, steps 2-9]
4. Additionally: See explanatory text about:
   - What this demo shows (end-to-end harness workflow)
   - What it doesn't show (not integrated with live systems, not authoritative guidance)
   - Why this matters (FCA Handbook AI-accessibility as infrastructure)
   - Feedback form (structured questions on usability, accuracy, next steps)
```

## Feature Breakdown

### Page 1: Input Form

**Title**: "FCA Handbook Compliance Analysis — Demo"

**Subtitle**: "Test the harness: Enter a product description and compliance question to see how the system retrieves and analyzes applicable FCA rules."

**Inputs**:
- **Product Description** (textarea, ~500 chars):
  - Placeholder with example markdown structure (features, use cases, etc.)
  - Help text: "Markdown format. Include: Features, Use Cases, Data Handled, Decision Authority"
- **Compliance Question** (text input):
  - Placeholder: "Which COBS rules apply to algorithmic advice?"
  - Help text: "Be specific. The system searches FCA Handbook for matching rules."

**Actions**:
- "Analyze" button (runs full workflow)
- "Load Example" button (populates with sample product)

---

### Page 2: Workflow Progress

**While running:**
- Show spinner/progress bar with status:
  - "Parsing product description..."
  - "Retrieving FCA rules (semantic search)..."
  - "Reasoning with Claude..."
  - "Routing to approval..."

**On completion:**
- Hide progress, show results below

---

### Page 3: Compliance Analysis Results

**Section A: Analysis Summary**
- Claude's answer (full text, ~500-2000 chars)
- Confidence score (0-1, colored badge: green >0.8, yellow 0.6-0.8, red <0.6)

**Section B: Cited Rules** (collapsible list)
- For each citation:
  - Rule ID (e.g., "COBS 2.1.1R")
  - Binding Level badge: R (binding), G (guidance), E (evidential), D (deleted)
  - Base similarity score (0-1, how closely matched the question)
  - Weight breakdown (collapsed detail):
    - rule_type_weight, hierarchy_multiplier, importance_multiplier, piece_weight
    - Final score (product of all)
  - Cited text (excerpt, ~200 chars)
  - Full text button (modal or expand)

**Section C: Reasoning Log** (collapsible)
- Reasoning chain (Claude's step-by-step logic)
- Gaps identified (array of edge cases or uncertainties)
- Confidence score explanation

**Transparency note**: "All weights, scores, and reasoning are shown for auditability. This is working code, not authoritative guidance."

---

### Page 4: Approval Gate

**Title**: "Compliance Officer Review"

**Shows**:
- Summary of analysis (above sections collapsed as reference)
- Question: "Do you approve this compliance analysis?"

**Approval Controls**:
- **Approve** button (green)
- **Reject** button (red)
- **Comments** textarea (required or strongly encouraged):
  - Placeholder: "What was helpful? What was unclear? Any concerns? This helps us refine the analysis quality."
  - Character limit: 500
  - Help text: "Your feedback drives iteration on this system."

**On submit**:
- Capture:
  - approver_name (text field, auto-populated if available, editable)
  - approver_email (text field, auto-populated if available, editable)
  - timestamp (captured server-side in UTC)
  - decision (approved/rejected)
  - comments (free text)
- Store snapshot of full ComplianceAnalysis at approval time
- Store all of the above in audit trail

---

### Page 5: Approval Decision Summary

**Shows**:
- Decision: "Approved by [Name] on [Timestamp]" or "Rejected by [Name]"
- Comments from approver (if provided)
- Timestamp (ISO 8601 UTC)
- Actions carried out (audit trail: ["presented_for_review", "identity_verified", "snapshot_recorded"])

**Export/Download**:
- Button to download full JSON audit trail (workflow inputs, node outputs, approval decision)
- Button to copy permalink (for sharing with team)

---

## Approval UI Design Rationale

The approval gate is intentionally designed to collect feedback:

1. **Comments are required (or strongly encouraged)** — Not optional, because we want to learn how to improve the system
2. **Approver name + email captured** — For audit trail and to understand who has reviewed what
3. **Full snapshot stored** — So we can see exactly what the approver saw when they made the decision
4. **Timestamp in UTC** — For compliance audit trail
5. **Actions documented** — "presented_for_review", "identity_verified", "snapshot_recorded" — proves the workflow executed correctly

The comments field becomes a **feedback log** for process refinement. Patterns in approver comments will show:
- Which analyses need clarification
- Which rule citations are confusing
- Whether the reasoning chain is transparent
- What gaps the approver is concerned about

## Technology Stack

### Frontend
- **Framework**: Streamlit (Python-native, minimal DevOps, fast to iterate)
- **Language**: Python (leverages same environment as backend)
- **UI Components**: 
  - Streamlit built-ins (text_input, text_area, button, spinner, tabs)
  - Custom CSS for badges, collapsibles, colors (rule binding level indicators)

### Backend
- **Language**: Python
- **Harness Integration**: Import and call `execute_workflow()` directly from `harness.py`
- **Data**: 
  - In-memory: Handbook embeddings (loaded once at startup)
  - Configuration: weights.yaml, prompt templates
  - Temporary: Approval decisions (could use SQLite or in-memory for demo)

### Hosting
- **Platform**: Streamlit Cloud
- **Authentication**: Streamlit's built-in `streamlit-authenticator` library (simple password)
- **Data**: 
  - Credentials: secrets.toml (managed by Streamlit Cloud)
  - Handbook: Bundled with app (FCA_Handbook_Template_ALL.json)
  - Audit logs: Local filesystem or cloud storage (optional for MVP)

### Deployment Flow
```
1. Push code to GitHub repo
2. Connect Streamlit Cloud to repo
3. Set environment variables (OPENAI_API_KEY, ANTHROPIC_API_KEY)
4. Streamlit auto-deploys on push
5. App URL: https://fca-compliance-harness-demo.streamlit.app
6. Share link + password with internal team and FCA
```

## Configuration & Secrets

**Environment variables** (set in Streamlit Cloud):
```
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-...
```

**App secrets** (.streamlit/secrets.toml, not committed):
```
password = "fca-demo-password"  # Shared with FCA users
internal_password = "internal-testing"  # For internal testing
approver_email_whitelist = ["@company.com", "@fca.org.uk"]
```

## Data Management

### Input Data
- **Product descriptions**: Text entered by user, not stored (unless saving full audit trail)
- **Compliance questions**: Text entered by user, not stored
- **FCA Handbook**: FCA_Handbook_Template_ALL.json (bundled with app, read-only)
- **Weights configuration**: weights.yaml (bundled with app, read-only)

### Output Data
- **Full audit trails**: JSON export option for each workflow execution
  - Includes: user input, all node outputs (ProductFeatures, RankedRules, ComplianceAnalysis, ApprovalDecision)
  - Stored locally on first iteration, cloud storage (S3/GCS) if needed later
- **Approval decisions**: Stored in SQLite (demo) or cloud database (production)

### Privacy/Data Residency
- No data is stored by default (stateless)
- Audit trails are export-on-demand
- FCA data is only the handbook (public info, redistributable)
- User product descriptions are ephemeral (only in-memory during workflow)

## Success Metrics: What We'll Learn

From internal testing:
1. **Integration correctness** — Do all nodes execute without errors?
2. **Data flow** — Does each node's output correctly become the next node's input?
3. **Semantic search quality** — Are retrieved rules relevant to the product description?
4. **Claude reasoning** — Is the analysis clear, complete, and well-cited?
5. **Citation validation** — Do cited rules actually match what the user sees?
6. **Performance** — How long does the full workflow take? (Target: <30s)
7. **Edge cases** — What breaks? (missing fields, unusual products, ambiguous questions)

From FCA feedback:
1. **Usability** — Is the UI clear? Do users understand each section?
2. **Accuracy** — Are the retrieved rules correct for the product?
3. **Completeness** — Are all relevant rules retrieved, or are there gaps?
4. **Reasoning quality** — Is Claude's logic sound and transparent?
5. **Next steps** — What would make this system production-ready for FCA?

## Roadmap

### Phase 1: MVP (Internal Testing)
- [ ] Complete FCA_Handbook_Template_ALL.json (full embeddings)
- [ ] Implement Python harness (all 4 actions)
- [ ] Build Streamlit app (pages 1-5 above)
- [ ] Deploy to Streamlit Cloud
- [ ] Internal testing + iteration (1-2 weeks)
- [ ] Document learnings and bugs found

### Phase 2: FCA Demo (External)
- [ ] Polish UI based on feedback
- [ ] Add FCA-specific explanatory text (what this shows, what it doesn't, why it matters)
- [ ] Structured feedback form for FCA users
- [ ] Give FCA controlled access (link + password)
- [ ] Collect structured feedback (1-2 weeks)
- [ ] Synthesize findings into recommendations

### Phase 3: Refinement (Iteration)
- [ ] Fix bugs found during testing
- [ ] Adjust prompt template or search ranking based on feedback
- [ ] Repeat phases 1-2 if needed
- [ ] Prepare for next level of formalization (production harness, integration)

## Non-Goals

This demo intentionally does NOT include:
- User accounts / role-based access (simple password only)
- Database persistence (audit trails are export-on-demand)
- Integration with live systems (standalone demo)
- Production-grade error recovery (errors fail explicitly)
- Performance optimization (MVP performance is acceptable)
- Multi-language support
- Mobile optimization (desktop-first)

These can be added post-MVP if the system proves valuable.

---

## Summary

The demo app is a **working spec validator**. By building it, we:
1. Prove the harness works end-to-end
2. Find integration bugs that specs miss
3. Gather real feedback on usability and accuracy
4. Build a persuasive artifact to show FCA why AI-accessibility matters
5. Establish a feedback loop to iterate toward a production system

Success is not perfection—it's learning what works and what needs to change.
