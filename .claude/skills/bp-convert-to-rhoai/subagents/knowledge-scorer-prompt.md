---
description: Score and rank knowledge base files by relevance to blueprint features
---

# Knowledge Base Relevance Scoring

## Your Role

You are scoring knowledge base files to determine which conversion patterns are most relevant to this specific blueprint. This enables the main conversion agent to load only the most applicable patterns, keeping context lean.

Your output will be used to:
1. **Load relevant knowledge** - Top-scored files will be read by the main agent
2. **Avoid noise** - Low-scored files will be skipped to save context
3. **Enable pattern reuse** - Match proven patterns from similar blueprints

## How Scoring Works

Each knowledge file has YAML frontmatter with tags:
- `components`: ["redis", "triton", "milvus", ...]
- `deployment_types`: ["helm", "docker-compose", ...]
- `resource_types`: ["gpu", "storage", ...]
- `architecture`: ["rag-pipeline", "agentic", ...]

Score files by overlap with the blueprint's feature vector. Higher overlap = higher relevance.

---

## Instructions

**Input Parameters:**
- Blueprint features: {blueprint_features}
- Knowledge base directory: {kb_dir}

### 2.1 Scan Knowledge Base

```bash
KB_DIR="{kb_dir}"
find "$KB_DIR" -name "*.md" -not -name "README.md" -type f
```

### 2.2 Extract Frontmatter and Score

For each knowledge file:

```bash
# Extract only frontmatter (efficient - don't read full file yet)
awk 'n==2{exit} /^---$/{n++} n' <knowledge-file.md>
```

Check frontmatter tags against blueprint features:

**Valid values to match:**
- **components**: redis, triton, milvus, qdrant, weaviate, pgvector, postgresql, mysql, mongodb, minio, vllm, tensorrt-llm, llama, mistral, embed-qa, nemo-retriever, memcached, s3, etc.
- **deployment_types**: helm, docker-compose, notebook, oc-apply
- **resource_types**: gpu, storage, networking, security-context
- **architecture**: rag-pipeline, agentic, inference-only, etc.

**Scoring criteria:**
1. **Component match** (highest priority): +10 points per matching component
2. **Deployment type match**: +5 points per matching deployment type
3. **Resource type match**: +3 points per matching resource type
4. **Architecture match**: +5 points if architecture matches

**Example:**
```
Blueprint features:
  components: ["redis", "triton", "milvus"]
  deployment_types: ["helm"]
  resource_types: ["gpu", "storage"]
  architecture: "rag-pipeline"

Knowledge file: components/redis-on-rhoai.md
  Frontmatter:
    components: ["redis"]
    deployment_types: ["helm", "docker-compose"]
    resource_types: ["storage"]

Score calculation:
  Component match (redis): +10
  Deployment match (helm): +5
  Resource match (storage): +3
  Total: 18 points
```

### 2.3 Rank and Return

Sort files by score (highest first). Group by category:
- **Components**: Files in `components/` directory
- **Deployment types**: Files in `deployment-types/` directory
- **Resource patterns**: Files in `resource-patterns/` directory
- **Architectures**: Files in `architectures/` directory
- **Integrations**: Files in `integrations/` directory

Return top-scored files (typically 5-10 most relevant).

---

## Output

Return ranked knowledge files as JSON matching this schema:

```json
{
  "ranked_files": [
    {
      "path": "components/redis-on-rhoai.md",
      "category": "component",
      "score": 18,
      "matched_on": ["redis", "helm", "storage"]
    },
    {
      "path": "components/triton-on-rhoai.md",
      "category": "component",
      "score": 23,
      "matched_on": ["triton", "helm", "gpu", "storage"]
    }
  ]
}
```

**Important:** 
- Return ONLY the JSON
- Include files with score > 0
- Sort by score descending
- Limit to top 15 files maximum
