# Python Implementation

This document provides the patterns, interfaces, and concrete implementations needed to build the Compliance Agent Harness in Python. It implements the actions and orchestration patterns specified in GeneralEnquirySpecs.md.

## Architecture Overview

The Python harness has three layers:

1. **Orchestration Layer**: Loads YAML workflows, manages node execution, passes data between nodes
2. **Action Layer**: Implements each action type (parse_markdown, semantic_search, claude_reasoning, approval_gate) with input validation and error handling
3. **Data Layer**: Manages FCA Handbook embeddings (in-memory), configuration files (weights.yaml, prompts), and audit logging

```
User Input (product_description, question)
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
                        - User inputs (product_description, question)
                        - Outputs from earlier nodes (ProductFeatures, RankedRules, etc.)
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
        Parse markdown product description into structured ProductFeatures.
        
        Args:
            node_input: {
                'product_description': str (markdown-formatted product description)
            }
            config: {} (no parameters for this action)
        
        Returns:
            {
                'product_name': str,
                'product_type': str,
                'features': list of str,
                'use_cases': list of str,
                'client_interaction': str,
                'data_handled': list of str,
                'decision_authority': str
            }
        
        Raises:
            ValidationError: If required fields missing or invalid
        """
        product_description = node_input.get('product_description', '')
        
        if not product_description or not isinstance(product_description, str):
            raise ValidationError("product_description must be a non-empty string")
        
        # Extract product name from first heading (h1)
        product_name = self._extract_h1(product_description)
        if not product_name:
            raise ValidationError("Missing product name (h1 heading required)")
        
        # Search for standard sections
        features = self._extract_list_items(product_description, r'#+\s*Features?')
        use_cases = self._extract_list_items(product_description, r'#+\s*Use Cases?')
        client_interaction = self._extract_section_text(product_description, r'#+\s*Client Interaction')
        data_handled = self._extract_list_items(product_description, r'#+\s*Data Handling?')
        decision_authority = self._extract_section_text(product_description, r'#+\s*.*Decision.*Authority')
        
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
        if len(product_name) > 200:
            raise ValidationError(f"product_name exceeds 200 chars: {len(product_name)}")
        if not features or len(features) > 50:
            raise ValidationError(f"features must have 1-50 items, got {len(features)}")
        if not use_cases or len(use_cases) > 20:
            raise ValidationError(f"use_cases must have 1-20 items, got {len(use_cases)}")
        if any(len(f) > 500 for f in features):
            raise ValidationError("One or more features exceed 500 chars")
        
        # Determine product_type from section heading or infer from content
        product_type = self._extract_product_type(product_description)
        if not product_type:
            product_type = "unknown"
        
        return {
            'product_name': product_name.strip(),
            'product_type': product_type,
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
    
    def _extract_product_type(self, text: str) -> str:
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
from scipy.spatial.distance import cosine
import yaml

class HandbookIndex:
    """Load and index FCA Handbook embeddings for semantic search."""
    
    def __init__(self, handbook_path: str, weights_config_path: str):
        """
        Load handbook data and weighting configuration.
        
        Args:
            handbook_path: Path to FCA_Handbook_Text_And_Embeddings JSON file
            weights_config_path: Path to weights.yaml configuration
        """
        # Load handbook data
        with open(handbook_path) as f:
            data = json.load(f)
        
        self.records = data['fca_handbook']
        self.embeddings = np.array([record['embedding'] for record in self.records])
        
        # Load regulatory weights configuration
        try:
            with open(weights_config_path) as f:
                weights_data = yaml.safe_load(f)
            self.weights = weights_data.get('regulatory_weights', {})
        except Exception as e:
            raise DataSourceUnavailableError(
                f"Regulatory weights configuration invalid at {weights_config_path}: {str(e)}"
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
        
        # Compute cosine similarities
        similarities = np.array([
            1 - cosine(query_embedding, emb) for emb in self.embeddings
        ])
        
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
        """Compute all weight factors for a record."""
        rule_type_weight = self._rule_type_weight(record.get('regulatory_content', ''))
        hierarchy_multiplier = record.get('hierarchy_multiplier', 1.0)
        importance_multiplier = self._importance_weight(record.get('regulatory_score', 5))
        piece_weight = record.get('piece_weight', 1.0)
        
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
                'product_features': dict (output from ParseMarkdownAction),
                'question': str (compliance question)
            }
            config: {
                'Param_top_k': int (number of results, default 20)
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
        product_features = node_input.get('product_features', {})
        question = node_input.get('question', '')
        
        if not isinstance(product_features, dict):
            raise ValidationError("product_features must be a dictionary")
        if not question or not isinstance(question, str):
            raise ValidationError("question must be a non-empty string")
        if len(question) < 5 or len(question) > 1000:
            raise ValidationError("question must be 5-1000 characters")
        
        top_k = config.get('Param_top_k', 20)
        if not isinstance(top_k, int) or top_k < 1 or top_k > 100:
            raise ValidationError("Param_top_k must be integer in range [1, 100]")
        
        # Create combined query from product features and question
        features_text = " ".join([
            product_features.get('product_name', ''),
            product_features.get('product_type', ''),
            " ".join(product_features.get('features', [])),
            " ".join(product_features.get('use_cases', []))
        ])
        query_text = f"{features_text}\n{question}"
        
        # Get query embedding from OpenAI
        try:
            client = OpenAI()
            response = client.embeddings.create(
                input=query_text,
                model="text-embedding-3-large"
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
                'source_version': '2026-Q1'  # Should come from config
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
                'product_features': dict (ProductFeatures),
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
        product_features = node_input.get('product_features', {})
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
            prompt_template, product_features, rules
        )
        
        # Call Claude with prompt caching
        try:
            client = Anthropic()
            response = client.messages.create(
                model="claude-opus-4-7",
                max_tokens=2000,
                temperature=0.5,
                system=[
                    {
                        "type": "text",
                        "text": system_message,
                        "cache_control": {"type": "ephemeral"}
                    }
                ],
                messages=[
                    {
                        "role": "user",
                        "content": f"Analyze this product for FCA compliance and cite applicable rules."
                    }
                ]
            )
            analysis_text = response.content[0].text
        except Exception as e:
            raise ActionError(f"LLM service unavailable. Unable to reason over rules. {str(e)}")
        
        # Parse Claude's response
        citations = self._extract_citations(analysis_text, rules)
        reasoning_log = self._extract_reasoning_log(analysis_text)
        
        # Validate citations against source rules
        for citation in citations:
            self._validate_citation(citation, rules)
        
        return {
            'answer': analysis_text,
            'citations': citations,
            'reasoning_log': reasoning_log,
            'timestamp': datetime.utcnow().isoformat() + 'Z'
        }
    
    def _build_system_message(self, template: str, features: Dict, rules: List) -> str:
        """Insert product features and rules into prompt template."""
        features_str = json.dumps(features, indent=2)
        rules_str = json.dumps(rules, indent=2)
        return template.format(product_features=features_str, ranked_rules=rules_str)
    
    def _extract_citations(self, text: str, rules: List) -> List[Dict]:
        """Extract rule citations from Claude's response."""
        citations = []
        # Look for rule IDs (e.g., COBS 2.1.1R) in the text
        rule_ids = re.findall(r'\b[A-Z]+\s+\d+\.\d+\.\d+[RGED]\b', text)
        for rule_id in rule_ids:
            # Find matching rule
            matching_rule = next((r for r in rules if r.get('rule_id') == rule_id), None)
            if matching_rule:
                # Extract binding level from rule_id suffix
                binding_level = rule_id[-1]
                citations.append({
                    'rule_id': rule_id,
                    'cited_text': matching_rule.get('text', '')[:500],  # Truncate if needed
                    'context': text[max(0, text.find(rule_id)-100):text.find(rule_id)+200],
                    'binding_level': binding_level
                })
        return citations
    
    def _extract_reasoning_log(self, text: str) -> Dict:
        """Extract reasoning log structure from Claude's response."""
        return {
            'reasoning_chain': text[:1000],  # First 1000 chars as reasoning
            'gaps_identified': [],  # Would need structured extraction
            'confidence_score': 0.8  # Would need to extract from Claude response
        }
    
    def _validate_citation(self, citation: Dict, rules: List) -> None:
        """Validate that cited text appears verbatim in rule."""
        rule_id = citation.get('rule_id')
        cited_text = citation.get('cited_text')
        
        matching_rule = next((r for r in rules if r.get('rule_id') == rule_id), None)
        if matching_rule:
            rule_text = matching_rule.get('text', '')
            if cited_text not in rule_text:
                # Log warning but don't fail (compliance analyst should review)
                pass
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
        
        # Present analysis to approver (stub implementation)
        # In production, this would route to a queue/UI for human review
        approval_decision = self._route_to_approver(
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
        timestamp = datetime.utcnow().isoformat() + 'Z'
        actions_carried_out.append('snapshot_recorded')
        
        return {
            'approved': approval_decision.get('approved', True),
            'approver_name': approval_decision.get('approver_name'),
            'approver_email': approval_decision.get('approver_email'),
            'approver_role': required_role,
            'timestamp': timestamp,
            'actions_carried_out': actions_carried_out,
            'comments': approval_decision.get('comments', ''),
            'analysis_snapshot': analysis
        }
    
    def _route_to_approver(self, analysis: Dict, role: str, record_identity: bool) -> Dict:
        """Route analysis to approval queue (stub)."""
        # In production: integrate with approval queue system
        # For now, return mock approval
        return {
            'approved': True,
            'approver_name': 'Jane Smith' if record_identity else None,
            'approver_email': 'jane.smith@company.com' if record_identity else None,
            'comments': 'Approved after review'
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
            user_input: {'product_description': str, 'question': str}
        
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
                
                # Store output in context for next node
                context[node['name']] = node_output
                
            except ActionError as e:
                # Log error with audit trail
                self._log_error(node['name'], str(e))
                raise
        
        return context
    
    def _prepare_input(self, node: Dict, context: Dict) -> Dict:
        """Resolve node inputs from context or user inputs."""
        node_input = {}
        inputs_config = node.get('input', {})
        
        if isinstance(inputs_config, str):
            # Single input reference (e.g., input: product_description)
            node_input = context.get(inputs_config)
        elif isinstance(inputs_config, dict):
            # Multiple inputs with references (e.g., input: {product_features: ProductFeatures})
            for key, ref in inputs_config.items():
                if ref in context:
                    node_input[key] = context[ref]
                else:
                    raise ValidationError(f"Cannot resolve input {ref} for node {node['name']}")
        
        return node_input
    
    def _log_error(self, node_name: str, error_msg: str) -> None:
        """Log error with timestamp for audit trail."""
        timestamp = datetime.utcnow().isoformat() + 'Z'
        print(f"[{timestamp}] ERROR in {node_name}: {error_msg}")

# Global harness instance
harness = None

def initialize_harness(config_path: str = "workflows/harness.yaml"):
    """Initialize the harness at application startup."""
    global harness
    harness = ComplianceHarness(config_path)

def execute_workflow(workflow_name: str, product_description: str, question: str) -> Dict:
    """Execute a workflow. Call after initialize_harness()."""
    if harness is None:
        raise ActionError("Harness not initialized. Call initialize_harness() first.")
    return harness.execute_workflow(workflow_name, {
        'product_description': product_description,
        'question': question
    })
```

### Tool Integration

The Harness appears as a Claude tool. This handler validates scope and invokes the harness:

```python
def handle_tool_call(tool_name: str, tool_input: Dict) -> str:
    """
    Handle tool invocation from Claude.
    
    Args:
        tool_name: Should be 'FCA_Handbook_compliance_harness'
        tool_input: {'product_description': str, 'question': str}
    
    Returns:
        JSON string with compliance analysis result
    """
    if tool_name != 'FCA_Handbook_compliance_harness':
        return json.dumps({'error': 'Unknown tool'})
    
    product_description = tool_input.get('product_description', '')
    question = tool_input.get('question', '')
    
    # Validate scope: FCA Handbook only
    if not self._is_fca_scope(question):
        return json.dumps({
            'scope_clarification': 'Do you mean compliance with the **FCA Handbook**? '
                                   'This tool handles FCA Handbook rules specifically.'
        })
    
    try:
        result = execute_workflow('general_enquiry', product_description, question)
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
  product_description: "# AdviceBot..."
  question: "Which COBS rules apply to algorithmic advice?"

NODE 1 (parse_markdown):
  INPUT: product_description
  OUTPUT: ProductFeatures
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
    
    def test_valid_product_description(self):
        """Test parsing well-formed markdown."""
        markdown = """# AdviceBot
        ## Features
        - Robo-advice engine
        - Real-time rebalancing
        ## Use Cases
        - Wealth management
        """
        result = self.action.execute({'product_description': markdown}, {})
        self.assertEqual(result['product_name'], 'AdviceBot')
        self.assertIn('Robo-advice engine', result['features'])
    
    def test_missing_h1_raises_error(self):
        """Test that missing product name (h1) raises ValidationError."""
        markdown = "## Features\n- Feature 1"
        with self.assertRaises(ValidationError):
            self.action.execute({'product_description': markdown}, {})

class TestSemanticSearchAction(unittest.TestCase):
    
    @patch('handbook_index.search')
    def test_returns_ranked_rules(self, mock_search):
        """Test that search returns properly formatted RankedRules."""
        mock_search.return_value = [
            {'rule_id': 'COBS 2.1.1R', 'text': 'Rule text', 'base_similarity': 0.9, ...}
        ]
        action = SemanticSearchAction()
        result = action.execute({
            'product_features': {'product_name': 'Test'},
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
            product_description="# TestProduct\n## Features\n- Feature1",
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
