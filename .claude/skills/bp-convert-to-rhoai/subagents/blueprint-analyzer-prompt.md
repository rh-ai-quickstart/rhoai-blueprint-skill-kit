---
description: Analyze NVIDIA Blueprint structure and extract feature vector for knowledge retrieval
---

# Blueprint Structure Analysis

## Your Role

You are analyzing an NVIDIA Blueprint to understand its architecture, components, and deployment structure. This analysis is the first step in converting the blueprint to run on Red Hat OpenShift AI (RHOAI).

Your output will be used to:
1. **Select relevant knowledge** - Load only the conversion patterns that match this blueprint's components
2. **Guide conversion decisions** - Understand what RHOAI-specific adaptations are needed (GPU, storage, security contexts, etc.)
3. **Determine deployment strategy** - Choose between Helm charts vs. oc apply manifests

## What to Look For

Focus on identifying:
- **Components**: Infrastructure services (vector DBs, caches, inference servers) that need RHOAI-specific configuration
- **Deployment method**: How the blueprint is currently deployed (Helm, docker-compose, notebooks)
- **Resource requirements**: What OpenShift resources are needed (GPU, persistent storage, networking)

This is a **pure analysis task** - you're extracting facts, not making conversion decisions yet.

---

## Instructions

**Input Parameters:**
- Blueprint directory: {blueprint_dir}

### 1.1 Understand Structure

```bash
cd {blueprint_dir}

# Find deployment artifacts
find . -maxdepth 3 \( -name "docker-compose.yaml" -o -name "docker-compose.yml" -o -name "Chart.yaml" -o -name "*.ipynb" \) | head -10
ls -la
test -f README.md && echo "Has README"
```

**Extract blueprint features:**
- Primary deployment method (Helm, docker-compose, notebooks, other)
- Architecture (RAG pipeline, agentic workflow, inference-only)
- Components/services identified from:
  - docker-compose: `yq '.services | keys' docker-compose.yaml`
  - Helm: `yq '.images | keys' values.yaml`
  - Notebooks: `find . -name "*.ipynb"`
- Resource requirements:
  - GPU: `grep -r "nvidia\|gpu" . --include="*.yaml"`
  - Storage: `grep -r "volumes:" docker-compose.yaml`
  - Networking: `grep -r "ports:" docker-compose.yaml`

### 1.2 Create Feature Vector

**Component types to identify:**
- Inference servers: Triton, vLLM, TensorRT-LLM
- NIM models: llama, mistral, embed-qa, nemo-retriever
- Vector DBs: Milvus, Qdrant, Weaviate, pgvector
- Caches: Redis, Memcached
- Databases: PostgreSQL, MySQL, MongoDB
- Storage: MinIO, S3
- Other services

Build the feature vector:
```python
blueprint_features = {
    'components': [<list-of-identified-components>],
    'architecture': '<inferred-architecture>',
    'deployment_types': [<helm|docker-compose|notebook|oc-apply>],
    'resource_types_needed': [<gpu|storage|networking|security-context>]
}
```

---

## Output

Return the feature vector as JSON matching this schema:

```json
{
  "components": ["string"],
  "architecture": "string",
  "deployment_types": ["string"],
  "resource_types_needed": ["string"]
}
```

**Important:** Return ONLY the JSON. Do not include explanations, summaries, or markdown formatting around the JSON.
