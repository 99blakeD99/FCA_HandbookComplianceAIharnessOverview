# Python Implementation

This document provides the patterns, interfaces, and concrete implementations needed to build the Compliance Agent Harness in Python. It implements the actions and orchestration patterns specified in GeneralEnquirySpecs.md.

**Cross-reference note:** Each action's input/output contract is defined in GeneralEnquirySpecs.md § Action Specifications. This document covers the implementation, not the contract.

---

## Table of Contents

1. [1. Orientation](#1-orientation)
   - [1.1 Architecture Overview](#11-architecture-overview)
   - [1.2 Data Flow Example](#12-data-flow-example)
   - [1.3 Deployment Models](#13-deployment-models)
2. [2. Foundations](#2-foundations)
   - [2.1 Exception Hierarchy](#21-exception-hierarchy)
   - [2.2 Action Base Class](#22-action-base-class)
   - [2.3 Action Registry](#23-action-registry)
   - [2.4 Claude Tool Schemas](#24-claude-tool-schemas)
   - [2.5 Common Patterns](#25-common-patterns)
3. [3. Action Implementations](#3-action-implementations)
   - [3.1 ParseMarkdownAction](#31-parsemarkdownaction)
   - [3.2 GlossaryLookupAction](#32-glossarylookupaction)
   - [3.3 SemanticSearchAction](#33-semanticsearchaction)
   - [3.4 ClaudeReasoningAction](#34-claudereasoningaction)
   - [3.5 ApprovalGateAction](#35-approvalgateaction)
4. [4. Harness Orchestration](#4-harness-orchestration)
   - [4.1 ComplianceHarness Class](#41-complianceharness-class)
   - [4.2 Workflow Execution](#42-workflow-execution)
   - [4.3 Audit Logging](#43-audit-logging)
   - [4.4 Tool Integration Entry Point](#44-tool-integration-entry-point)
5. [5. Testing](#5-testing)
6. [6. Implementation Checklist](#6-implementation-checklist)

---

## 1. Orientation

### 1.1 Architecture Overview

The Python harness has three layers:

1. **Orchestration Layer**: Loads YAML workflows, manages node execution, passes data between nodes
2. **Action Layer**: Implements each action type (parse_markdown, glossary_lookup, semantic_search, claude_reasoning, approval_gate) with input validation and error handling
3. **Data Layer**: Manages FCA Handbook embeddings (in-memory), configuration files (weights.yaml [schema in StructuredSearch.md], prompts), and audit logging

### 1.2 Data Flow Example

For a complete general_enquiry workflow:

```
INPUT:
  entity_description: "# AdviceBot..."
  question: "Which COBS rules apply to algorithmic advice?"

NODE 1 (parse_markdown):
  INPUT: entity_description
  OUTPUT: EntityFeatures
  STORED: context['extract_features']

NODE 2 (glossary_lookup):
  INPUT: {entity_features: context['extract_features'], question}
  OUTPUT: TerminologyMapped
  STORED: context['check_terminology']

NODE 3 (semantic_search):
  INPUT: {entity_features: context['extract_features'], question_terms: context['check_terminology'], question}
  OUTPUT: RankedRules (list of rules with similarity + weights)
  STORED: context['retrieve_rules']

NODE 4 (claude_reasoning):
  INPUT: {entity_features: context['extract_features'], rules: context['retrieve_rules']}
  OUTPUT: ComplianceAnalysis (answer + citations + reasoning_log)
  STORED: context['analyze_compliance']

NODE 5 (approval_gate):
  INPUT: {analysis: context['analyze_compliance']}
  OUTPUT: ApprovalDecision (approved + approver + timestamp + audit trail)
  STORED: context['human_review']

FINAL OUTPUT: context['human_review'] (ApprovalDecision)
```

### 1.3 Deployment Models

The Harness can be deployed in multiple ways. Code patterns differ by model; this document provides examples for **Option 1 (Direct Integration)** but notes how to adapt for others:

| Model | Description | Code Pattern | Tool Definition |
|-------|-------------|--------------|-----------------|
| **Option 1: Direct Integration** | Python code calls Claude API directly with Harness as tool | SDK call with tool definition embedded | Hardcoded in Python or config file |
| **Option 2: Service Deployment** | Harness exposed as REST/GraphQL API; external service invokes it | FastAPI/Flask handler + tool definition in service | Service configuration (YAML/JSON) |
| **Option 3: Tool Marketplace** | Registered in platform (e.g., Anthropic marketplace); platform manages invocation | Platform-specific integration (may not require custom code) | Platform UI / database |

**Important:** Before generating code for a specific deployment model, clarify:
- Is the Harness being called directly from Python, or exposed as a service?
- Where should the tool definition live (hardcoded, config file, platform)?
- What external systems need to integrate (Claude API, database, etc.)?

The code examples below assume **Option 1** (direct Python integration) but are adaptable.

---

## 2. Foundations

### 2.1 Exception Hierarchy

All actions raise domain-specific exceptions for clear error handling:

```python
class ActionError(Exception):
    """Base exception for all action errors."""
    pass

class ValidationError(ActionError):
    """Input validation failed."""
    pass

class DataSourceUnavailableError(ActionError):
    """Required data source (embeddings, config, etc.) is unavailable."""
    pass

class CitationValidationError(ActionError):
    """Citation text does not match source."""
    pass

class ApprovalError(ActionError):
    """Approval decision could not be recorded."""
    pass

class TimeoutError(ActionError):
    """Operation exceeded time limit (e.g., approval pending >48h)."""
    pass
```

### 2.2 Action Base Class

All actions inherit from a common interface:

```python
from abc import ABC, abstractmethod
from typing import Any, Dict

class Action(ABC):
    """Abstract base class for all harness actions."""
    
    def __init__(self, harness: 'ComplianceHarness' = None):
        """
        Initialize action with optional harness reference for data access.
        
        Args:
            harness: ComplianceHarness instance (provides handbook_index, workflows, etc.)
        """
        self.harness = harness
    
    @abstractmethod
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Any:
        """
        Execute the action.
        
        Args:
            node_input: Input prepared from YAML node["input"]. May contain:
                        - User inputs (entity_description, question)
                        - Outputs from earlier nodes (EntityFeatures, RankedRules, etc.)
            config: Node configuration from YAML node["config"]. Contains Param_* keys.
        
        Returns:
            Output object matching the action's output schema (see GeneralEnquirySpecs.md)
        
        Raises:
            Subclasses of ActionError with actionable messages
        """
        pass
    
    def _validate_config(self, config: Dict, required_params: list):
        """Helper: Ensure all required Param_* keys are present."""
        for param in required_params:
            if param not in config:
                raise ValidationError(f"Required config parameter missing: {param}")
```

### 2.3 Action Registry

Register all actions here. The harness uses this registry to look up action implementations by name from the YAML workflow:

```python
ACTION_REGISTRY = {
    "validate_scope": ValidateScopeAction,
    "parse_markdown": ParseMarkdownAction,
    "glossary_lookup": GlossaryLookupAction,
    "embed_text": EmbedTextAction,
    "semantic_search": SemanticSearchAction,
    "claude_reasoning": ClaudeReasoningAction,
    "approval_gate": ApprovalGateAction,
}
```

### 2.4 Claude Tool Schemas

These tools enforce structured output from Claude for citations and reasoning logs. Define once as module-level constants (used by ClaudeReasoningAction):

```python
# Module-level tool definitions (referenced by ClaudeReasoningAction)
CITATION_FORMATTER_TOOL = {
    "name": "citation_formatter",
    "description": "Format a citation to an FCA Handbook rule with rule_id, verbatim text, and context showing why it applies.",
    "input_schema": {
        "type": "object",
        "properties": {
            "rule_id": {
                "type": "string",
                "description": "FCA rule identifier from RankedRules (e.g., 'COBS 2.1.1R'). Must match a rule_id from the retrieved rules."
            },
            "cited_text": {
                "type": "string",
                "description": "Exact verbatim text from the FCA Handbook rule. Must appear word-for-word in the rule's text field."
            },
            "context": {
                "type": "string",
                "description": "Sentence or paragraph explaining why this rule applies to the product (why the product triggers this rule)."
            },
            "binding_level": {
                "type": "string",
                "enum": ["R", "G", "E", "D"],
                "description": "Binding level: R=Rules, G=Guidance, E=Evidential, D=Deleted. Extracted from rule_id suffix."
            }
        },
        "required": ["rule_id", "cited_text", "context", "binding_level"]
    }
}

AUDIT_LOGGER_TOOL = {
    "name": "audit_logger",
    "description": "Log structured reasoning and confidence metrics for compliance analysis.",
    "input_schema": {
        "type": "object",
        "properties": {
            "reasoning_chain": {
                "type": "string",
                "description": "Step-by-step logic explaining which rules apply to the product and why (e.g., 'Rule A applies because feature X matches criterion Y; Rule B does not apply because...')"
            },
            "gaps_identified": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Array of edge cases, uncertainties, or areas where the spec is ambiguous"
            },
            "confidence_score": {
                "type": "number",
                "minimum": 0,
                "maximum": 1,
                "description": "Model's confidence in the compliance analysis (0-1). Lower if gaps or ambiguities exist."
            }
        },
        "required": ["reasoning_chain", "gaps_identified", "confidence_score"]
    }
}
```

### 2.5 Common Patterns

These patterns are repeated across action implementations. Reference these sections to avoid inline repetition.

#### Input Validation

All actions validate their inputs at the start of `execute()`:

```python
def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
    # Extract inputs
    x = node_input.get('x')
    
    # Validate shape and type
    if not x or not isinstance(x, str):
        raise ValidationError("x must be a non-empty string")
```

Pattern: Use `isinstance()` checks and raise `ValidationError` with a clear message naming the field and expected type.

#### Config Parameter Extraction

All actions extract configuration from the YAML `config` dict. Parameters are prefixed with `Param_`:

```python
# From YAML: config: { Param_top_k: 20, Param_source_version: "2026-Q1" }
top_k = config.get('Param_top_k', 20)  # Use defaults if specified
source_version = config.get('Param_source_version')

# Validate required params using _validate_config helper
self._validate_config(config, ['Param_top_k'])
```

Pattern: Extract with `.get()`, supply defaults, validate required params with the base class helper.

#### UTC Timestamp Emission

When emitting timestamps (used in ClaudeReasoningAction, ApprovalGateAction, and logging):

```python
from datetime import datetime, timezone

timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
# Result: "2026-05-16T14:32:45.123456+00:00Z" (ISO 8601 with explicit UTC marker)
```

#### External-Service Unavailability Errors

When a required service (embedding API, FCA Glossary, Claude API) fails, raise `DataSourceUnavailableError` with actionable guidance:

```python
raise DataSourceUnavailableError(
    f"FCA Handbook Glossary not available. Unable to perform terminology mapping.\n"
    f"Ensure FCA_Handbook_Text_And_Embeddings is loaded and accessible."
)
```

Pattern: (1) what failed, (2) actionable next step (file path, config, escalation contact, or doc reference).

#### UTC Timestamp with Timezone Import

All files that emit timestamps should include:

```python
from datetime import datetime, timezone
```

---

## 3. Action Implementations

### 3.1 ValidateScopeAction

#### Contract

**Input**: `question` (string)
**Output**: `ScopeValidation` (dict with valid, reason)
**Config**: `Param_require_explicit_scope` (bool, default true)
**Raises**: `ValidationError` if question invalid

#### Implementation

```python
class ValidateScopeAction(Action):
    """Validate question scope before workflow execution."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Validate that question concerns FCA Handbook compliance specifically.
        
        Args:
            node_input: {'question': str}
            config: {'Param_require_explicit_scope': bool}
        
        Returns:
            {'valid': bool, 'reason': str}
        """
        question = node_input.get('question', '')
        require_explicit = config.get('Param_require_explicit_scope', True)
        
        if not question or not isinstance(question, str):
            raise ValidationError("question must be a non-empty string")
        
        if len(question) < 5 or len(question) > 1000:
            raise ValidationError("question must be 5-1000 characters")
        
        question_lower = question.lower()
        
        # Check if scope is clear (explicit FCA Handbook mention)
        if 'fca handbook' in question_lower or 'fca requirements' in question_lower:
            return {
                'valid': True,
                'reason': 'Scope is clear and valid (FCA Handbook explicitly mentioned)'
            }
        
        # Check if scope is out-of-bounds (clearly not regulatory)
        out_of_scope_keywords = [
            'best investment strategy', 'how should i invest', 'market outlook',
            'stock recommendation', 'which stock to buy', 'crypto investment',
            'personal financial advice'
        ]
        for keyword in out_of_scope_keywords:
            if keyword in question_lower:
                return {
                    'valid': False,
                    'reason': f'Question out of scope: this harness answers FCA Handbook compliance questions, not general financial advice'
                }
        
        # If ambiguous, apply Param_require_explicit_scope logic
        if require_explicit:
            return {
                'valid': False,
                'reason': 'Scope ambiguous. Please clarify: "Do you mean compliance with the FCA Handbook?"'
            }
        else:
            return {
                'valid': True,
                'reason': 'Scope ambiguous but proceeding (Param_require_explicit_scope=false)'
            }
```

### 3.2 ParseMarkdownAction

#### Contract

**Input**: `entity_description` (string: markdown-formatted product description)
**Output**: `EntityFeatures` (dict with entity_name, entity_type, features, use_cases, client_interaction, data_handled, decision_authority)
**Config**: None required
**Raises**: `ValidationError` on invalid markdown or missing required fields

#### Implementation

Extracts structured product information from markdown.

```python
import re
from typing import Dict, List, Any

class ParseMarkdownAction(Action):
    """Extract and structure entity information from markdown-formatted entity description."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Parse markdown product description into structured EntityFeatures.
        
        Args:
            node_input: {
                'entity_description': str (markdown-formatted product description)
            }
            config: {} (no parameters for this action)
        
        Returns:
            {
                'entity_name': str,
                'entity_type': str,
                'features': list of str,
                'use_cases': list of str,
                'client_interaction': str,
                'data_handled': list of str,
                'decision_authority': str
            }
        
        Raises:
            ValidationError: If required fields missing or invalid
        """
        entity_description = node_input.get('entity_description', '')
        
        if not entity_description or not isinstance(entity_description, str):
            raise ValidationError("entity_description must be a non-empty string")
        
        # Extract product name from first heading (h1)
        entity_name = self._extract_h1(entity_description)
        if not entity_name:
            raise ValidationError("Missing product name (h1 heading required)")
        
        # Search for standard sections
        features = self._extract_list_items(entity_description, r'#+\s*Features?')
        use_cases = self._extract_list_items(entity_description, r'#+\s*Use Cases?')
        client_interaction = self._extract_section_text(entity_description, r'#+\s*Client Interaction')
        data_handled = self._extract_list_items(entity_description, r'#+\s*Data Handling?')
        decision_authority = self._extract_section_text(entity_description, r'#+\s*.*Decision.*Authority')
        
        # Validate required fields
        if not features:
            raise ValidationError("Missing required field: features")
        if not use_cases:
            raise ValidationError("Missing required field: use_cases")
        if not decision_authority:
            raise ValidationError("Missing required field: decision_authority")
        
        # Normalize decision_authority to allowed values
        authority_normalized = self._normalize_authority(decision_authority)
        if authority_normalized not in ["algorithm", "advisor", "client", "hybrid"]:
            raise ValidationError(
                f"Invalid value for decision_authority: {decision_authority} "
                "(must be one of: algorithm, advisor, client, hybrid)"
            )
        
        # Validate field constraints
        if len(entity_name) > 200:
            raise ValidationError(f"entity_name exceeds 200 chars: {len(entity_name)}")
        if not features or len(features) > 50:
            raise ValidationError(f"features must have 1-50 items, got {len(features)}")
        if not use_cases or len(use_cases) > 20:
            raise ValidationError(f"use_cases must have 1-20 items, got {len(use_cases)}")
        if any(len(f) > 500 for f in features):
            raise ValidationError("One or more features exceed 500 chars")
        
        # Determine entity_type from section heading or infer from content
        entity_type = self._extract_entity_type(entity_description)
        if not entity_type:
            entity_type = "unknown"
        
        return {
            'entity_name': entity_name.strip(),
            'entity_type': entity_type,
            'features': [f.strip() for f in features],
            'use_cases': [u.strip() for u in use_cases],
            'client_interaction': client_interaction.strip() if client_interaction else "unspecified",
            'data_handled': [d.strip() for d in data_handled] if data_handled else [],
            'decision_authority': authority_normalized
        }
    
    def _extract_h1(self, text: str) -> str:
        """Extract text from first h1 (# heading)."""
        match = re.search(r'^#\s+(.+?)$', text, re.MULTILINE)
        return match.group(1) if match else None
    
    def _extract_list_items(self, text: str, section_pattern: str) -> List[str]:
        """Extract bullet/numbered list items under a section heading."""
        section_match = re.search(section_pattern, text, re.IGNORECASE | re.MULTILINE)
        if not section_match:
            return []
        
        start_pos = section_match.end()
        next_heading = re.search(r'^#+\s+', text[start_pos:], re.MULTILINE)
        end_pos = start_pos + next_heading.start() if next_heading else len(text)
        
        section_text = text[start_pos:end_pos]
        items = re.findall(r'^\s*[-*•]\s+(.+?)$', section_text, re.MULTILINE)
        return items
    
    def _extract_section_text(self, text: str, section_pattern: str) -> str:
        """Extract plain text content under a section heading."""
        section_match = re.search(section_pattern, text, re.IGNORECASE | re.MULTILINE)
        if not section_match:
            return ""
        
        start_pos = section_match.end()
        next_heading = re.search(r'^#+\s+', text[start_pos:], re.MULTILINE)
        end_pos = start_pos + next_heading.start() if next_heading else len(text)
        
        section_text = text[start_pos:end_pos].strip()
        section_text = re.sub(r'^\s*[-*•]\s+', '', section_text)
        return section_text.split('\n')[0]
    
    def _extract_entity_type(self, text: str) -> str:
        """Infer product type from content."""
        keywords = {
            'robo-advisor': ['robo', 'automated advice', 'algorithmic advice'],
            'advisory platform': ['advisory', 'platform'],
            'trading system': ['trading', 'execution'],
            'portfolio management': ['portfolio', 'management']
        }
        text_lower = text.lower()
        for ptype, keywords_list in keywords.items():
            if any(kw in text_lower for kw in keywords_list):
                return ptype
        return None
    
    def _normalize_authority(self, text: str) -> str:
        """Normalize decision authority text to allowed values."""
        text_lower = text.lower().strip()
        if 'algorithm' in text_lower:
            return 'algorithm'
        elif 'advisor' in text_lower:
            return 'advisor'
        elif 'client' in text_lower:
            return 'client'
        elif 'hybrid' in text_lower or 'combination' in text_lower:
            return 'hybrid'
        return text_lower
```

### 3.3 GlossaryLookupAction

#### Contract

**Input**: `entity_features` (EntityFeatures), `question` (string)
**Output**: `TerminologyMapped` (dict with mapped_terms, unmapped_terms, all_search_terms)
**Config**: `Param_glossary_piece` (default: "Glossary")
**Raises**: `DataSourceUnavailableError` if FCA Glossary unavailable

#### Implementation

Maps user question terminology to FCA Glossary canonical terms.

```python
import re
from typing import Dict, List, Any

class GlossaryLookupAction(Action):
    """Map question terminology to FCA Glossary canonical terms."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Map user question and entity features to FCA Glossary canonical terms.
        
        Args:
            node_input: {
                'entity_features': EntityFeatures,
                'question': str
            }
            config: {
                'Param_glossary_piece': str (default "Glossary")
            }
        
        Returns:
            {
                'mapped_terms': [
                    {'user_term': str, 'canonical_term': str, 'glossary_entry': str},
                    ...
                ],
                'unmapped_terms': [str, ...],
                'all_search_terms': [str, ...]
            }
        """
        entity_features = node_input.get('entity_features')
        question = node_input.get('question', '')
        glossary_piece = config.get('Param_glossary_piece', 'Glossary')
        
        if not question or not isinstance(question, str):
            raise ValidationError("question must be a non-empty string")
        if not entity_features or not isinstance(entity_features, dict):
            raise ValidationError("entity_features must be a valid EntityFeatures object")
        
        # Extract key terms from question and entity_features
        question_terms = self._extract_terms(question)
        entity_terms = self._extract_entity_terms(entity_features)
        all_terms = list(set(question_terms + entity_terms))
        
        # Look up each term in FCA Glossary
        mapped_terms = []
        unmapped_terms = []
        
        for term in all_terms:
            glossary_match = self._lookup_glossary(term, glossary_piece)
            if glossary_match:
                mapped_terms.append({
                    'user_term': term,
                    'canonical_term': glossary_match['canonical_term'],
                    'glossary_entry': glossary_match['definition']
                })
            else:
                unmapped_terms.append(term)
        
        # Compile all search terms: mapped canonical terms + unmapped user terms
        all_search_terms = [m['canonical_term'] for m in mapped_terms] + unmapped_terms
        
        return {
            'mapped_terms': mapped_terms,
            'unmapped_terms': unmapped_terms,
            'all_search_terms': all_search_terms
        }
    
    def _extract_terms(self, text: str) -> List[str]:
        """Extract capitalized words and multi-word phrases from text."""
        terms = re.findall(r'\b[A-Z][a-z]+(?:\s+[A-Z][a-z]+)*\b', text)
        lowercase_terms = re.findall(r'\b(?:cash|deposit|advance payment|cryptocurrency|virtual assets?)\b', text, re.IGNORECASE)
        return [t for t in terms + lowercase_terms if len(t) > 2]
    
    def _extract_entity_terms(self, entity_features: Dict) -> List[str]:
        """Extract key terms from entity_features structure."""
        terms = []
        if entity_features.get('entity_type'):
            terms.extend(entity_features['entity_type'].split())
        if entity_features.get('features'):
            for feature in entity_features['features']:
                terms.extend(feature.split())
        if entity_features.get('use_cases'):
            for use_case in entity_features['use_cases']:
                terms.extend(use_case.split())
        return [t.strip() for t in terms if len(t.strip()) > 2]
    
    def _lookup_glossary(self, term: str, glossary_piece: str) -> Dict[str, str] or None:
        """
        Look up term in FCA Glossary.
        
        Real FCA examples:
        - term "money handling" → canonical_term "cash" (FCA Glossary: "includes money in any form")
        - term "deposit" → canonical_term "advance payment" (FCA Glossary: "includes any deposit but...")
        """
        # Stub: actual implementation would query glossary records from FCA_Handbook_Text_And_Embeddings
        # For now, return None (term not found); implementer fills in actual glossary lookup logic
        return None
```

### 3.4 EmbedTextAction

#### Contract

**Input**: `question` (string)
**Output**: `QuestionEmbedding` (vector)
**Config**: None (uses embedding model from harness.data_sources.fca_handbook.model)
**Raises**: `DataSourceUnavailableError` if embedding API unavailable

#### Implementation

```python
class EmbedTextAction(Action):
    """Embed question using same model as FCA Handbook embeddings."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> List[float]:
        """
        Embed the user question with the same embedding model as FCA_Handbook_Text_And_Embeddings.
        
        Args:
            node_input: {'question': str}
            config: {} (embedding model comes from harness.handbook_index.embedding_model)
        
        Returns:
            List[float]: Embedding vector matching handbook embeddings dimensionality
        """
        question = node_input.get('question', '')
        
        if not question or not isinstance(question, str):
            raise ValidationError("question must be a non-empty string")
        
        if len(question) > 1000:
            raise ValidationError("question must be max 1000 characters")
        
        # Call _embed_query (same as SemanticSearchAction uses internally)
        embedding = self._embed_query(question)
        
        return embedding
    
    def _embed_query(self, text: str) -> List[float]:
        """
        Embed query text using the same model as handbook embeddings.
        
        Detects embedding model from handbook_index.embedding_model and calls
        appropriate API (Voyage AI or OpenAI) with credentials from environment.
        """
        import os
        import requests
        
        if not text or not isinstance(text, str):
            raise ValidationError("text must be a non-empty string")
        
        embedding_model = self.harness.handbook_index.embedding_model
        
        try:
            if embedding_model == 'voyage-3-large':
                # Call Voyage AI API
                api_key = os.environ.get('VOYAGE_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. VOYAGE_API_KEY environment variable not set.\n"
                        "Set VOYAGE_API_KEY to enable Voyage AI embeddings."
                    )
                
                response = requests.post(
                    'https://api.voyageai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'voyage-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            elif embedding_model == 'text-embedding-3-large':
                # Call OpenAI API
                api_key = os.environ.get('OPENAI_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. OPENAI_API_KEY environment variable not set.\n"
                        "Set OPENAI_API_KEY to enable OpenAI embeddings."
                    )
                
                response = requests.post(
                    'https://api.openai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'text-embedding-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            else:
                raise DataSourceUnavailableError(
                    f"Unknown embedding model: {embedding_model}\n"
                    f"Expected 'voyage-3-large' or 'text-embedding-3-large'"
                )
        
        except requests.exceptions.RequestException as e:
            raise DataSourceUnavailableError(
                f"Embedding service unavailable. Failed to call {embedding_model} API: {str(e)}\n"
                f"Check API credentials and network connectivity."
            )
        except (KeyError, ValueError) as e:
            raise DataSourceUnavailableError(
                f"Embedding API returned invalid response: {str(e)}\n"
                f"Ensure API credentials are valid and model {embedding_model} is available."
            )
```

### 3.5 SemanticSearchAction

#### Contract

**Input**: `entity_features` (EntityFeatures), `question_terms` (TerminologyMapped), `question_embedding` (vector)
**Output**: `RankedRules` (array of rule objects with similarity + weight factors)
**Config**: `Param_top_k` (int, default 20), `Param_source_version` (string, default "2026-Q1")
**Raises**: `DataSourceUnavailableError` if data source unavailable

#### HandbookIndex (Data Layer)

Load and index the FCA Handbook once at startup:

```python
import json
import numpy as np
from typing import Dict, List, Any

class HandbookIndex:
    """Load and index FCA Handbook embeddings for semantic search."""
    
    def __init__(self, handbook_path: str, weights_config_path: str = None):
        """
        Load handbook data and weighting configuration.
        
        Args:
            handbook_path: Path to FCA_Handbook_Text_And_Embeddings JSON file
            weights_config_path: Unused (kept for compatibility). Weights are hardcoded from StructuredSearch.md
        
        Raises:
            DataSourceUnavailableError: If JSON is missing 'fca_handbook' key, has no records, or records lack embeddings
        """
        # Load handbook data
        try:
            with open(handbook_path) as f:
                data = json.load(f)
        except FileNotFoundError:
            raise DataSourceUnavailableError(
                f"FCA Handbook JSON file not found at: {handbook_path}\n"
                f"See README.md § Getting Started (item 8) for expected file format."
            )
        except json.JSONDecodeError as e:
            raise DataSourceUnavailableError(
                f"FCA Handbook JSON is malformed: {e}\n"
                f"Ensure the file is valid JSON from FCA_Handbook_Text_And_Embeddings"
            )
        
        # Validate structure
        if 'fca_handbook' not in data:
            raise DataSourceUnavailableError(
                f"FCA Handbook JSON missing 'fca_handbook' key.\n"
                f"Expected structure: {{'fca_handbook': [{{'rule_id': '...', 'text': '...', 'voyage-3-large_embedding': [...]}}]}}\n"
                f"See FCA_Handbook_Template_PRIN.json for the correct format."
            )
        
        self.records = data['fca_handbook']
        
        if not self.records:
            raise DataSourceUnavailableError(
                f"FCA Handbook has no records (fca_handbook array is empty).\n"
                f"Ensure FCA_Handbook_Text_And_Embeddings contains at least one record."
            )
        
        # Detect and load embeddings: try common embedding key names (Voyage or OpenAI)
        self.embedding_key = self._detect_embedding_key(self.records[0])
        self.embeddings = np.array([record[self.embedding_key] for record in self.records])
        
        # Map embedding key to model name for query embedding
        self.embedding_model = self._get_model_name(self.embedding_key)
        
        # Validate embedding dimensions match expected model
        self._validate_embedding_dimensions()
        
        # Default regulatory weights (from StructuredSearch.md § Weighting Factors)
        self.weights = {
            'rule_type_weights': {
                'RULES': 1.0,
                'GUIDANCE': 0.8,
                'EVIDENTIAL': 0.6,
                'UNCLASSIFIED': 0.8,
                'DELETED': 0.1
            },
            'importance_multipliers': {
                'high': 1.3,
                'medium': 1.1,
                'low': 0.7
            },
            'piece_base_weights': {
                'Main Handbook': 1.2,
                'Glossary': 0.95,
                'Instruments': 1.0,
                'Forms': 0.85,
                'Technical Standards': 0.9,
                'Level3Materials': 0.85
            },
            'hierarchy_multipliers': {
                1: 1.2,
                2: 1.0,
                'default': 0.9
            }
        }
    
    def _detect_embedding_key(self, sample_record: Dict) -> str:
        """
        Detect which embedding key exists in the record.
        Tries: voyage-3-large_embedding (default), OpenAI_text-embedding-3-large_embedding, or 'embedding'.
        
        Raises:
            DataSourceUnavailableError: If no embedding key found (plain JSON without embeddings)
        """
        embedding_keys = [
            'voyage-3-large_embedding',
            'OpenAI_text-embedding-3-large_embedding',
            'embedding'
        ]
        
        for key in embedding_keys:
            if key in sample_record:
                return key
        
        available_keys = list(sample_record.keys())
        raise DataSourceUnavailableError(
            f"ERROR: FCA_Handbook_Text_And_Embeddings is missing vector embeddings.\n\n"
            f"The Harness requires each record to include a pre-computed embedding vector.\n"
            f"Your JSON has these keys: {available_keys}\n"
            f"But is missing one of: {embedding_keys}\n\n"
            f"SOLUTION: Follow these steps to add embeddings to your JSON:\n"
            f"1. Read EmbeddingModel.md § 'Action' for instructions on embedding your data\n"
            f"2. Use an LLM API (Voyage AI or OpenAI) to embed each record\n"
            f"3. Add the embedding vector to each record with the key: 'voyage-3-large_embedding'\n"
            f"4. Example record:\n"
            f"   {{\n"
            f"     'rule_id': 'COBS 2.1.1R',\n"
            f"     'text': '...',\n"
            f"     'voyage-3-large_embedding': [0.123, -0.456, ...]\n"
            f"   }}\n\n"
            f"See FCA_Handbook_Template_PRIN.json for a complete example with two embedding models."
        )
    
    def _get_model_name(self, embedding_key: str) -> str:
        """Map embedding key to model name for query embedding."""
        model_map = {
            'voyage-3-large_embedding': 'voyage-3-large',
            'OpenAI_text-embedding-3-large_embedding': 'text-embedding-3-large',
            'embedding': 'text-embedding-3-large'
        }
        return model_map.get(embedding_key, 'text-embedding-3-large')
    
    def _validate_embedding_dimensions(self):
        """
        Validate that embeddings have correct dimensionality for the detected model.
        Prevents silent errors if embeddings were generated with a different model.
        
        Raises:
            DataSourceUnavailableError: If embedding dimensions don't match expected model
        """
        expected_dims = {
            'voyage-3-large': 1024,
            'text-embedding-3-large': 3072
        }
        
        actual_dim = self.embeddings.shape[1] if self.embeddings.ndim == 2 else len(self.embeddings[0])
        expected_dim = expected_dims.get(self.embedding_model, None)
        
        if expected_dim and actual_dim != expected_dim:
            raise DataSourceUnavailableError(
                f"Embedding dimensionality mismatch.\n"
                f"Expected {expected_dim} dimensions for model '{self.embedding_model}'\n"
                f"but found {actual_dim} dimensions in key '{self.embedding_key}'.\n\n"
                f"Possible causes:\n"
                f"- Embeddings were generated with a different model than {self.embedding_model}\n"
                f"- The wrong embedding key was detected\n\n"
                f"Solution: Ensure all embeddings use {self.embedding_model} model "
                f"(see EmbeddingModel.md § 'Current Choice' for details)."
            )
    
    def search(self, query_embedding, top_k: int = 20) -> List[Dict[str, Any]]:
        """
        Find top-k most similar records using cosine similarity, then apply regulatory weighting.
        
        Args:
            query_embedding: 3072-dimensional embedding from text-embedding-3-large
            top_k: Number of results to return (default 20, range 1-100)
        
        Returns:
            List of dicts with: rule_id, text, base_similarity, weight_factors, final_score
        """
        query_embedding = np.array(query_embedding)
        
        # Vectorized cosine similarity
        query_norm = np.linalg.norm(query_embedding)
        embeddings_norms = np.linalg.norm(self.embeddings, axis=1)
        dot_products = np.dot(self.embeddings, query_embedding)
        similarities = dot_products / (embeddings_norms * query_norm)
        
        # Pre-filter: get top 50 candidates before weighting
        top_50_indices = np.argsort(similarities)[-50:][::-1]
        candidates = []
        for idx in top_50_indices:
            candidates.append({
                'index': int(idx),
                'record': self.records[idx],
                'base_similarity': float(similarities[idx])
            })
        
        # Apply regulatory weighting to candidates
        for candidate in candidates:
            candidate['weight_factors'] = self._compute_weights(candidate['record'])
            candidate['final_score'] = (
                candidate['base_similarity'] 
                * candidate['weight_factors']['rule_type_weight']
                * candidate['weight_factors']['hierarchy_multiplier']
                * candidate['weight_factors']['importance_multiplier']
                * candidate['weight_factors']['piece_weight']
            )
        
        # Sort by final_score and return top_k
        candidates.sort(key=lambda x: x['final_score'], reverse=True)
        return candidates[:top_k]
    
    def _compute_weights(self, record: Dict) -> Dict[str, float]:
        """
        Compute all weight factors for a record per StructuredSearch.md § Weighting Factors.
        
        Extracts rule type, hierarchy level, importance score, and piece authority
        to compute final weighting multipliers for regulatory ranking.
        """
        # Extract rule_type_weight from regulatory_content suffix (R/G/E/U/D)
        regulatory_content = record.get('regulatory_content', '')
        rule_type_map = {'R': 'RULES', 'G': 'GUIDANCE', 'E': 'EVIDENTIAL', 'D': 'DELETED', 'U': 'UNCLASSIFIED'}
        
        rule_type_key = 'UNCLASSIFIED'  # Default if not found
        for char in regulatory_content:
            if char in rule_type_map:
                rule_type_key = rule_type_map[char]
                break
        
        rule_type_weight = self.weights['rule_type_weights'].get(rule_type_key, 0.8)
        
        # Extract hierarchy_multiplier from record['level']
        level = record.get('level', 3)
        hierarchy_multiplier = self.weights['hierarchy_multipliers'].get(
            level,
            self.weights['hierarchy_multipliers'].get('default', 0.9)
        )
        
        # Extract importance_multiplier from record['regulatory_score'] (1-12 → high/medium/low bands)
        regulatory_score = record.get('regulatory_score', 4)
        if regulatory_score >= 7:
            importance_key = 'high'
        elif regulatory_score >= 4:
            importance_key = 'medium'
        else:
            importance_key = 'low'
        
        importance_multiplier = self.weights['importance_multipliers'].get(importance_key, 1.1)
        
        # Extract piece_weight from record['piece']
        piece = record.get('piece', 'Main Handbook')
        piece_weight = self.weights['piece_base_weights'].get(piece, 1.0)
        
        return {
            'rule_type_weight': rule_type_weight,
            'hierarchy_multiplier': hierarchy_multiplier,
            'importance_multiplier': importance_multiplier,
            'piece_weight': piece_weight
        }
```

#### SemanticSearchAction (Action Layer)

```python
from anthropic import Anthropic

class SemanticSearchAction(Action):
    """Query FCA Handbook via semantic search with regulatory weighting."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> List[Dict[str, Any]]:
        """
        Search FCA_Handbook_Text_And_Embeddings for rules matching question and entity features.
        Apply regulatory weighting to rank by binding authority and relevance.
        
        Uses TerminologyMapped from glossary_lookup to augment search with canonical terms.
        
        Args:
            node_input: {
                'entity_features': EntityFeatures,
                'question_terms': TerminologyMapped (from glossary_lookup),
                'question': str
            }
            config: {
                'Param_top_k': int (default 20, range 1-100)
            }
        
        Returns:
            List of RankedRule dicts (rule_id, text, base_similarity, weight_factors, final_score)
        """
        entity_features = node_input.get('entity_features', {})
        question_terms = node_input.get('question_terms', {})
        question = node_input.get('question', '')
        top_k = config.get('Param_top_k', 20)
        source_version = config.get('Param_source_version', '2026-Q1')
        
        if not isinstance(top_k, int) or top_k < 1 or top_k > 100:
            raise ValidationError("Param_top_k must be an integer between 1 and 100")
        
        # Extract search terms: original question + mapped canonical terms
        search_terms = question_terms.get('all_search_terms', [])
        enriched_question = question + ' ' + ' '.join(search_terms)
        
        # Embed the enriched question
        # Stub: call embedding API to get query_embedding
        query_embedding = self._embed_query(enriched_question)
        
        # Search and rank
        results = self.handbook_index.search(query_embedding, top_k)
        
        return [
            {
                'rule_id': r['record'].get('rule_id'),
                'text': r['record'].get('text'),
                'base_similarity': r['base_similarity'],
                'weight_factors': r['weight_factors'],
                'final_score': r['final_score'],
                'source_version': source_version
            }
            for r in results
        ]
    
    def _embed_query(self, text: str) -> List[float]:
        """
        Embed query text using the same model as handbook embeddings.
        
        Detects embedding model from handbook_index.embedding_model and calls
        appropriate API (Voyage AI or OpenAI) with credentials from environment.
        """
        import os
        import requests
        
        if not text or not isinstance(text, str):
            raise ValidationError("text must be a non-empty string")
        
        embedding_model = self.harness.handbook_index.embedding_model
        
        try:
            if embedding_model == 'voyage-3-large':
                # Call Voyage AI API
                api_key = os.environ.get('VOYAGE_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. VOYAGE_API_KEY environment variable not set.\n"
                        "Set VOYAGE_API_KEY to enable Voyage AI embeddings."
                    )
                
                response = requests.post(
                    'https://api.voyageai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'voyage-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            elif embedding_model == 'text-embedding-3-large':
                # Call OpenAI API
                api_key = os.environ.get('OPENAI_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. OPENAI_API_KEY environment variable not set.\n"
                        "Set OPENAI_API_KEY to enable OpenAI embeddings."
                    )
                
                response = requests.post(
                    'https://api.openai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'text-embedding-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            else:
                raise DataSourceUnavailableError(
                    f"Unknown embedding model: {embedding_model}\n"
                    f"Expected 'voyage-3-large' or 'text-embedding-3-large'"
                )
        
        except requests.exceptions.RequestException as e:
            raise DataSourceUnavailableError(
                f"Embedding service unavailable. Failed to call {embedding_model} API: {str(e)}\n"
                f"Check API credentials and network connectivity."
            )
        except (KeyError, ValueError) as e:
            raise DataSourceUnavailableError(
                f"Embedding API returned invalid response: {str(e)}\n"
                f"Ensure API credentials are valid and model {embedding_model} is available."
            )
```

### 3.6 ClaudeReasoningAction

#### Contract

**Input**: `entity_features` (EntityFeatures), `rules` (RankedRules)
**Output**: `ComplianceAnalysis` (dict with answer, citations, reasoning_log, timestamp)
**Config**: `Param_tools` (list), `Param_prompt_template` (path)
**Raises**: `ValidationError`, `DataSourceUnavailableError`

#### Implementation

```python
from anthropic import Anthropic
from datetime import datetime, timezone
from jinja2 import Environment, FileSystemLoader

class ClaudeReasoningAction(Action):
    """Use Claude to reason over rules and produce compliance analysis."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Invoke Claude to analyze product against retrieved FCA rules.
        
        Args:
            node_input: {
                'entity_features': dict (EntityFeatures),
                'rules': list (RankedRules from semantic_search)
            }
            config: {
                'Param_tools': list (e.g., ['citation_formatter', 'audit_logger']),
                'Param_prompt_template': str (path to prompt template file)
            }
        
        Returns:
            {
                'answer': str (Claude's analysis),
                'citations': list of {'rule_id', 'cited_text', 'context', 'binding_level'},
                'reasoning_log': {'reasoning_chain': str, 'gaps_identified': list, 'confidence_score': float},
                'timestamp': str (ISO 8601)
            }
        """
        entity_features = node_input.get('entity_features', {})
        rules = node_input.get('rules', [])
        
        if not isinstance(rules, list) or len(rules) == 0:
            raise ValidationError("rules must be non-empty list of RankedRules")
        
        try:
            prompt_template_path = config.get('Param_prompt_template')
            if not prompt_template_path:
                raise ValidationError("Param_prompt_template required")
            with open(prompt_template_path) as f:
                template_text = f.read()
        except FileNotFoundError:
            raise ValidationError(f"Prompt template not found at {prompt_template_path}")
        
        # Construct system prompt using Jinja2
        jinja_env = Environment()
        template = jinja_env.from_string(template_text)
        system_prompt = template.render(
            entity_features=json.dumps(entity_features, indent=2),
            ranked_rules=json.dumps(rules, indent=2)
        )
        
        # Build tool list from config (or default to citation_formatter and audit_logger)
        tools_config = config.get('Param_tools', ['citation_formatter', 'audit_logger'])
        tool_map = {
            'citation_formatter': CITATION_FORMATTER_TOOL,
            'audit_logger': AUDIT_LOGGER_TOOL
        }
        tools = [tool_map[name] for name in tools_config if name in tool_map]
        if not tools:
            tools = [CITATION_FORMATTER_TOOL, AUDIT_LOGGER_TOOL]
        
        # Call Claude with extended thinking and prompt caching
        client = Anthropic()
        
        response = client.messages.create(
            model="claude-opus-4-7",
            max_tokens=4000,
            temperature=0.5,
            thinking={
                "type": "enabled",
                "budget_tokens": 1000
            },
            system=[
                {
                    "type": "text",
                    "text": system_prompt,
                    "cache_control": {"type": "ephemeral"}
                }
            ],
            tools=tools,
            messages=[
                {
                    "role": "user",
                    "content": "Perform the compliance analysis using the tools to cite rules and log reasoning."
                }
            ]
        )
        
        # Extract citations and reasoning from response
        citations = self._extract_citations(response, rules)
        reasoning_log = self._extract_reasoning_log(response)
        answer = self._extract_answer(response)
        
        # Validate citations
        for citation in citations:
            if citation['rule_id'] not in [r['rule_id'] for r in rules]:
                # Flag for review but don't halt
                reasoning_log['_citation_warning'] = f"Citation {citation['rule_id']} not in retrieved rules"
        
        timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
        
        return {
            'answer': answer,
            'citations': citations,
            'reasoning_log': reasoning_log,
            'timestamp': timestamp
        }
    
    def _extract_citations(self, response, rules) -> List[Dict]:
        """
        Extract citation_formatter tool calls from Claude response.
        
        Iterates over response.content blocks, finds tool_use blocks where
        name == 'citation_formatter', and validates rule_id against retrieved rules.
        """
        citations = []
        rule_ids = {r['rule_id'] for r in rules}
        
        if not hasattr(response, 'content') or not response.content:
            return citations
        
        for block in response.content:
            # Check if this is a tool_use block with name == 'citation_formatter'
            if hasattr(block, 'type') and block.type == 'tool_use':
                if hasattr(block, 'name') and block.name == 'citation_formatter':
                    try:
                        # Extract input data from tool call
                        tool_input = block.input
                        
                        rule_id = tool_input.get('rule_id')
                        cited_text = tool_input.get('cited_text')
                        context = tool_input.get('context')
                        binding_level = tool_input.get('binding_level')
                        
                        # Validate rule_id exists in retrieved rules
                        if rule_id not in rule_ids:
                            # Log warning but continue processing other citations
                            import logging
                            logging.warning(
                                f"Citation references rule_id '{rule_id}' not in retrieved rules. "
                                f"Available: {sorted(rule_ids)}"
                            )
                            continue
                        
                        # Validate required fields present
                        if not all([rule_id, cited_text, context, binding_level]):
                            import logging
                            logging.warning(
                                f"Citation incomplete: rule_id={rule_id}, cited_text={bool(cited_text)}, "
                                f"context={bool(context)}, binding_level={binding_level}"
                            )
                            continue
                        
                        citations.append({
                            'rule_id': rule_id,
                            'cited_text': cited_text,
                            'context': context,
                            'binding_level': binding_level
                        })
                    
                    except (AttributeError, KeyError, TypeError) as e:
                        import logging
                        logging.warning(f"Failed to parse citation_formatter tool call: {str(e)}")
                        continue
        
        return citations
    
    def _extract_reasoning_log(self, response) -> Dict:
        """
        Extract audit_logger tool calls from Claude response.
        
        Iterates over response.content blocks, finds tool_use blocks where
        name == 'audit_logger', and extracts reasoning chain and confidence metrics.
        """
        reasoning_log = {
            'reasoning_chain': '',
            'gaps_identified': [],
            'confidence_score': 0.0
        }
        
        if not hasattr(response, 'content') or not response.content:
            return reasoning_log
        
        for block in response.content:
            # Check if this is a tool_use block with name == 'audit_logger'
            if hasattr(block, 'type') and block.type == 'tool_use':
                if hasattr(block, 'name') and block.name == 'audit_logger':
                    try:
                        # Extract input data from tool call
                        tool_input = block.input
                        
                        reasoning_chain = tool_input.get('reasoning_chain', '')
                        gaps_identified = tool_input.get('gaps_identified', [])
                        confidence_score = tool_input.get('confidence_score', 0.0)
                        
                        # Validate and normalize data
                        if not isinstance(gaps_identified, list):
                            gaps_identified = [str(gaps_identified)] if gaps_identified else []
                        
                        if not isinstance(confidence_score, (int, float)):
                            confidence_score = 0.0
                        else:
                            confidence_score = float(max(0.0, min(1.0, confidence_score)))
                        
                        reasoning_log = {
                            'reasoning_chain': str(reasoning_chain) if reasoning_chain else '',
                            'gaps_identified': [str(g) for g in gaps_identified],
                            'confidence_score': confidence_score
                        }
                        
                        # Return first valid audit_logger call (should only be one per response)
                        return reasoning_log
                    
                    except (AttributeError, KeyError, TypeError, ValueError) as e:
                        import logging
                        logging.warning(f"Failed to parse audit_logger tool call: {str(e)}")
                        continue
        
        return reasoning_log
    
    def _extract_answer(self, response) -> str:
        """
        Extract text response from Claude.
        
        Concatenates all text blocks from response.content (type == 'text'),
        skipping tool_use and metadata blocks.
        """
        text_blocks = []
        
        if not hasattr(response, 'content') or not response.content:
            return ''
        
        for block in response.content:
            # Extract text blocks only (skip tool_use, thinking, input, etc.)
            if hasattr(block, 'type') and block.type == 'text':
                if hasattr(block, 'text'):
                    text = block.text.strip()
                    if text:  # Only add non-empty text
                        text_blocks.append(text)
        
        # Concatenate with double newline to preserve paragraph spacing
        answer = '\n\n'.join(text_blocks)
        
        return answer.strip()
```

### 3.7 ApprovalGateAction

#### Contract

**Input**: `analysis` (ComplianceAnalysis)
**Output**: `ApprovalDecision` (dict with approved, approver_name, approver_email, timestamp, actions_carried_out, analysis_snapshot)
**Config**: `Param_required_approval` (role, record_identity)
**Raises**: `ApprovalError`, `TimeoutError`

#### Implementation

```python
from datetime import datetime, timezone

class ApprovalGateAction(Action):
    """Present compliance analysis for formal human review and approval."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Present analysis to user and capture formal approval decision.
        
        Args:
            node_input: {
                'analysis': ComplianceAnalysis
            }
            config: {
                'Param_required_approval': {
                    'Param_role': str (e.g., 'compliance_officer'),
                    'Param_record_identity': bool (true = capture approver name/email)
                }
            }
        
        Returns:
            {
                'approved': bool,
                'approver_name': str,
                'approver_email': str,
                'approver_role': str,
                'timestamp': str (ISO 8601 UTC),
                'actions_carried_out': list of str,
                'comments': str,
                'analysis_snapshot': ComplianceAnalysis
            }
        """
        analysis = node_input.get('analysis', {})
        
        if not isinstance(analysis, dict) or 'citations' not in analysis:
            raise ValidationError("analysis must be a valid ComplianceAnalysis object")
        
        required_role = config.get('Param_required_approval', {}).get('Param_role')
        record_identity = config.get('Param_required_approval', {}).get('Param_record_identity', False)
        
        # Present analysis (stub: in real implementation, render to UI or CLI)
        # decision = prompt_user_for_approval(analysis, required_role)
        # Stub implementation:
        decision = {
            'approved': True,
            'approver_name': 'Jane Doe',
            'approver_email': 'jane.doe@company.com',
            'comments': 'Approved pending final audit.'
        }
        
        if record_identity and (not decision.get('approver_name') or not decision.get('approver_email')):
            raise ApprovalError("Approver name and email required when Param_record_identity=true")
        
        timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
        
        return {
            'approved': decision['approved'],
            'approver_name': decision.get('approver_name'),
            'approver_email': decision.get('approver_email'),
            'approver_role': required_role,
            'timestamp': timestamp,
            'actions_carried_out': ['approved', 'identity_verified'] if record_identity else ['approved'],
            'comments': decision.get('comments', ''),
            'analysis_snapshot': analysis
        }
```

---

## 4. Harness Orchestration

### 4.1 ComplianceHarness Class

Core orchestration: loads YAML workflows and manages node execution.

```python
import yaml
import json
from typing import Dict, Any

class ComplianceHarness:
    """Orchestrates workflow execution across action nodes."""
    
    def __init__(self, workflow_config_path: str):
        """
        Initialize harness: load YAML workflow definition, instantiate actions, load data layers.
        
        Args:
            workflow_config_path: Path to harness.yaml (defines workflows and data sources)
        """
        with open(workflow_config_path) as f:
            config = yaml.safe_load(f)
        
        self.workflows = config.get('harness', {}).get('workflows', {})
        self.data_sources = config.get('harness', {}).get('data_sources', {})
        
        # Load handbook index (data layer)
        fca_handbook_config = self.data_sources.get('fca_handbook', {})
        handbook_path = fca_handbook_config.get('artifact')
        self.handbook_index = HandbookIndex(handbook_path)
        
        # Instantiate action classes with harness reference for data access
        self.actions = {
            name: ACTION_REGISTRY[name](self) 
            for name in ACTION_REGISTRY.keys()
        }
```

### 4.2 Workflow Execution

Execute a workflow node-by-node, passing outputs to dependent nodes.

```python
    def execute_workflow(self, workflow_name: str, entity_description: str, question: str) -> Dict[str, Any]:
        """
        Execute named workflow end-to-end.
        
        Args:
            workflow_name: Name from YAML (e.g., 'general_enquiry')
            entity_description: Markdown product description
            question: User's compliance question
        
        Returns:
            Workflow context dict with all node outputs
        """
        workflow = self.workflows.get(workflow_name)
        if not workflow:
            raise ValidationError(f"Workflow '{workflow_name}' not found")
        
        nodes = workflow.get('nodes', [])
        context = {
            'entity_description': entity_description,
            'question': question
        }
        
        # Execute each node in order
        for node in nodes:
            node_name = node['name']
            action_type = node['action']
            node_input = self._prepare_input(node['input'], context)
            node_config = node.get('config', {})
            
            try:
                action = self.actions[action_type]
                output = action.execute(node_input, node_config)
                context[node_name] = output
                self._log_interaction(node_name, 'success')
            except ActionError as e:
                self._log_error(node_name, str(e))
                raise
        
        return context
    
    def _prepare_input(self, input_spec: Any, context: Dict) -> Dict[str, Any]:
        """
        Prepare node input by resolving references to prior node outputs.
        
        Input spec may be:
        - A string (key name): resolve from context
        - A dict: recursively resolve values that reference context
        """
        if isinstance(input_spec, str):
            return context.get(input_spec, input_spec)
        elif isinstance(input_spec, dict):
            result = {}
            for key, value in input_spec.items():
                if isinstance(value, str) and value in context:
                    result[key] = context[value]
                elif isinstance(value, dict):
                    result[key] = self._prepare_input(value, context)
                else:
                    result[key] = value
            return result
        return input_spec
    
    def _validate_node_output(self, output: Any, expected_schema: str) -> bool:
        """Validate node output against expected schema."""
        # Stub: implement schema validation (e.g., jsonschema)
        return True
```

### 4.3 Audit Logging

Log all interactions and errors for compliance audit trails.

```python
    def _log_interaction(self, node_name: str, status: str):
        """Log successful node execution."""
        timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
        log_entry = {
            'timestamp': timestamp,
            'node': node_name,
            'status': status
        }
        # Stub: write to audit log (file, database, or event stream)
        print(json.dumps(log_entry))
    
    def _log_error(self, node_name: str, error_message: str):
        """Log node execution error."""
        timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
        log_entry = {
            'timestamp': timestamp,
            'node': node_name,
            'status': 'error',
            'error': error_message
        }
        # Stub: write to audit log
        print(json.dumps(log_entry))
```

### 4.4 Tool Integration Entry Point

Expose the harness as a named tool for external LLMs (e.g., Claude).

```python
def fca_handbook_compliance_enquiry(entity_description: str, question: str) -> str:
    """
    FCA Handbook Compliance Enquiry Tool.
    
    Invoke the harness as a tool from external LLM.
    """
    if not question or not entity_description:
        return json.dumps({
            'scope_clarification': 'Do you mean compliance with the **FCA Handbook**? '
                                   'This tool handles FCA Handbook rules specifically.'
        })
    
    try:
        harness = ComplianceHarness('harness.yaml')
        result = harness.execute_workflow('general_enquiry', entity_description, question)
        return json.dumps(result, default=str)
    except ActionError as e:
        return json.dumps({'error': str(e)})

def _is_fca_scope(question: str) -> bool:
    """Check if question is about FCA Handbook."""
    fca_keywords = ['fca', 'handbook', 'cobs', 'prin', 'compliance']
    return any(kw in question.lower() for kw in fca_keywords)
```

---

## 5. Testing

### Unit Test Pattern (Per-Action)

```python
import unittest
from unittest.mock import patch, MagicMock

class TestParseMarkdownAction(unittest.TestCase):
    
    def setUp(self):
        self.action = ParseMarkdownAction()
    
    def test_valid_entity_description(self):
        """Test parsing well-formed markdown."""
        markdown = """# AdviceBot
        ## Features
        - Robo-advice engine
        - Real-time rebalancing
        ## Use Cases
        - Wealth management
        """
        result = self.action.execute({'entity_description': markdown}, {})
        self.assertEqual(result['entity_name'], 'AdviceBot')
        self.assertIn('Robo-advice engine', result['features'])
    
    def test_missing_h1_raises_error(self):
        """Test that missing product name (h1) raises ValidationError."""
        markdown = "## Features\n- Feature 1"
        with self.assertRaises(ValidationError):
            self.action.execute({'entity_description': markdown}, {})

class TestSemanticSearchAction(unittest.TestCase):
    
    @patch('handbook_index.search')
    def test_returns_ranked_rules(self, mock_search):
        """Test that search returns properly formatted RankedRules."""
        mock_search.return_value = [
            {'rule_id': 'COBS 2.1.1R', 'text': 'Rule text', 'base_similarity': 0.9}
        ]
        action = SemanticSearchAction()
        result = action.execute({
            'entity_features': {'entity_name': 'Test'},
            'question_terms': {'all_search_terms': []},
            'question': 'What rules apply?'
        }, {'Param_top_k': 20})
        self.assertIsInstance(result, list)
        self.assertGreater(len(result), 0)
```

### Integration Test Pattern (Full Workflow)

```python
class TestGeneralEnquiryWorkflow(unittest.TestCase):
    
    def setUp(self):
        self.harness = ComplianceHarness("tests/fixtures/harness.yaml")
    
    def test_full_workflow_execution(self):
        """Test end-to-end execution with fixture data."""
        result = self.harness.execute_workflow('general_enquiry',
            entity_description="# TestProduct\n## Features\n- Feature1\n## Use Cases\n- Use1\n## Decision Authority\nAlgorithm",
            question="What COBS rules apply?"
        )
        self.assertIn('human_review', result)
        self.assertIn('approved', result['human_review'])
```

---

## 6. Implementation Checklist

Follow this order to implement the Harness:

### Phase 1: Foundations
- [ ] Define exception classes (§2.1)
- [ ] Define Action base class (§2.2)
- [ ] Define Claude tool schemas as module constants (§2.4)
- [ ] Define Common Patterns helpers (timestamp, validation, config extraction)

### Phase 2: Data Layer
- [ ] Implement HandbookIndex: load embeddings, detect model, validate dimensions (§3.3)
- [ ] Test HandbookIndex._detect_embedding_key with sample JSON
- [ ] Implement HandbookIndex.search with cosine similarity + weighting

### Phase 3: Deterministic Actions
- [ ] Implement ParseMarkdownAction (§3.1) — no external dependencies
- [ ] Implement GlossaryLookupAction (§3.2) — no external dependencies
- [ ] Unit test both; verify error messages are clear

### Phase 4: LLM-Dependent Actions
- [ ] Implement SemanticSearchAction with HandbookIndex integration (§3.3)
- [ ] Implement ClaudeReasoningAction with tool integration (§3.4)
- [ ] Implement ApprovalGateAction human gate (§3.5)
- [ ] Unit test with mocked Claude API

### Phase 5: Orchestration
- [ ] Implement ComplianceHarness class (§4.1–4.3)
- [ ] Implement workflow execution loop (§4.2)
- [ ] Implement audit logging (§4.3)
- [ ] Register all actions in ACTION_REGISTRY (§2.3)

### Phase 6: External Integration
- [ ] Implement tool integration entry point (§4.4)
- [ ] Wire Harness into external LLM SDK (Anthropic, OpenAI)
- [ ] Test tool invocation end-to-end

### Phase 7: Testing & Validation
- [ ] Write unit tests for each action (§5)
- [ ] Write integration test for full workflow (§5)
- [ ] Test error handling for each exception type
- [ ] Test audit logging with real and fixture data

### Phase 8: Deployment
- [ ] Load production FCA_Handbook_Text_And_Embeddings
- [ ] Configure harness.yaml with correct paths and versions
- [ ] Deploy to target environment (direct Python, FastAPI service, or platform)
- [ ] Smoke test with sample queries
