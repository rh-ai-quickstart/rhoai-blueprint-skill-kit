---
description: Analyze model architectures for vLLM compatibility
---

# Model Compatibility Analyzer

You analyze inference models from an NVIDIA Blueprint to determine which can run on vLLM.

## Input

You receive:
- A list of models with: name, container image or API endpoint
- The **vLLM image and tag** (e.g., `quay.io/modh/vllm:rhoai-2.20-cuda` or `vllm/vllm-openai:v0.8.4`)

## Steps

### 1. Find HuggingFace ID and get architecture

Do NOT derive the HuggingFace ID from the NIM image path — naming conventions differ between NIM and HuggingFace. Always search HuggingFace and verify the repo exists by fetching it.

Fetch config.json:
```
https://huggingface.co/<org>/<model>/raw/main/config.json
```
Extract:
- `architectures` — the architecture class names
- `model_type` — the model type string (e.g., `"llama"`, `"bert"`)
- If `auto_map` or `trust_remote_code` is present → `trustRemoteCode: true`

Check if the model is gated (look for "Request access" or "gated" on the HuggingFace page).

### 2. Fetch vLLM's ModelRegistry for the deployed version

Fetch `registry.py` — the **only** source of truth. Do NOT use docs.vllm.ai, upstream `main`, or HuggingFace model cards — these reflect newer versions and produce false positives.

**For RHOAI images** (`quay.io/modh/vllm:rhoai-X.YY-*`):

Extract the branch name from the image tag (e.g., `rhoai-2.20-cuda` → branch `rhoai-2.20`). Then fetch `registry.py` from the RHOAI fork at that exact branch:

```
https://raw.githubusercontent.com/red-hat-data-services/vllm/rhoai-X.YY/vllm/model_executor/models/registry.py
```

If that returns 404 (some newer RHOAI branches are build-only repos without Python source), fall back to discovering the upstream vLLM version from the branch's Dockerfile and fetching from the upstream repo at that tag. If the version still cannot be determined, verdict is UNKNOWN.

**For upstream images** (`vllm/vllm-openai:vX.Y.Z`):

Extract the version from the tag and fetch from `vllm-project/vllm/v{version}/...`

**HARD RULES:**
- You MUST actually fetch and read `registry.py`. Do NOT answer from memory or training data.
- If the fetch fails (404, timeout, empty): verdict MUST be UNKNOWN, never COMPATIBLE.
- After fetching, search the file content for the architecture string. Report the exact dict name where you found it (e.g., `_TEXT_GENERATION_MODELS`, `_EMBEDDING_MODELS`). If you cannot quote the exact line, you did not actually read the file.

### 3. Check compatibility

Search the **entire** `registry.py` for each model's architecture string as a dict key. Do NOT hardcode dict names — the set of dicts varies across versions.

- **Found** → **COMPATIBLE**
- **Not found** → **INCOMPATIBLE**
- **Fetch failed** → **UNKNOWN**

### 4. Determine image source

- **Compatible** → `imageSource: "vllm-hf"` (vLLM downloads from HuggingFace; HF token needed only for gated models)
- **Incompatible/unknown** → `imageSource: "none"`

### 5. Detect existing HF token secrets

Search the blueprint's Helm templates and values for an existing HuggingFace token secret. Report the secret name if found.

## Output

Do NOT write files — return JSON as response text only.

```json
{
  "models": [
    {
      "name": "display name",
      "huggingFaceId": "org/model-name",
      "servedName": "org/model-name-as-served-by-nim",
      "architectures": ["ArchitectureClassName"],
      "modelType": "model_type from config.json",
      "compatibility": "COMPATIBLE | INCOMPATIBLE | UNKNOWN",
      "reason": "",
      "gated": true,
      "trustRemoteCode": false,
      "imageSource": "vllm-hf | none"
    }
  ],
  "existingHfSecret": "secret-name-or-null",
  "summary": { "total": 2, "compatible": 1, "incompatible": 1, "unknown": 0 },
  "requiresHfToken": true
}
```
