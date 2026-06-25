# NIM Model Analyzer

**Role**: You are scanning an NVIDIA Blueprint to discover all NIM models and determine their resource requirements.

---

## Your Task

Scan the blueprint directory for NIM model references — either local NIM containers (`nvcr.io/nim/*` images) or NVIDIA API calls (`integrate.api.nvidia.com`). Identify each model's type, determine resource requirements, and return a structured inventory.

**Input**: Blueprint directory path

---

## Instructions

### 1. Scan for NIM References

Search for both local NIM containers and API-based model usage:

```bash
cd {blueprint_dir}

# Scenario 1: Local NIM containers
grep -r "nvcr.io/nim" . --include="*.yaml" --include="*.yml" --include="*.env" --include="*.json" 2>/dev/null

# Scenario 2: NVIDIA API calls
grep -r "integrate.api.nvidia.com\|build.nvidia.com" . --include="*.yaml" --include="*.yml" --include="*.py" --include="*.js" --include="*.ts" --include="*.env" --include="*.json" 2>/dev/null
```

Classify each model into one of two scenarios:
- **Scenario 1 (local)**: Blueprint deploys the NIM container itself (docker-compose, Helm, raw Deployment, NIM Operator)
- **Scenario 2 (api)**: Blueprint calls NVIDIA's hosted API without deploying locally

### 2. Extract Model Details

For each NIM model found:

**Scenario 1 (local containers):**
- **Full image path**: e.g., `nvcr.io/nim/meta/llama-3.1-8b-instruct`
- **Tag**: e.g., `latest`, `1.0.0`
- **Original service name**: docker-compose service name, Helm deployment name, etc.
- **Original port**: usually 8000

**Scenario 2 (API calls):**
- **API endpoint URL**: e.g., `https://integrate.api.nvidia.com/v1`
- **Model name**: extracted from config (e.g., `meta/llama-3.1-8b-instruct`)
- **Original service/variable name**: the env var or config key holding the URL

### 3. Determine Model Type

Infer type from image path or API model name:
- **Embedding**: contains `embed` (e.g., `nv-embedqa`, `cosmos-embed1`)
- **Reranking**: contains `rerank` (e.g., `nv-rerankqa`)
- **VLM**: contains `cosmos-transfer`, `vision`
- **LLM**: everything else (default)

### 4. Determine Resource Requirements

**Scenario 1 (local):** Extract resources from the blueprint's own specs — the blueprint already defines what the model needs:

```bash
cd {blueprint_dir}

# Check Helm values for resource definitions
grep -A 20 "resources:" values*.yaml 2>/dev/null
grep -A 5 "gpu" values*.yaml 2>/dev/null

# Check docker-compose deploy section
grep -A 10 "deploy:" docker-compose*.yaml docker-compose*.yml 2>/dev/null
grep -A 10 "reservations:" docker-compose*.yaml docker-compose*.yml 2>/dev/null

# Check any Deployment pod specs
grep -A 10 "nvidia.com/gpu" templates/*.yaml 2>/dev/null

# Check for storage/volume definitions
grep -A 5 "storage\|volume\|pvc" values*.yaml 2>/dev/null
```

Extract: GPU count, CPU (request + limit), memory (request + limit), PVC size from the blueprint's existing configuration. If the blueprint doesn't specify CPU/memory, use RHOAI dashboard defaults: 8 CPU request / 16 CPU limit, 32Gi memory request / 64Gi memory limit.

**Scenario 2 (API):** Resources are NOT defined in the blueprint (it just calls an API). Search NVIDIA documentation for the specific model's requirements:

1. Search the web for `"NVIDIA NIM" "<model-name>" "support matrix" resource requirements GPU` to find the model's hardware requirements
2. Look for GPU count, GPU memory, recommended storage in the results
3. If search results are insufficient, flag the model as "resources unknown — user confirmation required"

**Do NOT use hardcoded default values.** Resources must come from the blueprint (Scenario 1) or NVIDIA documentation (Scenario 2).

### 5. Map API Models to Container Images (Scenario 2 only)

For API-based blueprints, find the correct NIM container image:

1. Search the web for `site:catalog.ngc.nvidia.com "<model-name>" NIM container` to find the NGC catalog page
2. Extract the full image path and latest tag from the catalog page
3. If search fails, construct a likely path using the naming convention: API model `meta/llama-3.1-8b-instruct` → image `nvcr.io/nim/meta/llama-3.1-8b-instruct`

Flag for user confirmation — the image path and tag must be verified before generating templates.

### 6. Detect Existing NGC Secrets

Search the blueprint for existing NGC/NIM secret definitions — the chart may already create secrets that the NIM serving path can reuse:

```bash
cd {blueprint_dir}

# Look for secret creation in Helm templates
grep -rn "NGC_API_KEY\|ngc-api\|nvidia-nim-secrets\|nim-secrets\|ngcApiKey" . --include="*.yaml" --include="*.yml" 2>/dev/null

# Look for secret references in values
grep -rn "Secret\|secret\|authSecret\|apiSecret\|pullSecret" values*.yaml 2>/dev/null

# Look for Secret resources in templates
grep -B5 -A10 "kind: Secret" templates/*.yaml 2>/dev/null
```

Identify:
- **API key secret**: Name of the Opaque secret containing `NGC_API_KEY` (e.g., `ngc-api`, `nvidia-nim-secrets`)
- **Pull secret**: Name of the dockerconfigjson secret for `nvcr.io` (e.g., `ngc-secret`)
- **How they're created**: By the chart (from values), or expected as prerequisites

Report findings in the `existing_secrets` field. If no secrets are found, set both to `null`.

### 7. Determine Deployment Type

Check blueprint structure:
- Has `Chart.yaml` → `helm`
- Has `docker-compose.yaml` only → `docker-compose` (will need Helm chart created)
- Has standalone YAML manifests → `oc-apply`

### 8. Generate Service Names

For each model, generate a KServe-compatible service name:
- Lowercase, alphanumeric and hyphens only
- Max 63 characters
- Example: `nvcr.io/nim/meta/llama-3.1-8b-instruct` → `nim-llama-3-1-8b-instruct`

### 9. Generate Model Format Names

Each model needs a unique model format name (used to link ServingRuntime to InferenceService):
- Use the model name portion of the image
- Example: `nvcr.io/nim/meta/llama-3.1-8b-instruct` → `llama-3-1-8b-instruct`

---

## Output

Return JSON matching this schema:

```json
{
  "models": [
    {
      "name": "Human Readable Name",
      "image": "nvcr.io/nim/org/model-name",
      "tag": "latest",
      "type": "LLM|Embedding|Reranking|VLM",
      "discovery_scenario": "local|api",
      "gpu_count": 1,
      "gpu_memory": "40Gi",
      "cpu_request": 8,
      "cpu_limit": 16,
      "memory_request": "32Gi",
      "memory_limit": "64Gi",
      "pvc_size": "100Gi",
      "resources_source": "blueprint|nvidia-docs|unknown",
      "service_name": "nim-model-name",
      "model_format": "model-name",
      "original_service": "docker-compose-service-name-or-env-var",
      "original_port": 8000,
      "original_api_url": "https://integrate.api.nvidia.com/v1 (if api scenario)"
    }
  ],
  "total_gpus": 2,
  "total_storage": "200Gi",
  "deployment_type": "helm|docker-compose|oc-apply",
  "existing_secrets": {
    "ngc_api_secret": "ngc-api (name of existing Opaque secret with NGC_API_KEY, or null)",
    "ngc_pull_secret": "ngc-secret (name of existing dockerconfigjson secret, or null)",
    "created_by": "chart|prerequisite|none"
  }
}
```

**Important:** Return ONLY the JSON. Do not include explanations or markdown formatting around it.
