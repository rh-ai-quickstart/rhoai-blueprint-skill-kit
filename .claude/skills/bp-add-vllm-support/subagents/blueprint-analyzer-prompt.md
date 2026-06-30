---
description: Analyze blueprint to find inference models and extract details
---

# Blueprint Analyzer for vLLM

## Your Role

You analyze an NVIDIA Blueprint's Helm chart to find all inference models. This is the first step before checking vLLM compatibility.

## Input

- Blueprint directory: `{blueprint_dir}`

## Instructions

### 1. Verify RHOAI Support

```bash
cd {blueprint_dir}
test -f Chart.yaml || test -f */Chart.yaml
grep -r "openshift" values*.yaml
```

If no Helm chart or no OpenShift support found → return `"rhoaiSupport": false` and stop.

### 2. Find All Inference Models

Scan `values.yaml`, `values-openshift.yaml`, and `templates/` for inference model containers. Look for:

- NIM references: `nimOperator`, `NIMCache`, `NIMService`, `nvcr.io/nim/`
- NVIDIA images: `nvcr.io/` in container specs or values
- API calls: `api.nvidia.com`, `build.nvidia.com`, `NVIDIA_API_KEY` in env vars
- Model-serving containers: anything serving inference (not databases, caches, frontends, or utility services)

### 3. For Each Model, Extract

- **name**: the key or identifier used in the chart (e.g., `llm`, `embedding`, `reranker`)
- **image**: full container image string (e.g., `nvcr.io/nim/meta/llama-3.1-8b-instruct:1.2.0`) or `null` for API calls
- **apiEndpoint**: the API URL if it's an API call model, or `null`
- **serviceName**: the Kubernetes Service name that exposes this model
- **servicePort**: the port the service listens on
- **gpu**: GPU count from `resources.limits.nvidia.com/gpu` or `0` if none

### 4. Discover vLLM Image

Check if the blueprint already has a vLLM section:
1. `values-openshift.yaml` → `vllm.servingRuntime.image.repository` and `tag`
2. `values.yaml` → same path

If found, extract the image and tag. If not found, set both to `null`.

## Output

Return ONLY this JSON:

```json
{
  "rhoaiSupport": true,
  "models": [
    {
      "name": "llm",
      "image": "nvcr.io/nim/meta/llama-3.1-8b-instruct:1.2.0",
      "apiEndpoint": null,
      "serviceName": "nim-llm",
      "servicePort": 8000,
      "gpu": 1
    }
  ],
  "vllmImage": {
    "repository": "quay.io/modh/vllm or null",
    "tag": "rhoai-2.20-cuda or null"
  }
}
```

Do NOT write files — return JSON as response text only.
