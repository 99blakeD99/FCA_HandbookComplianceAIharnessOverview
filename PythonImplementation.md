# Python Implementation

This document provides the patterns, interfaces, and concrete implementations needed to build the Compliance Agent Harness in Python. It implements the actions and orchestration patterns specified in GeneralEnquirySpecs.md.

## Architecture Overview

The Python harness has three layers:

1. **Orchestration Layer**: Loads YAML workflows, manages node execution, passes data between nodes
2. **Action Layer**: Implements each action type (parse_markdown, semantic_search, claude_reasoning, approval_gate) with input validation and error handling
3. **Data Layer**: Manages FCA Handbook embeddings (in-memory), configuration files (weights.yaml (schema in StructuredSearch.md), prompts), and audit logging

```
User Input (entity_description, question)
    ↓
Tool Invocation → Scope Validation
    ↓
Harness Initialization (load YAML, embeddings, configs)
    ↓
Node Execution Loop
    ├─ NODE 1: ParseMarkdownAction
    ├─ NODE 2: SemanticSearchAction
    ├─ NODE 3: ClaudeReasoningAction
    └─ NODE 4: ApprovalGateAction
    ↓
Audit Trail + ComplianceAnalysis Output
```

## Deployment Models

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

## Exception Hierarchy

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

## Action Registry

Register all actions here. The harness uses this registry to look up action implementations by name from the YAML workflow:

```python
ACTION_REGISTRY = {
    "parse_markdown": ParseMarkdownAction,
    "semantic_search": SemanticSearchAction,
    "claude_reasoning": ClaudeReasoningAction,
    "approval_gate": ApprovalGateAction,
}
```

## Claude Tool Schemas

These tools enforce structured output from Claude for citations and reasoning logs (used in ClaudeReasoningAction).

### citation_formatter

Claude uses this tool to cite rules by rule_id with verbatim text and context.

```json
{
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
```

### audit_logger

Claude uses this tool to emit structured reasoning logs for auditability.

```json
{
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

## Action Base Class

All actions inherit from a common interface:

```python
from abc import ABC, abstractmethod
from typing import Any, Dict

class Action(ABC):
    """Abstract base class for all harness actions."""
    
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

## Implementation: ParseMarkdownAction

Extracts structured product information from markdown. See GeneralEnquirySpecs.md for the complete contract.

```python
import re
from typing import Dict, List, Any

class ParseMarkdownAction(Action):
    """Extract and structure product information from markdown."""
    
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
        # Find next heading or end of text
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
        # Remove list markers and return first paragraph/line
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

## Implementation: SemanticSearchAction

Performs vector similarity search over FCA Handbook embeddings with regulatory weighting. See GeneralEnquirySpecs.md and StructuredSearch.md for the complete specification.

### HandbookIndex Class

Load and index the FCA Handbook once at startup:

```python
import json
import numpy as np
import yaml

class HandbookIndex:
    """Load and index FCA Handbook embeddings for semantic search."""
    
    def __init__(self, handbook_path: str, weights_config_path: str = None):
        """
        Load handbook data and weighting configuration.
        
        Args:
            handbook_path: Path to FCA_Handbook_Text_And_Embeddings JSON file
            weights_config_path: Unused (kept for compatibility). Weights are hardcoded from StructuredSearch.md
        """
        # Load handbook data
        with open(handbook_path) as f:
            data = json.load(f)
        
        self.records = data['fca_handbook']
        
        # Detect and load embeddings: try common embedding key names (Voyage or OpenAI)
        self.embedding_key = self._detect_embedding_key(self.records[0] if self.records else {})
        self.embeddings = np.array([record[self.embedding_key] for record in self.records])
        
        # Map embedding key to model name for query embedding
        self.embedding_model = self._get_model_name(self.embedding_key)
        
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
                'high': 1.3,      # Scores 7-12
                'medium': 1.1,    # Scores 4-6
                'low': 0.7        # Scores 1-3
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
        Raises error if no embedding key found.
        """
        # Try keys in order of preference (Voyage 3-Large is default)
        embedding_keys = [
            'voyage-3-large_embedding',
            'OpenAI_text-embedding-3-large_embedding',
            'embedding'  # Fallback
        ]
        
        for key in embedding_keys:
            if key in sample_record:
                return key
        
        # No embedding key found
        available_keys = list(sample_record.keys())
        raise DataSourceUnavailableError(
            f"No embedding key found in FCA Handbook record. "
            f"Tried: {embedding_keys}. Available keys: {available_keys}"
        )
    
    def _get_model_name(self, embedding_key: str) -> str:
        """Map embedding key to model name for query embedding."""
        model_map = {
            'voyage-3-large_embedding': 'voyage-3-large',
            'OpenAI_text-embedding-3-large_embedding': 'text-embedding-3-large',
            'embedding': 'text-embedding-3-large'  # Assume OpenAI if key is generic
        }
        return model_map.get(embedding_key, 'text-embedding-3-large')
    
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
        
        # Vectorized cosine similarity (faster than loop over 10,438 embeddings)
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
        """Compute all weight factors for a record using hardcoded defaults from StructuredSearch.md."""
        rule_type_weight = self._rule_type_weight(record.get('regulatory_content', ''))
        
        # Hierarchy multiplier: look up record['level'] in weights config
        level = record.get('level', 2)  # Default to level 2 if not specified
        hierarchy_multipliers = self.weights.get('hierarchy_multipliers', {})
        hierarchy_multiplier = hierarchy_multipliers.get(level, hierarchy_multipliers.get('default', 0.9))
        
        importance_multiplier = self._importance_weight(record.get('regulatory_score', 5))
        
        # Piece weight: look up record['piece'] in weights config
        piece = record.get('piece', 'Main Handbook')  # Default to Main Handbook
        piece_base_weights = self.weights.get('piece_base_weights', {})
        piece_weight = piece_base_weights.get(piece, 1.0)
        
        return {
            'rule_type_weight': rule_type_weight,
            'hierarchy_multiplier': hierarchy_multiplier,
            'importance_multiplier': importance_multiplier,
            'piece_weight': piece_weight
        }
    
    def _rule_type_weight(self, regulatory_content: str) -> float:
        """Extract rule type from regulatory_content and return weight.
        
        Rule type appears as single letter [RGED] after rule ID at start of content.
        If not found, assign 'U' (Unclassified) for annexes/definitions/forms.
        """
        import re
        
        # Extract rule type from start of regulatory_content: RULE_ID TYPE
        match = re.search(r'\b([A-Z0-9]+\s+[\d.A-Z]+)\s+([RGED])\b', regulatory_content)
        
        if match:
            suffix = match.group(2)
        else:
            # No explicit type found -> Unclassified
            suffix = 'U'
        
        type_weights = self.weights.get('rule_type_weights', {
            'RULES': 1.0, 'GUIDANCE': 0.8, 'EVIDENTIAL': 0.6, 
            'UNCLASSIFIED': 0.8, 'DELETED': 0.1
        })
        # Map suffix to type
        type_map = {'R': 'RULES', 'G': 'GUIDANCE', 'E': 'EVIDENTIAL', 
                    'U': 'UNCLASSIFIED', 'D': 'DELETED'}
        rule_type = type_map.get(suffix, 'UNCLASSIFIED')
        return type_weights.get(rule_type, 0.8)
    
    def _importance_weight(self, regulatory_score: int) -> float:
        """Map regulatory score to importance multiplier."""
        importance = self.weights.get('importance_multipliers', {
            'high': 1.3, 'medium': 1.1, 'low': 0.7
        })
        if regulatory_score >= 7:
            return importance.get('high', 1.3)
        elif regulatory_score >= 4:
            return importance.get('medium', 1.1)
        else:
            return importance.get('low', 0.7)

# Global instance (initialized once at harness startup)
handbook_index = None

def initialize_handbook(handbook_path: str, weights_path: str):
    """Initialize the handbook index. Call once at harness startup."""
    global handbook_index
    try:
        handbook_index = HandbookIndex(handbook_path, weights_path)
    except Exception as e:
        raise DataSourceUnavailableError(
            f"Failed to initialize FCA Handbook: {str(e)}"
        )
```

### SemanticSearchAction

```python
from openai import OpenAI

class SemanticSearchAction(Action):
    """Retrieve and rank FCA rules via semantic similarity."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Search for FCA rules matching product features and question.
        
        Args:
            node_input: {
                'entity_features': dict (output from ParseMarkdownAction),
                'question': str (compliance question)
            }
            config: {
                'Param_top_k': int (number of results, default 20),
                'Param_source_version': str (FCA Handbook version, e.g., '2026-Q1', from harness config)
            }
        
        Returns:
            List of ranked rules with similarity and weight factor breakdown
        """
        if handbook_index is None:
            raise DataSourceUnavailableError(
                "FCA Handbook data (FCA_Handbook_Text_And_Embeddings) is temporarily unavailable. "
                "Escalate to compliance_team@company.com"
            )
        
        # Validate inputs
        entity_features = node_input.get('entity_features', {})
        question = node_input.get('question', '')
        
        if not isinstance(entity_features, dict):
            raise ValidationError("entity_features must be a dictionary")
        if not question or not isinstance(question, str):
            raise ValidationError("question must be a non-empty string")
        if len(question) < 5 or len(question) > 1000:
            raise ValidationError("question must be 5-1000 characters")
        
        top_k = config.get('Param_top_k', 20)
        if not isinstance(top_k, int) or top_k < 1 or top_k > 100:
            raise ValidationError("Param_top_k must be integer in range [1, 100]")
        
        source_version = config.get('Param_source_version', '2026-Q1')  # From harness config
        
        # Combine entity features and question as text, then embed once.
        # Approach: Concatenate text (not separate embeddings) for deterministic, fair cosine similarity comparison.
        features_text = " ".join([
            entity_features.get('entity_name', ''),
            entity_features.get('entity_type', ''),
            " ".join(entity_features.get('features', [])),
            " ".join(entity_features.get('use_cases', []))
        ])
        query_text = f"{features_text}\n{question}"
        
        # Get query embedding using the same model as the handbook
        try:
            # Use the embedding model detected from the handbook (Voyage or OpenAI)
            embedding_model = handbook_index.embedding_model if handbook_index else 'text-embedding-3-large'
            client = OpenAI()
            response = client.embeddings.create(
                input=query_text,
                model=embedding_model
            )
            query_embedding = response.data[0].embedding
        except Exception as e:
            raise DataSourceUnavailableError(
                f"Embedding service unavailable. Unable to query FCA Handbook. {str(e)}"
            )
        
        # Search using pre-loaded handbook index
        results = handbook_index.search(query_embedding, top_k=top_k)
        
        # Format results as RankedRules
        ranked_rules = []
        for result in results:
            rule = {
                'rule_id': result['record'].get('id'),
                'text': result['record'].get('regulatory_content'),
                'base_similarity': result['base_similarity'],
                'weight_factors': result['weight_factors'],
                'final_score': result['final_score'],
                'source_version': source_version  # From harness config (Param_source_version)
            }
            ranked_rules.append(rule)
        
        return ranked_rules if ranked_rules else []
```

## Implementation: ClaudeReasoningAction

Invokes Claude to reason over retrieved rules and produce structured compliance analysis.

```python
from anthropic import Anthropic
from datetime import datetime

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
        # Validate inputs
        entity_features = node_input.get('entity_features', {})
        rules = node_input.get('rules', [])
        
        if not isinstance(rules, list) or len(rules) == 0:
            raise ValidationError("rules must be non-empty list of RankedRules")
        
        # Validate config
        try:
            prompt_template_path = config.get('Param_prompt_template')
            if not prompt_template_path:
                raise ValidationError("Param_prompt_template required")
            with open(prompt_template_path) as f:
                prompt_template = f.read()
        except FileNotFoundError:
            raise ValidationError(f"Prompt template not found at {prompt_template_path}")
        
        # Build system message from template
        system_message = self._build_system_message(
            prompt_template, entity_features, rules
        )
        
        # Call Claude with prompt caching
        try:
            client = Anthropic()
            response = client.messages.create(
                model="claude-opus-4-7",
                max_tokens=2000,
                temperature=0.5,
                thinking={"type": "enabled", "budget_tokens": 5000},
                system=[
                    {
                        "type": "text",
                        "text": system_message,
                        "cache_control": {"type": "ephemeral"}
                    }
                ],
                tools=[
                    {
                        "name": "citation_formatter",
                        "description": "Extract and format a specific FCA rule citation with exact text and context.",
                        "input_schema": {
                            "type": "object",
                            "properties": {
                                "rule_id": {"type": "string", "description": "FCA rule identifier (e.g., COBS 2.1.1R)"},
                                "cited_text": {"type": "string", "description": "Verbatim text from the rule"},
                                "context": {"type": "string", "description": "Sentence showing why this rule applies"},
                                "binding_level": {"type": "string", "enum": ["R", "G", "E", "D"], "description": "Binding level from rule_id"}
                            },
                            "required": ["rule_id", "cited_text", "context", "binding_level"]
                        }
                    },
                    {
                        "name": "audit_logger",
                        "description": "Log the reasoning chain, identified gaps, and confidence score.",
                        "input_schema": {
                            "type": "object",
                            "properties": {
                                "reasoning_chain": {"type": "string", "description": "Step-by-step reasoning logic"},
                                "gaps_identified": {"type": "array", "items": {"type": "string"}, "description": "Edge cases or uncertainties"},
                                "confidence_score": {"type": "number", "minimum": 0, "maximum": 1, "description": "Model confidence in analysis"}
                            },
                            "required": ["reasoning_chain", "gaps_identified", "confidence_score"]
                        }
                    }
                ],
                messages=[
                    {
                        "role": "user",
                        "content": f"Analyze this product for FCA compliance and cite applicable rules. Use citation_formatter for each applicable rule and audit_logger to record your reasoning."
                    }
                ]
            )
        except Exception as e:
            raise ActionError(f"LLM service unavailable. Unable to reason over rules. {str(e)}")
        
        # Extract answer from response text blocks
        answer_text = ""
        for block in response.content:
            if hasattr(block, 'text') and block.type == 'text':
                answer_text = block.text
                break
        
        # Parse Claude's response from tool use blocks
        citations = self._extract_citations(response)
        reasoning_log = self._extract_reasoning_log(response)
        
        # Validate citations against source rules
        for citation in citations:
            self._validate_citation(citation, rules)
        
        return {
            'answer': answer_text,
            'citations': citations,
            'reasoning_log': reasoning_log,
            'timestamp': datetime.now(timezone.utc).isoformat() + 'Z'
        }
    
    def _build_system_message(self, template: str, features: Dict, rules: List) -> str:
        """Render Jinja2 prompt template with product features and rules."""
        from jinja2 import Template
        features_str = json.dumps(features, indent=2)
        rules_str = json.dumps(rules, indent=2)
        # Template syntax: {{ entity_features }}, {{ ranked_rules }}
        jinja_template = Template(template)
        return jinja_template.render(entity_features=features_str, ranked_rules=rules_str)
    
    def _extract_citations(self, response) -> List[Dict]:
        """
        Extract citations from Claude's structured tool_use output (citation_formatter tool).
        Claude must use the citation_formatter tool to emit rule_id + quoted text.
        Do NOT parse free-form text with regex.
        """
        citations = []
        # Parse Claude's tool_use calls for citation_formatter
        for block in response.content:
            if block.type == 'tool_use' and block.name == 'citation_formatter':
                tool_input = block.input
                citations.append({
                    'rule_id': tool_input.get('rule_id'),
                    'cited_text': tool_input.get('cited_text'),
                    'context': tool_input.get('context'),
                    'binding_level': tool_input.get('binding_level')
                })
        return citations
    
    def _extract_reasoning_log(self, response) -> Dict:
        """
        Extract reasoning log from Claude's structured tool_use output (audit_logger tool).
        Claude must use the audit_logger tool to emit reasoning_chain, gaps_identified, confidence_score.
        Do NOT parse free-form text.
        """
        # Parse Claude's tool_use calls for audit_logger
        for block in response.content:
            if block.type == 'tool_use' and block.name == 'audit_logger':
                return block.input  # Assumes input matches reasoning_log schema
        # Fallback if no audit_logger output
        return {'reasoning_chain': '', 'gaps_identified': [], 'confidence_score': 0.0}
    
    def _validate_citation(self, citation: Dict, rules: List) -> bool:
        """
        Validate that citation's rule_id exists in RankedRules and cited_text matches rule text.
        Uses rule_id lookup (not regex).
        Logs warnings for validation failures instead of raising errors.
        
        Returns:
            bool: True if valid, False if validation failed (but doesn't halt workflow)
        """
        rule_id = citation.get('rule_id')
        cited_text = citation.get('cited_text')
        
        # Look up rule by rule_id from RankedRules
        matching_rule = next((r for r in rules if r.get('rule_id') == rule_id), None)
        if not matching_rule:
            timestamp = datetime.now(timezone.utc).isoformat() + 'Z'
            print(f"[{timestamp}] WARNING: Citation rule_id '{rule_id}' not found in retrieved rules")
            return False
        
        # Validate cited_text appears verbatim in rule text
        rule_text = matching_rule.get('text', '')
        if cited_text not in rule_text:
            timestamp = datetime.now(timezone.utc).isoformat() + 'Z'
            print(f"[{timestamp}] WARNING: Citation text mismatch for {rule_id}: cited text not found in rule source")
            return False
        
        return True
```

## Implementation: ApprovalGateAction

Routes analysis to human approver and records decision with audit trail.

```python
class ApprovalGateAction(Action):
    """Route compliance analysis to human approver."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Present analysis to human approver and record decision.
        
        Args:
            node_input: {
                'analysis': dict (ComplianceAnalysis from claude_reasoning)
            }
            config: {
                'Param_required_approval': {
                    'Param_role': str (e.g., 'compliance_officer'),
                    'Param_record_identity': bool (whether to capture approver name/email)
                }
            }
        
        Returns:
            {
                'approved': bool,
                'approver_name': str (if record_identity=true),
                'approver_email': str (if record_identity=true),
                'approver_role': str,
                'timestamp': str (ISO 8601 UTC),
                'actions_carried_out': list (audit trail of actions),
                'comments': str (optional approver notes),
                'analysis_snapshot': dict (copy of analysis at approval time)
            }
        """
        # Validate input
        analysis = node_input.get('analysis')
        if not isinstance(analysis, dict):
            raise ValidationError("analysis must be a ComplianceAnalysis object")
        
        required_fields = ['citations', 'reasoning_log', 'answer']
        for field in required_fields:
            if field not in analysis or not analysis[field]:
                raise ValidationError(f"ComplianceAnalysis incomplete: missing {field}")
        
        # Get config
        required_approval = config.get('Param_required_approval', {})
        required_role = required_approval.get('Param_role', 'compliance_officer')
        record_identity = required_approval.get('Param_record_identity', False)
        
        # Request formal approval from user
        # Harness presents analysis and requests explicit approval/rejection with formal identity
        approval_decision = self._request_formal_approval(
            analysis, required_role, record_identity
        )
        
        # Validate approver identity if required
        actions_carried_out = ['presented_for_review']
        if record_identity:
            if not approval_decision.get('approver_name') or not approval_decision.get('approver_email'):
                raise ValidationError("approver_name and approver_email required when record_identity=true")
            
            # Validate email domain
            email = approval_decision.get('approver_email', '')
            if not self._validate_email_domain(email):
                raise ApprovalError(f"Approver email invalid or not authorized: {email}")
            
            actions_carried_out.append('identity_verified')
        
        # Record timestamp in UTC
        timestamp = datetime.now(timezone.utc).isoformat() + 'Z'
        actions_carried_out.append('snapshot_recorded')
        
        return {
            'approved': approval_decision.get('approved', False),
            'approver_name': approval_decision.get('approver_name'),
            'approver_email': approval_decision.get('approver_email'),
            'approver_role': required_role,
            'timestamp': timestamp,
            'actions_carried_out': actions_carried_out,
            'comments': approval_decision.get('comments', ''),
            'analysis_snapshot': analysis
        }
    
    def _request_formal_approval(self, analysis: Dict, role: str, record_identity: bool) -> Dict:
        """
        Request formal approval from user with explicit decision and identity.
        
        Flow:
        1. Format and display analysis to user
        2. Formal request: "Do you formally approve this compliance analysis? (Approve / Reject)"
        3. User provides explicit decision: Approve or Reject
        4. If record_identity=true:
           - User enters name and email for formal signature
           - Email domain validated against whitelist
        5. Optional: User provides comments on analysis quality
        6. Return decision dict with approved, approver_name, approver_email, comments
        
        In MVP (Streamlit demo): User clicks "Approve" or "Reject" button,
        enters formal name/email, optionally provides comments.
        
        In production: Integrates with approval workflow that captures formal identity
        (digital signature, single sign-on context, audit system, etc.).
        """
        # Stub: In MVP, this is handled by the Streamlit UI form.
        # Here we return structure that caller expects; actual input comes from formal approval form.
        return {
            'approved': False,  # Placeholder; defaults to False (fail-closed) until formal decision provided
            'approver_name': None,  # Placeholder; actual value from formal approval form
            'approver_email': None,  # Placeholder; actual value from formal approval form
            'comments': ''  # Placeholder; actual value from optional comments field
        }
    
    def _validate_email_domain(self, email: str) -> bool:
        """Check email domain against whitelist."""
        allowed_domains = ['@company.com', '@fca.org.uk']
        return any(email.endswith(domain) for domain in allowed_domains)
```

## Harness Orchestration

### Initialization

Load all data and configurations once at startup:

```python
import yaml

class ComplianceHarness:
    """Main harness orchestrator."""
    
    def __init__(self, config_path: str = "workflows/harness.yaml"):
        """Initialize harness: load YAML, embeddings, configs."""
        self.config = self._load_yaml(config_path)
        
        # Initialize handbook index (pre-load embeddings once)
        handbook_path = self.config['harness']['data_sources']['fca_handbook']['artifact']
        weights_path = self.config['harness']['data_sources']['fca_handbook']['weighting_config']
        initialize_handbook(handbook_path, weights_path)
        
        # Load prompt templates
        self.prompts = {}
        for workflow_name, workflow_config in self.config['harness']['workflows'].items():
            for node in workflow_config['nodes']:
                if node['action'] == 'claude_reasoning':
                    template_path = node['config'].get('Param_prompt_template')
                    if template_path not in self.prompts:
                        with open(template_path) as f:
                            self.prompts[template_path] = f.read()
    
    def _load_yaml(self, path: str) -> Dict:
        """Load and validate YAML workflow configuration."""
        try:
            with open(path) as f:
                return yaml.safe_load(f)
        except Exception as e:
            raise ActionError(f"Failed to load harness config: {str(e)}")
    
    def execute_workflow(self, workflow_name: str, user_input: Dict) -> Dict:
        """
        Execute a workflow end-to-end.
        
        Args:
            workflow_name: Key from harness.workflows (e.g., 'general_enquiry')
            user_input: {'entity_description': str, 'question': str}
        
        Returns:
            Final workflow output (typically ApprovalDecision)
        """
        workflow = self.config['harness']['workflows'].get(workflow_name)
        if not workflow:
            raise ValidationError(f"Unknown workflow: {workflow_name}")
        
        context = dict(user_input)  # Start with user inputs
        
        # Execute nodes sequentially
        for node in workflow['nodes']:
            try:
                # Prepare input from context
                node_input = self._prepare_input(node, context)
                
                # Get action class from registry
                action_class = ACTION_REGISTRY[node['action']]
                
                # Execute action
                node_output = action_class().execute(node_input, node.get('config', {}))
                
                # Validate required fields in output
                self._validate_node_output(node, node_output)
                
                # Store output in context for next node
                context[node['name']] = node_output
                
            except ActionError as e:
                # Log error with audit trail
                self._log_error(node['name'], str(e))
                raise
        
        # Log successful interaction for audit trail and process improvement
        self._log_interaction(context, workflow_name)
        
        return context
    
    def _prepare_input(self, node: Dict, context: Dict) -> Dict:
        """Resolve node inputs from context or user inputs."""
        node_input = {}
        inputs_config = node.get('input', {})
        
        if isinstance(inputs_config, str):
            # Single input reference (e.g., input: entity_description)
            node_input = context.get(inputs_config)
        elif isinstance(inputs_config, dict):
            # Multiple inputs with references (e.g., input: {product_features: EntityFeatures})
            for key, ref in inputs_config.items():
                if ref in context:
                    node_input[key] = context[ref]
                else:
                    raise ValidationError(f"Cannot resolve input {ref} for node {node['name']}")
        
        return node_input
    
    def _validate_node_output(self, node: Dict, node_output: Dict) -> None:
        """
        Validate that node output contains required_fields declared in YAML.
        Raises ValidationError if required fields are missing.
        """
        output_spec = node.get('output', {})
        
        # Handle different output spec formats
        if isinstance(output_spec, dict):
            # Check each output type for required_fields
            for output_type, output_config in output_spec.items():
                if isinstance(output_config, dict) and 'required_fields' in output_config:
                    required_fields = output_config.get('required_fields', [])
                    for field in required_fields:
                        if field not in node_output or node_output[field] is None:
                            raise ValidationError(
                                f"Node '{node['name']}' output missing required field: {field}"
                            )
    
    def _log_error(self, node_name: str, error_msg: str) -> None:
        """Log error with timestamp for audit trail."""
        timestamp = datetime.now(timezone.utc).isoformat() + 'Z'
        print(f"[{timestamp}] ERROR in {node_name}: {error_msg}")
    
    def _log_interaction(self, context: Dict, workflow_name: str) -> None:
        """
        Record interaction for audit trail and process improvement.
        Appends to interactions.json with full workflow state.
        """
        import json
        import uuid
        from pathlib import Path
        
        timestamp = datetime.now(timezone.utc).isoformat() + 'Z'
        
        # Extract relevant fields from context
        entity_features = context.get('extract_features', {})
        ranked_rules = context.get('retrieve_rules', [])
        analysis = context.get('analyze_compliance', {})
        approval = context.get('human_review', {})
        
        # Build rules summary (id, header, rank, similarity)
        rules_summary = []
        for idx, rule in enumerate(ranked_rules):
            rules_summary.append({
                'id': idx + 1,
                'header': rule.get('rule_id', 'unknown'),
                'rank': idx + 1,
                'similarity': rule.get('base_similarity', 0.0)
            })
        
        # Build interaction record
        interaction = {
            'timestamp': timestamp,
            'session_id': str(uuid.uuid4()),
            'workflow': workflow_name,
            'entity_description': context.get('entity_description', ''),
            'question': context.get('question', ''),
            'rules_retrieved': rules_summary,
            'rules_count': len(ranked_rules),
            'claude_reasoning': analysis.get('answer', ''),
            'citations_count': len(analysis.get('citations', [])),
            'approval_decision': 'approved' if approval.get('approved', False) else 'rejected',
            'approver_name': approval.get('approver_name'),
            'approver_comment': approval.get('comments', '')
        }
        
        # Append to interactions.json
        interactions_path = Path('interactions.json')
        interactions = []
        
        if interactions_path.exists():
            try:
                with open(interactions_path) as f:
                    interactions = json.load(f)
            except (json.JSONDecodeError, IOError):
                interactions = []
        
        interactions.append(interaction)
        
        try:
            with open(interactions_path, 'w') as f:
                json.dump(interactions, f, indent=2)
        except IOError as e:
            # Log warning but don't fail workflow
            print(f"[{timestamp}] WARNING: Failed to write interactions.json: {str(e)}")

# Global harness instance
harness = None

def initialize_harness(config_path: str = "workflows/harness.yaml"):
    """Initialize the harness at application startup."""
    global harness
    harness = ComplianceHarness(config_path)

def execute_workflow(workflow_name: str, entity_description: str, question: str) -> Dict:
    """Execute a workflow. Call after initialize_harness()."""
    if harness is None:
        raise ActionError("Harness not initialized. Call initialize_harness() first.")
    return harness.execute_workflow(workflow_name, {
        'entity_description': entity_description,
        'question': question
    })
```

### Tool Integration

**Deployment Model:** This section assumes **Option 1 (Direct Integration)** — the Harness is called directly from Python code that invokes Claude. See "Deployment Models" section above for other patterns.

The Harness appears as a Claude tool. This handler validates scope and invokes the harness:

```python
def handle_tool_call(tool_name: str, tool_input: Dict) -> str:
    """
    Handle tool invocation from Claude.
    
    Args:
        tool_name: Should be 'FCA_Handbook_compliance_harness'
        tool_input: {'entity_description': str, 'question': str}
    
    Returns:
        JSON string with compliance analysis result
    """
    if tool_name != 'FCA_Handbook_compliance_harness':
        return json.dumps({'error': 'Unknown tool'})
    
    entity_description = tool_input.get('entity_description', '')
    question = tool_input.get('question', '')
    
    # Validate scope: FCA Handbook only
    if not _is_fca_scope(question):
        return json.dumps({
            'scope_clarification': 'Do you mean compliance with the **FCA Handbook**? '
                                   'This tool handles FCA Handbook rules specifically.'
        })
    
    try:
        result = execute_workflow('general_enquiry', entity_description, question)
        return json.dumps(result, default=str)
    except ActionError as e:
        return json.dumps({'error': str(e)})

def _is_fca_scope(question: str) -> bool:
    """Check if question is about FCA Handbook."""
    fca_keywords = ['fca', 'handbook', 'cobs', 'prin', 'compliance']
    return any(kw in question.lower() for kw in fca_keywords)
```

## Data Flow Example

For a complete general_enquiry workflow:

```
INPUT:
  entity_description: "# AdviceBot..."
  question: "Which COBS rules apply to algorithmic advice?"

NODE 1 (parse_markdown):
  INPUT: entity_description
  OUTPUT: EntityFeatures
  STORED: context['extract_features']

NODE 2 (semantic_search):
  INPUT: {product_features: context['extract_features'], question}
  OUTPUT: RankedRules (list of rules with similarity + weights)
  STORED: context['retrieve_rules']

NODE 3 (claude_reasoning):
  INPUT: {product_features: context['extract_features'], rules: context['retrieve_rules']}
  OUTPUT: ComplianceAnalysis (answer + citations + reasoning_log)
  STORED: context['analyze_compliance']

NODE 4 (approval_gate):
  INPUT: {analysis: context['analyze_compliance']}
  OUTPUT: ApprovalDecision (approved + approver + timestamp + audit trail)
  STORED: context['human_review']

FINAL OUTPUT: context['human_review'] (ApprovalDecision)
```

## Testing Strategy

Unit test pattern for each action:

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
            {'rule_id': 'COBS 2.1.1R', 'text': 'Rule text', 'base_similarity': 0.9, ...}
        ]
        action = SemanticSearchAction()
        result = action.execute({
            'product_features': {'entity_name': 'Test'},
            'question': 'What rules apply?'
        }, {'Param_top_k': 20})
        self.assertIsInstance(result, list)
        self.assertGreater(len(result), 0)

# Integration test (full workflow)
class TestNewProductReviewWorkflow(unittest.TestCase):
    
    def setUp(self):
        initialize_harness("tests/fixtures/harness.yaml")
    
    def test_full_workflow_execution(self):
        """Test end-to-end execution with fixture data."""
        result = execute_workflow('general_enquiry',
            entity_description="# TestProduct\n## Features\n- Feature1",
            question="What COBS rules apply?"
        )
        self.assertIn('human_review', result)
        self.assertIn('approved', result['human_review'])
```

## Next Steps for Implementers

1. **Implement each action class** with full input validation and error handling per GeneralEnquirySpecs.md
2. **Load handbook data** once at startup (not per-query) for performance
3. **Integrate with Anthropic SDK** for Claude reasoning (use prompt caching for efficiency)
4. **Add audit logging** to every action with timestamps
5. **Write comprehensive unit and integration tests** for each action
6. **Test error paths** thoroughly (missing data sources, invalid inputs, approval timeouts)
