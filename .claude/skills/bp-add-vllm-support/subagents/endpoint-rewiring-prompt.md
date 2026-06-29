---
description: Rewire model endpoints to vLLM companion Service URLs
---

# vLLM Endpoint Rewiring

**Role**: You are finding all references to model endpoints in a blueprint and adding a vLLM conditional branch that points to the vLLM companion Service URL.

## Input

You receive:
- Blueprint directory path
- List of compatible models with their companion Service names and original service names

## Instructions

### 1. Find All Model Endpoint References

Search the entire blueprint for references to model endpoints:

```bash
cd {blueprint_dir}

# Search for original service names
grep -rn "<original-service-name>" . \
--include="*.yaml" --include="*.yml" \
--include="*.py" --include="*.js" \
--include="*.ts" --include="*.env" \
--include="*.cfg" --include="*.json"

# Search for NIM image references
grep -rn "nvcr.io/nim" . \
--include="*.yaml" --include="*.yml"

# Search for NVIDIA API endpoints
grep -rn "integrate.api.nvidia.com\|build.nvidia.com" . \
--include="*.yaml" --include="*.yml" \
--include="*.py" --include="*.ts" \
--include="*.env" --include="*.json"

# Search for port 8000 references near model service names
grep -rn "8000" . \
--include="*.yaml" --include="*.yml" \
--include="*.py" --include="*.env"
```

Common patterns to find:
- Docker-compose service names used as hostnames: `http://nemollm-inference:8000`
- Environment variables pointing to models: `LLM_SERVER_URL=http://nim-llm:8000`
- Helm values with model endpoints: `inferenceUrl: http://nim-service:8000`
- NVIDIA API URLs: `https://integrate.api.nvidia.com/v1`

### 2. Determine vLLM Replacement URLs

**First, detect the chart's namespace pattern.** Search existing templates for how namespace is referenced — charts may use a helper (e.g., `{{ include "wosa.namespace" . }}`) instead of `{{ .Release.Namespace }}`. Check `templates/_helpers.tpl` and existing endpoint URLs in deployment templates. Use whatever pattern the chart already uses.

The vLLM companion Service URL pattern is:
```
{{ .Values.vllm.models.<model-key>.service.name }}.{{ <namespace-pattern> }}.svc.cluster.local:8000
```

Where `<namespace-pattern>` matches the chart's existing convention (e.g., `include "chartname.namespace" .` or `.Release.Namespace`).

The companion Service exposes port 8000 (matching the original NIM port), so applications don't need code changes.

### 3. Add vLLM Branch to Existing Conditionals

Add `vllm` as a **new branch** in existing endpoint conditionals. Do NOT modify or remove existing branches. Use the same namespace pattern as existing branches.

#### In Helm templates (env vars)

```yaml
- name: LLM_SERVER_URL
  {{- if .Values.vllm.models.<model-key>.enabled }}
  value: "http://{{ .Values.vllm.models.<model-key>.service.name }}.{{ <namespace-pattern> }}.svc.cluster.local:8000"
  {{- else if .Values.nimOperator.<model-key>.enabled }}
  value: ...  # existing branch — leave as-is
  {{- else }}
  value: ...  # existing default — leave as-is
  {{- end }}
```

If no conditional exists yet, create one with the vLLM branch first, then the original value as else.

#### In application code / config files

- Do NOT modify application source code directly
- Ensure the URL is configurable via environment variable
- Set the environment variable conditionally in the Helm template

### 4. Patch NetworkPolicy for vLLM

If the blueprint has a NetworkPolicy that restricts egress, add `component: predictor` to the allowed pod selectors. KServe labels all predictor pods with this label — without it, the backend cannot reach the vLLM InferenceService pods.

```bash
grep -rn "NetworkPolicy\|networkPolicy" \
{blueprint_dir} --include="*.yaml" --include="*.yml"
```

In the NetworkPolicy's egress `to:` list, add alongside existing pod selectors:

```yaml
- podSelector:
    matchLabels:
      component: predictor
```

### 5. Preserve API Compatibility

vLLM exposes the same OpenAI-compatible API as NIM:
- `/v1/chat/completions` (LLM)
- `/v1/embeddings` (Embedding)
- `/v1/models` (all)

The application code should work without changes — only the base URL changes.

Verify that:
- No path rewiring is needed (same API paths)
- Port 8000 is used (via companion Service)

**Do NOT disable or wrap existing model containers.** The user controls which path is active via `enabled` flags in values.yaml.

## Output

Report what was rewired:

```json
{
  "rewired_endpoints": [
    {
      "model": "model-name",
      "original_ref": "http://nim-llm:8000",
      "new_ref": "http://{{ .Values.vllm.models.llm.service.name }}.{{ .Release.Namespace }}.svc.cluster.local:8000",
      "files_modified": ["templates/deployment-app.yaml"],
      "type": "env_var"
    }
  ],
  "networkPolicyPatched": true,
  "files_modified": ["path/to/file1", "path/to/file2"]
}
```
