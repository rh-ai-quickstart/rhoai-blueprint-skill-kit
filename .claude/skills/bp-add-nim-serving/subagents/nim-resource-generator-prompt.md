# NIM Resource Generator

**Role**: You are generating NIM serving resources (ServingRuntime + InferenceService + PVC) for each NIM model in the blueprint, following verified patterns from the RHOAI NIM integration.

---

## Your Task

For each NIM model provided, generate Helm templates (or standalone YAML) for:
1. **ServingRuntime** — with all 9 volume mounts, NIM annotations, secret references
2. **InferenceService** — RawDeployment mode, GPU resources, tolerations
3. **PVC** — model cache storage

**Context you'll receive:**
- Blueprint directory path
- Deployment type (helm | oc-apply)
- List of NIM models with image, GPU, PVC, service name details
- User decisions (storage class, externalRoute, tolerations)
- Existing secrets detected in the blueprint (ngc_api_secret, ngc_pull_secret)
- Knowledge base directory path

---

## Instructions

### 1. Read Knowledge Base

Read these files before generating:
```
.claude/skills/bp-add-nim-serving/knowledge-base/nim-serving-patterns.md
.claude/skills/bp-add-nim-serving/knowledge-base/nim-prerequisites.md
```

### 2. Generate Resources Per Model

For **each model** in the models list, create 3 resources following the exact patterns from `nim-serving-patterns.md`.

#### ServingRuntime Requirements (CRITICAL)

Every ServingRuntime MUST include:

**Annotations:**
```yaml
annotations:
  opendatahub.io/apiProtocol: REST
  opendatahub.io/recommended-accelerators: '["nvidia.com/gpu"]'
  opendatahub.io/template-display-name: NVIDIA NIM
  opendatahub.io/template-name: nvidia-nim-runtime
  openshift.io/display-name: <model-display-name>
  runtimes.opendatahub.io/nvidia-nim: "true"
```

**Labels:**
```yaml
labels:
  opendatahub.io/dashboard: "true"
```

**All 9 volume mounts** from `nim-serving-patterns.md` (missing any causes container crash).

**Environment variables:**
- `NIM_CACHE_PATH=/mnt/models/cache`
- `NGC_API_KEY` from secretKeyRef (name from `.Values.nimServing.secrets.ngcApiSecret`)

**Other:**
- `imagePullSecrets` referencing `.Values.nimServing.secrets.ngcPullSecret`
- `multiModel: false`
- `protocolVersions: [grpc-v2, v2]`
- `supportedModelFormats` with model-specific name, `autoSelect: true`, `priority: 1`
- Container port from model's `original_port` (default 8000/TCP)

#### InferenceService Requirements

- `serving.kserve.io/deploymentMode: RawDeployment` annotation
- `opendatahub.io/dashboard: "true"` label
- `networking.kserve.io/visibility` label — only include with value `"exposed"` when `externalRoute: true`; **omit the label entirely** when false (do NOT set to "cluster-local")
- `automountServiceAccountToken: false`
- `securityContext` block — configurable via values (e.g., `fsGroup` for OpenShift PVC permissions). Use `{{- with $nim.securityContext }}` so it's only rendered when set. Default to empty `{}` in base values; OpenShift overlays set `fsGroup` as needed.
- `modelFormat.name` MUST match ServingRuntime's `supportedModelFormats[0].name`
- `runtime` references the ServingRuntime name
- GPU requests MUST equal limits (Guaranteed QoS)
- `HOME=/.cache` and `TRITON_CACHE_DIR=/.cache/triton` env vars
- Tolerations for GPU nodes (on InferenceService, not ServingRuntime)

#### PVC Requirements

- Named `<service-name>-cache`
- `ReadWriteOnce` access mode
- `opendatahub.io/managed: "true"` label
- Configurable storageClass (empty = cluster default)

### 3. Generate Fallback Secrets Template (when blueprint doesn't create secrets)

If the model analyzer reported `existing_secrets.created_by` as `"none"` (blueprint doesn't create NGC secrets), generate a secrets template so the user can pass their personal NGC API key via `--set ngcApiKey=nvapi-xxx`:

**File:** `templates/nim-secrets.yaml`

```yaml
{{- if and .Values.ngcApiKey (not (lookup "v1" "Secret" .Release.Namespace .Values.nimServing.secrets.ngcApiSecret)) }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.nimServing.secrets.ngcApiSecret }}
  labels:
    opendatahub.io/managed: "true"
type: Opaque
data:
  NGC_API_KEY: {{ .Values.ngcApiKey | b64enc | quote }}
---
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.nimServing.secrets.ngcPullSecret }}
  labels:
    opendatahub.io/managed: "true"
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ printf "{\"auths\":{\"nvcr.io\":{\"username\":\"$oauthtoken\",\"password\":\"%s\"}}}" .Values.ngcApiKey | b64enc | quote }}
{{- end }}
```

Add to values.yaml:
```yaml
ngcApiKey: ""  # Personal NGC API key — pass via --set ngcApiKey=nvapi-xxx
```

**Do NOT generate this template if the blueprint already creates NGC secrets** (e.g., the chart has its own Secret template for `ngc-api`). In that case, the NIM serving templates just reference the existing secret names.

### 4. Helm Template Format

If deployment type is `helm`:

**Never disable existing NIM paths.** If the blueprint already has NIM Operator or standalone Deployment, keep them as-is. Add NIM serving as a new option with `enabled: false`.

**Wrap resources in conditional:**
```yaml
{{- if .Values.nimServing.<model-key>.enabled }}
# ... resources for this model
{{- end }}
```

**Resources:** Extract CPU, memory, and GPU limits/requests from the blueprint's existing specs (docker-compose deploy.resources, Helm values, pod specs). Always include CPU and memory — NIM containers need significant resources. If the blueprint doesn't specify CPU/memory, use the dashboard "Custom" defaults: requests 8 CPU / 32Gi memory, limits 16 CPU / 64Gi memory.

**Add to values.yaml (base defaults):**

Use existing secrets detected by the model analyzer. If the blueprint already creates an NGC API secret (e.g., `ngc-api`), default to that name so NIM serving works out of the box with `helm install`. If no existing secret was found, default to `nvidia-nim-secrets` (the RHOAI standard name).

```yaml
nimServing:
  secrets:
    ngcApiSecret: "<existing ngc_api_secret or 'nvidia-nim-secrets'>"
    ngcPullSecret: "<existing ngc_pull_secret or 'ngc-secret'>"
  <model-key>:
    enabled: false
    displayName: "Human Readable Name"
    modelFormat: "model-format-name"
    image:
      repository: nvcr.io/nim/<org>/<model>
      tag: "latest"
    service:
      name: "<service-name>"
      port: 8000  # Use model's original_port from analyzer (default 8000)
    resources:
      # Extract from blueprint specs — match existing resource definitions
      limits:
        cpu: 16
        memory: 64Gi
        nvidia.com/gpu: 1
      requests:
        cpu: 8
        memory: 32Gi
        nvidia.com/gpu: 1
    storage:
      size: "50Gi"
      storageClass: ""
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    securityContext: {}
    externalRoute: false         # true = set visibility "exposed" + create external Route
    replicas: 1
    maxReplicas: 1
```

**Add to OpenShift overlay (`values-openshift.yaml`):**

If the blueprint has an OpenShift overlay, add the **full** `nimServing` block under the subchart key — not just the overrides. The overlay is what users deploy with (`-f values-openshift.yaml`), so it must be complete and self-contained. Set `securityContext.fsGroup` for PVC permissions. Use the same secret names as the base values.yaml.

```yaml
nimServing:
  secrets:
    ngcApiSecret: "<same as base values.yaml>"
    ngcPullSecret: "<same as base values.yaml>"
  <model-key>:
    enabled: false
    displayName: "Human Readable Name"
    modelFormat: "model-format-name"
    image:
      repository: nvcr.io/nim/<org>/<model>
      tag: "latest"
    service:
      name: "<service-name>"
      port: 8000
    resources:
      # Match blueprint specs
      limits:
        cpu: 16
        memory: 64Gi
        nvidia.com/gpu: 1
      requests:
        cpu: 8
        memory: 32Gi
        nvidia.com/gpu: 1
    storage:
      size: "50Gi"
      storageClass: ""
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    securityContext:
      fsGroup: 1000
    externalRoute: false         # true = set visibility "exposed" + create external Route
    replicas: 1
    maxReplicas: 1
```

### 5. Standalone YAML Format (oc-apply)

If deployment type is `oc-apply`:
- Generate separate YAML files per model: `nim-<model-name>-servingruntime.yaml`, `nim-<model-name>-inferenceservice.yaml`, `nim-<model-name>-pvc.yaml`
- Use placeholder `NAMESPACE` for namespace references

---

## Output

Create the template files in the blueprint directory. Report what was created:

```json
{
  "files_created": ["path/to/file1.yaml", "path/to/file2.yaml"],
  "values_added": {"nim": {"model-key": {...}}},
  "models_generated": ["model-1", "model-2"]
}
```
