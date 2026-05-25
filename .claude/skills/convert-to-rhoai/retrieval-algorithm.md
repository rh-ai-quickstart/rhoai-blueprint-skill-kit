# Knowledge Retrieval Algorithm

This document defines how the `convert-to-rhoai` skill retrieves relevant knowledge from the knowledge base.

## Overview

The retrieval algorithm uses **tag-based matching with weighted scoring** to find knowledge files relevant to a specific blueprint conversion.

**Key principle**: Don't load all knowledge upfront - only retrieve what's relevant to avoid context overflow.

## Indexing Dimensions

Knowledge files are tagged across four dimensions:

### Dimension A: Components
Services/tools the pattern applies to (highest weight - direct applicability)
- Examples: `redis`, `triton`, `milvus`, `postgresql`, `llama-server`

### Dimension B: Architecture  
Architecture patterns (medium weight)
- Examples: `rag-pipeline`, `agentic-workflow`, `inference-only`

### Dimension C: Deployment Types
Deployment methods (medium weight)
- Examples: `helm`, `docker-compose`, `oc-apply`, `notebook`

### Dimension D: Resource Types
RHOAI concerns addressed (lower weight - contextual)
- Examples: `gpu`, `storage`, `networking`, `security-context`

## Retrieval Process

### Phase 1: Blueprint Analysis

Extract features from the blueprint being converted:

```python
blueprint_features = {
    'components': ['triton', 'milvus', 'redis', ...],
    'architecture': 'rag-pipeline',
    'deployment_types': ['docker-compose'],
    'resource_types_needed': ['gpu', 'storage', 'networking', 'security-context']
}
```

**How to extract:**
- **Components**: From docker-compose services, Helm values, notebooks
- **Architecture**: Infer from component combinations and data flow
- **Deployment types**: What files exist (docker-compose.yaml, Chart.yaml, *.ipynb)
- **Resource types**: From GPU requirements, volumes, networking needs

### Phase 2: Knowledge File Discovery

Scan knowledge base directories for knowledge files:

```bash
find knowledge-base -name "*.md" -not -name "README.md"
```

For each knowledge file, read **frontmatter only** (not full content yet):

```bash
# Extract just the frontmatter
sed -n '/^---$/,/^---$/p' <knowledge-file>
```

### Phase 3: Relevance Scoring

Score each knowledge file based on tag overlap:

```python
def score_knowledge_file(knowledge_file, blueprint_features):
    score = 0
    
    # Component overlap (highest weight - direct applicability)
    component_matches = set(knowledge_file.components) & set(blueprint_features.components)
    score += len(component_matches) * 10
    
    # Architecture match (medium weight)
    architecture_matches = set(knowledge_file.architecture) & set([blueprint_features.architecture])
    score += len(architecture_matches) * 5
    
    # Deployment type overlap (medium weight)
    deployment_matches = set(knowledge_file.deployment_types) & set(blueprint_features.deployment_types)
    score += len(deployment_matches) * 4
    
    # Resource type relevance (lower weight - contextual)
    resource_matches = set(knowledge_file.resource_types) & set(blueprint_features.resource_types_needed)
    score += len(resource_matches) * 2
    
    return score
```

**Weights rationale:**
- **Component (10x)**: Direct pattern reuse - highest value
- **Architecture (5x)**: Context for how components work together
- **Deployment type (4x)**: Implementation approach matters
- **Resource type (2x)**: Cross-cutting concerns, lower weight

### Phase 4: Selective Loading

Sort knowledge files by score (descending) and load selectively:

```python
# Get all scored knowledge files
scored_knowledge = [
    (knowledge_file, score_knowledge_file(knowledge_file, blueprint_features))
    for knowledge_file in all_knowledge_files
]

# Sort by score
scored_knowledge.sort(key=lambda x: x[1], reverse=True)

# Filter to only relevant (score > 0)
relevant_knowledge = [(kf, score) for kf, score in scored_knowledge if score > 0]

# Load top-scored knowledge
top_knowledge = relevant_knowledge[:10]  # Limit to top 10

# Read full content only for top-scored files
for knowledge_file, score in top_knowledge:
    content = read_file(knowledge_file.path)
    load_into_context(content)
```

**Limits:**
- Maximum 10 knowledge files loaded initially
- Only files with score > 0 (at least one tag match)
- Full content loaded only for top-scored files

### Phase 5: On-Demand Retrieval

During reasoning, if specific questions arise:

**Example:**
```
Agent: "I need more detail about Redis PVC configuration"
  ↓ Check if redis-on-rhoai.md already loaded → Yes
  ↓ Re-read specific section if needed
  
Agent: "This blueprint uses Kafka - haven't seen that before"
  ↓ Check if kafka-on-rhoai.md exists in knowledge base → Yes
  ↓ Score it: components=['kafka'] = 10 points
  ↓ Load kafka-on-rhoai.md content
  ↓ Apply pattern
```

## Retrieval Examples

### Example 1: RAG Pipeline with Triton + Milvus + Redis

**Blueprint features:**
```python
{
    'components': ['triton', 'milvus', 'redis'],
    'architecture': 'rag-pipeline',
    'deployment_types': ['docker-compose'],
    'resource_types_needed': ['gpu', 'storage', 'networking', 'security-context']
}
```

**Scoring:**

| Knowledge File | Components | Arch | Deploy | Resource | Total Score |
|---|---|---|---|---|---|
| `triton-on-rhoai.md` | 10 (triton) | 5 (rag) | 4 (docker-compose) | 4 (gpu,storage) | **23** |
| `milvus-on-rhoai.md` | 10 (milvus) | 5 (rag) | 4 (docker-compose) | 4 (storage,networking) | **23** |
| `redis-on-rhoai.md` | 10 (redis) | 0 | 4 (docker-compose) | 2 (storage) | **16** |
| `rag-pipeline-pattern.md` | 0 | 5 (rag) | 4 (docker-compose) | 0 | **9** |
| `triton-milvus-integration.md` | 20 (both) | 5 (rag) | 0 | 2 (networking) | **27** ← **Top** |
| `gpu-allocation-openshift.md` | 0 | 0 | 0 | 2 (gpu) | **2** |
| `postgresql-on-rhoai.md` | 0 | 0 | 0 | 0 | **0** (excluded) |

**Top 10 loaded:**
1. triton-milvus-integration.md (27)
2. triton-on-rhoai.md (23)
3. milvus-on-rhoai.md (23)
4. redis-on-rhoai.md (16)
5. rag-pipeline-pattern.md (9)
6. gpu-allocation-openshift.md (2)

PostgreSQL knowledge excluded (score=0, no relevance).

### Example 2: Simple Notebook Deployment

**Blueprint features:**
```python
{
    'components': ['jupyter'],
    'architecture': 'notebook-inference',
    'deployment_types': ['notebook', 'oc-apply'],
    'resource_types_needed': ['storage']
}
```

**Scoring:**

| Knowledge File | Components | Arch | Deploy | Resource | Total Score |
|---|---|---|---|---|---|
| `notebook-adaptation.md` | 0 | 0 | 8 (notebook,oc-apply) | 0 | **8** |
| `storage-pvc-patterns.md` | 0 | 0 | 0 | 2 (storage) | **2** |
| `triton-on-rhoai.md` | 0 | 0 | 0 | 0 | **0** (excluded) |

**Top files loaded:**
1. notebook-adaptation.md (8)
2. storage-pvc-patterns.md (2)

Much smaller knowledge set - only what's relevant.

## GitHub Source Retrieval

When knowledge file lacks detail, retrieve from source:

```python
# Knowledge file says: "Use anyuid SCC" but doesn't show exact YAML
knowledge_file = "triton-on-rhoai.md"
source_examples = knowledge_file.frontmatter.source_examples

for example in source_examples:
    if example.approach == "A":  # Relevant approach
        repo_url = example.repo
        # Clone or fetch repo
        repo_path = clone_repo(repo_url)
        # Find relevant files (e.g., Helm templates for Triton)
        triton_deployment = find_file(repo_path, "templates/services/triton*.yaml")
        # Read actual implementation
        actual_yaml = read_file(triton_deployment)
        # Extract SecurityContext configuration
        security_context = extract_security_context(actual_yaml)
        # Apply to new blueprint
        apply_pattern(security_context)
```

**When to trigger:**
- Knowledge file summary insufficient
- Need exact YAML/configuration
- Edge case not documented in knowledge
- Multiple approaches and need to see actual implementation

## Implementation Notes

### Efficient Frontmatter Reading

Don't read full files just to check tags:

```bash
# Fast: Read only frontmatter (first ~20 lines)
head -n 20 knowledge-file.md | sed -n '/^---$/,/^---$/p'

# Slow: Read entire file (may be hundreds of lines)
cat knowledge-file.md
```

### Caching

Frontmatter can be cached across skill invocations:
- Knowledge base doesn't change mid-conversion
- Parse frontmatter once, reuse scores

### Partial Loading

If context is tight, load knowledge incrementally:
1. Load top 3 component files
2. Apply patterns
3. If stuck, load next 3 files
4. Repeat as needed

## Debugging Retrieval

If wrong knowledge is retrieved:

**Check blueprint feature extraction:**
- Are components correctly identified?
- Is architecture inferred correctly?
- Are deployment types accurate?

**Check knowledge file tags:**
- Are frontmatter tags accurate?
- Do tags reflect what knowledge actually covers?
- Are there missing tags causing low scores?

**Check scoring weights:**
- Should different weights be used for this case?
- Is a particular dimension under/over-weighted?

**Manual override:**
- If algorithm fails, manually specify knowledge files to load
- Document why automatic retrieval didn't work
- Consider updating tags or algorithm
