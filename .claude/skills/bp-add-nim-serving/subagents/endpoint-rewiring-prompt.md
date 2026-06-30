# NIM Endpoint Rewiring

**Role**: You are finding all references to NIM containers in a blueprint and rewiring them to use RHOAI InferenceService endpoints.

---

## Your Task

Find every place the blueprint references NIM model endpoints (service names, URLs, ports) and add a new conditional branch for the NIM serving InferenceService URL. Keep all existing branches unchanged.

**Context you'll receive:**
- Blueprint directory path
- Deployment type (helm | oc-apply)
- Models list with original service names and generated InferenceService names
- Generated InferenceService names

---

## Instructions

### 1. Find All NIM References

Search the entire blueprint for references to NIM — both local containers and NVIDIA API endpoints:

```bash
cd {blueprint_dir}

# Search for original service names (from docker-compose)
grep -rn "<original-service-name>" . --include="*.yaml" --include="*.yml" --include="*.py" --include="*.js" --include="*.ts" --include="*.env" --include="*.cfg" --include="*.ini" --include="*.json"

# Search for NIM image references
grep -rn "nvcr.io/nim" . --include="*.yaml" --include="*.yml"

# Search for NVIDIA API endpoints (Scenario 2)
grep -rn "integrate.api.nvidia.com\|build.nvidia.com" . --include="*.yaml" --include="*.yml" --include="*.py" --include="*.js" --include="*.ts" --include="*.env" --include="*.json"

# Search for port 8000 references near NIM service names
grep -rn "8000" . --include="*.yaml" --include="*.yml" --include="*.py" --include="*.env"
```

Common patterns to find:
- Docker-compose service names used as hostnames: `http://nemollm-inference:8000`
- Environment variables pointing to NIM: `LLM_SERVER_URL=http://nim-llm:8000`
- Helm values with NIM endpoints: `inferenceUrl: http://nim-service:8000`
- NVIDIA API URLs: `https://integrate.api.nvidia.com/v1`
- Application config files with hardcoded NIM URLs or API endpoints

### 2. Determine Replacement URLs

For each NIM model, the InferenceService internal URL pattern is:
```
http://<isvc-name>-predictor.{{ .Release.Namespace }}.svc.cluster.local:{{ .Values.nimServing.<model-key>.service.port }}
```

**Port is required.** KServe creates a headless service (`ClusterIP: None`) for the predictor — port remapping (80→8000) does not work on headless services. The client must connect on the container port (default 8000) explicitly. Use the configurable `service.port` value.

### 3. Add NIM Serving Branch to Existing Conditionals

Add `nimServing` as a **new branch** in existing endpoint conditionals. Do not modify or remove existing branches.

#### In Helm templates (env vars)
```yaml
- name: LLM_SERVER_URL
  {{- if .Values.nimServing.<model-key>.enabled }}
  value: "http://{{ .Values.nimServing.<model-key>.service.name }}-predictor.{{ .Release.Namespace }}.svc.cluster.local:{{ .Values.nimServing.<model-key>.service.port }}"
  {{- else if .Values.nimOperator.<model-key>.enabled }}
  value: ...  # existing branch — leave as-is
  {{- else }}
  value: ...  # existing default — leave as-is
  {{- end }}
```

If no conditional exists yet, create one with the NIM serving branch first, then the original value as else.

#### In application code / config files
- Do NOT modify application source code directly
- Ensure the URL is configurable via environment variable
- Set the environment variable conditionally in the Helm template

**Do NOT disable or wrap existing NIM containers.** The user controls which path is active via `enabled` flags in values.yaml.

### 4. Rewire NVIDIA API URLs (Scenario 2)

For blueprints that call NVIDIA's hosted API instead of local containers:

```bash
# Find API base URLs
grep -rn "integrate.api.nvidia.com" . --include="*.yaml" --include="*.yml" --include="*.py" --include="*.env" --include="*.json"
```

Replace the NVIDIA API base URL with the InferenceService internal URL:
- `https://integrate.api.nvidia.com/v1` → `http://<isvc-name>-predictor.{{ .Release.Namespace }}.svc.cluster.local:{{ .Values.nimServing.<model-key>.service.port }}/v1`

**Authentication: do NOT modify API key env vars.** Blueprints typically populate API keys (NVIDIA_API_KEY, EMBEDDING_API_KEY, etc.) from a shared secret that already contains the NGC key. Self-hosted NIMs accept but ignore auth headers, so leaving the original secret references in place is harmless and avoids breaking the `Authorization: Bearer` header (an empty value causes HTTP errors).

### 5. Patch NetworkPolicy for NIM Serving

If the blueprint has a NetworkPolicy that restricts egress, add `component: predictor` to the allowed pod selectors. KServe labels all predictor pods with this label — without it, the backend cannot reach the NIM InferenceService pods.

```bash
grep -rn "NetworkPolicy\|networkPolicy\|networkpolicy" {blueprint_dir} --include="*.yaml" --include="*.yml"
```

In the NetworkPolicy's egress `to:` list, add alongside existing pod selectors:

```yaml
- podSelector:
    matchLabels:
      component: predictor
```

If the NetworkPolicy is conditional (e.g., `{{- if .Values.networkPolicy.enabled }}`), add the predictor selector inside the same conditional block, alongside existing selectors.

### 6. Preserve API Compatibility

NIM InferenceService exposes the same OpenAI-compatible API:
- `/v1/chat/completions` (LLM)
- `/v1/embeddings` (Embedding)
- `/v1/models` (all)
- `/v1/health/ready` (health)

The application code should work without changes — only the base URL changes.

Verify that:
- No path rewiring is needed (same API paths)
- Port is explicit in the URL (`service.port`, default 8000) — headless service does not remap ports
- Authentication headers are handled (removed for local, kept for API)

---

## Output

Report what was rewired:

```json
{
  "rewired_endpoints": [
    {
      "model": "model-name",
      "original_ref": "http://nemollm-inference:8000",
      "new_ref": "http://nim-llama-3-1-8b-predictor.{{ .Release.Namespace }}.svc.cluster.local:{{ .Values.nimServing.llm.service.port }}",
      "files_modified": ["templates/deployment-app.yaml", "values.yaml"],
      "type": "env_var|url|config"
    }
  ],
  "files_modified": ["path/to/file1", "path/to/file2"]
}
```
