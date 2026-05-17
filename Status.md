# Code-Ready Status

Below are outstanding issues, which need design decisions before coding. 

Some will be done as feedback emerges. Others need to be decided by those forking the repo for their own implementation.

Assessed by Claude Opus against GeneralEnquirySpecs.md, StructuredSearch.md, and PythonImplementation.md.

## FCA_Handbook_Text.JSON - Starting Point

FCA_Handbook_Text.JSON is the starting point for the Harness. 

This will be used to create FCA_Handbook_Text_And_Embeddings.

A formal copy has been requested from FCA. 

(A pirate copy has been contrived but is not included in this repo, because of potential unreliability. Here is how to create your own pirate copy: Start with the PRIN bits. Click the link of PRIN 1. Inspect the html. Find the part where the Angular is invoked. Give that to Claude, to create a program that extracts the text using Selenium, and moves on to the next page. Rinse and repeat. Do an eyeball test to see if the right text has been extracted. It probably won't have been, so refine everything and rerun. Repeat until correct. Now move on to the SYSC bits. See if the PRIN method works, if not adapt it. Daisy-chain this until you've done PRIN, COCON, COND... Do a lot of checks and tests. Make sure you build repeatable processes, you'll probably have to adapt and rerun for each new edition.)

## Critical Stubs (Must Complete Before Production)

### 1. GlossaryLookupAction._lookup_glossary()
- Location: PythonImplementation.md line 590
- Status: STUBBED (returns None)
- Specification: GeneralEnquirySpecs.md § glossary_lookup
- What it must do:
  - Implement hierarchical matching: exact term → prefix match → fuzzy semantic (0.7 threshold)
  - Query FCA Glossary records from FCA_Handbook_Text_And_Embeddings
  - Return None if term not found (graceful fallback; semantic_search proceeds with original term)
- Blocker for: Terminology mapping; search quality degrades without this

### 2. ApprovalGateAction human interaction
- Location: PythonImplementation.md line 1087
- Status: STUBBED (hardcoded approval)
- Specification: GeneralEnquirySpecs.md § approval_gate
- What it must do:
  - Integrate with your approval system (web UI, CLI, async approval service)
  - Prompt approver (compliance officer role) to review analysis
  - Validate approver email/identity
  - Handle timeout (spec defines 48h timeout)
  - Return ApprovalDecision with timestamp, approver identity, actions carried out
- Design decision needed: Approval flow (synchronous UI vs. async service)
- Blocker for: Human-in-the-loop; currently hardcoded to auto-approve

### 3. Audit logging storage backend
- Location: PythonImplementation.md line ~750 (audit logging pattern)
- Status: PATTERN DEFINED, STORAGE NOT SPECIFIED
- Specification: GeneralEnquirySpecs.md § Validation (logging mentioned but not detailed)
- What it must do:
  - Choose storage mechanism: local file, database, event stream
  - Implement persistence for interaction logs (question, features, retrieved rules, analysis, approval)
  - Define rotation/archival policy for audit trail
  - Ensure audit trail is immutable and timestamped (ISO 8601)
- Design decision needed: Storage backend (file vs. DB vs. cloud event stream)
- Blocker for: Audit trail completeness; currently logs to stdout only

## Important Gaps (Functionality Incomplete)

### 4. Error recovery logic
- Status: PARTIALLY IMPLEMENTED
- Issues:
  - Error messages defined but recovery strategies not specified
  - No retry logic for transient API failures (rate limiting, timeouts)
  - No fallback behavior for "no results found" scenario
  - Citation validation catches mismatches but doesn't prevent them at source
- Specification gaps: Error handling section of GeneralEnquirySpecs.md needs recovery strategies
- Impact: Production resilience; currently fails hard on errors

### 5. Configuration migration to external weights.yaml
- Location: PythonImplementation.md line ~700
- Status: HARDCODED (by design for MVP)
- Specification: StructuredSearch.md § Hardcoded Weights (Future Improvement)
- What it must do:
  - Load regulatory weighting factors from external weights.yaml
  - Validate weights schema before harness startup
  - Enable non-technical tuning without code changes
  - Optional: Dynamic weight switching per-query without restart
- Current state: Weights are hardcoded in HandbookIndex.__init__(); git history documents changes
- Blocker for: Compliance officer tuning; currently requires code change

### 6. Embedding model portability
- Status: DETECTED AT RUNTIME, NO MIGRATION PATH
- Issues:
  - Code assumes two models (Voyage 3-large, OpenAI text-embedding-3-large)
  - Adding a third model requires code changes
  - Switching models requires re-embedding entire handbook (not specified)
  - No versioning for embedding model in FCA_Handbook_Text_And_Embeddings.json
- Specification gaps: EmbeddingModel.md § Current Choice needs future migration guidance
- Impact: Model switching complexity; locked into chosen model currently

## Additional Unspecified (Lower Priority)

### 7. Authentication & Authorization
- Status: ROLE-BASED ACCESS SKETCHED, NOT IMPLEMENTED
- What's missing:
  - Approver role validation against whitelist
  - Email domain checking (compliance_officer@company.com)
  - RBAC for different user types
- Priority: Medium (needed if multi-user approval)

### 8. Performance optimization
- Status: ACCEPTABLE FOR MVP, NOT OPTIMIZED
- Known limitations:
  - Handbook embeddings loaded entirely into memory (OK for 10k rules; breaks at 100k+)
  - No result caching for repeated queries
  - No latency SLA enforced (UIStrategy.md mentions <30s target)
- Priority: Low (address post-launch if needed)

### 9. Data validation at output boundaries
- Status: INPUT VALIDATED, OUTPUT WEAK
- Missing:
  - Schema validation for node outputs (_validate_node_output stub at line ~1150)
  - Type checking for TerminologyMapped, RankedRules, ComplianceAnalysis
- Priority: Medium (catches bugs early)

### 10. UI implementation (demo)
- Status: NOT STARTED
- Specification: UIStrategy.md (design outline only)
- Missing: Complete Streamlit app implementation
- Priority: Low (nice-to-have for demo; harness works headless)

### 11. Testing & fixtures
- Status: TEST PATTERNS PROVIDED, FIXTURES MISSING
- Missing:
  - Sample markdown entity descriptions
  - Mock API responses (embedding, Claude)
  - Integration test fixtures
  - End-to-end test against template data
- Priority: High (needed before production validation)

---

## Summary Table

| Item | Type | Blocker | Priority |
|------|------|---------|----------|
| glossary_lookup() | Critical Stub | YES | CRITICAL |
| approval_gate UI | Critical Stub | YES | CRITICAL |
| Audit logging backend | Critical Stub | YES | CRITICAL |
| Error recovery | Gap | NO | HIGH |
| weights.yaml config | Gap | NO | HIGH |
| Testing fixtures | Gap | NO | HIGH |
| Embedding portability | Gap | NO | MEDIUM |
| Auth/RBAC | Gap | NO | MEDIUM |
| Output validation | Gap | NO | MEDIUM |
| Performance optimization | Gap | NO | LOW |
| UI implementation | Gap | NO | LOW |

---

## Path to Production

Minimum viable (3 weeks):
- Week 1: Implement critical stubs (glossary_lookup, approval_gate, audit logging)
- Week 2: Error recovery, weights.yaml migration, test fixtures
- Week 3: Integration testing, edge cases, validation against real data

Full production (6–8 weeks):
- Add items above
- Performance optimization
- Auth/RBAC implementation
- UI implementation
- Load testing, security audit

---

## Notes

- All gaps identified by Claude Opus (May 2026) against commit e704969
- Code is functionally complete but requires deployment-specific implementation
- No architectural blockers; all gaps are straightforward implementation work
- Specifications are sufficient for code generation (GeneralEnquirySpecs.md + StructuredSearch.md are authoritative)
