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

### User Flow

```
1. Open demo app
2. Acknowledge consent (beta status, interaction recording)
3. Enter product description (paste text or upload .md file)
4. Enter compliance question
5. Click "Analyze Compliance"
6. View results:
   - Top ~20 retrieved FCA Handbook rules
   - LLM reasoning explaining rule applicability
7. Approval gate:
   - Review analysis
   - Choose: Approve or Request Clarification
   - Optionally add comments on analysis quality
   - Submit
8. Interaction recorded to audit trail for process improvement
```

All interactions (product, question, rules, reasoning, approval, feedback) are logged per **Usage Records** (FirstImplementation.md) for later analysis.

## Feature Breakdown

### Input Section

**Product Description** (textarea):
- Accept markdown-formatted product details
- File upload option to attach .md files
- Example populated for quick testing

**Compliance Question** (textarea):
- Free-form compliance question
- Example: "Which parts of the FCA Handbook apply to this product?"

**Submit**: "Analyze Compliance" button

---

### Results Section

**Retrieved Rules** (top ~20 matching FCA Handbook rules):
- Rule ID and header
- Brief excerpt of rule text

**LLM Reasoning**:
- Natural language analysis of which rules apply and why
- Interpretation of rule applicability to the product

---

### Approval Gate

**Human Review**:
- Question: "Do these results correctly identify the applicable FCA rules for this product?"
- **Approve** button (green)
- **Request Clarification** button (yellow)
- Optional comments field (for feedback on analysis quality)

**Data Captured** (stored in interactions.json per Usage Records):
- Approval decision (approved / rejected)
- Approver feedback/comments
- Timestamp
- Full interaction context (product, question, rules retrieved, reasoning)

---

### Consent & Transparency

**Consent section at top**:
- User acknowledges this is a beta/non-functional demonstration
- User agrees interaction data may be recorded for system improvement
- No user data will be stored long-term

**Design principle**: Clear, honest labeling that this is a working prototype, not authoritative guidance

---

## Approval Gate & Feedback

The approval gate collects implicit and explicit feedback:

1. **Decision capture** — Approve or request clarification signals system effectiveness
2. **Optional comments** — Approver feedback on analysis quality, rule accuracy, reasoning clarity
3. **Full interaction recorded** — Per **Usage Records** (FirstImplementation.md), every interaction is logged with product, question, rules retrieved, reasoning, and approval outcome
4. **Later analysis** — Patterns in approver decisions and comments are analyzed separately (via Claude) to identify process improvements and ranking refinements

This feedback-first approach avoids survey fatigue while capturing honest, contextual feedback about system performance.

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

**See FirstImplementation.md § Usage Records for complete audit trail and data handling.**

### UI-Level
- Product descriptions and questions: Ephemeral (in-memory during workflow only)
- Approval decisions and comments: Captured for interaction log

### Audit Trail (Backend)
- All interactions logged to `interactions.json` per Usage Records section
- Includes: product, question, rules retrieved, LLM reasoning, approval decision, comments, timestamp
- Used for analysis and process refinement; no external sharing

### Privacy
- No user data stored by default
- Approval comments analyzed in aggregate only (patterns, not individuals)
- FCA Handbook data is public, redistributable

## Success Metrics: What We'll Learn

From interaction logs and approver feedback:
1. **Semantic search quality** — Are retrieved rules relevant? (signals: approval rate, comments on rule accuracy)
2. **Reasoning clarity** — Is LLM analysis understandable and well-reasoned? (signals: approval decisions, clarification requests)
3. **Completeness** — Are all relevant rules retrieved, or are gaps identified? (signals: "missing rule X" in comments)
4. **False positives** — Are irrelevant rules ranked too high? (signals: approver rejection, comments on noise)
5. **Performance** — How long does analysis take? (target: <30s)
6. **Usability** — Do users understand the interface and results? (signals: usage patterns, comment tone)
7. **Edge cases** — What product types or question phrasings cause issues? (signals: multiple rejections for similar inputs)

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
