# NIM Deployment Reasoning Guardrails

Concern areas to check during NIM resource generation — reasoning prompts, not specs. For exact YAML patterns and required fields, see `nim-serving-patterns.md`.

## Concern Areas

### 1. Model Identification
- Are all NIM references found? Check both `nvcr.io/nim/*` images AND `integrate.api.nvidia.com` API calls.
- Is each model's type correctly identified (LLM, embedding, reranking, VLM)?
- For local models: are resources extracted from blueprint specs (values.yaml, deploy section, pod specs)?
- For API-based models: are resources determined from NVIDIA documentation and confirmed with user?
- Does the model format name match between ServingRuntime and InferenceService?

### 2. Volume Mount Completeness
- Does every ServingRuntime include all 9 volume mounts from `nim-serving-patterns.md`? Missing mounts cause container crashes.

### 3. PVC Sizing
- Is PVC sized based on blueprint specs or NVIDIA documentation? (no hardcoded defaults)
- Has the user confirmed PVC sizing?
- Is PVC named `<service-name>-cache`?
- Is `ReadWriteOnce` access mode used (single replica)?
- Is storageClass configurable (default to cluster default)?

### 4. Secret References
- Does ServingRuntime reference secrets via `.Values.nimServing.secrets.*` (not hardcoded names)?
- Does `ngcApiSecret` default to the blueprint's existing NGC secret (e.g., `ngc-api`) if one exists?
- If no existing secret found, does it fall back to `nvidia-nim-secrets` (RHOAI standard name)?
- If blueprint doesn't create any NGC secrets (`existing_secrets.created_by == "none"`), is a fallback Secret template generated (`nim-secrets.yaml`) so the user can pass `--set ngcApiKey=nvapi-xxx`?

### 5. PVC Permissions and SecurityContext
- Does the InferenceService include a configurable `securityContext` block (`{{- with $nim.securityContext }}`)?
- Is `securityContext` defaulting to empty `{}` in base values (vanilla Kubernetes doesn't need it)?
- On OpenShift: does the overlay set `fsGroup: 1000`? Without it, PVC is owned by root and model download fails with "Permission denied".

### 6. Endpoint Rewiring
- Are all references to NIM containers found? Check:
  - Docker-compose service names (e.g., `nemollm-inference`)
  - Hardcoded URLs in env vars, config files, application code
  - Port references (usually 8000)
- Are replacements using correct InferenceService URL pattern?
  - Internal: `http://<isvc-name>-predictor.<namespace>.svc.cluster.local:<service-port>`
- **Port is required.** KServe creates a headless service (`ClusterIP: None`) for the predictor — port remapping doesn't work. The client must connect on the container port (default 8000), not the service port (80). Make the port configurable via `service.port` in values.
- Is namespace parameterized (not hardcoded)?

### 7. Conditional Deployment
- Are NIM serving resources wrapped in `{{- if .Values.nimServing.<model>.enabled }}`?
- Are existing NIM paths (NIM Operator, standalone) left unchanged?
- Are endpoint URLs conditionally switched (new branch added, not replacing)?

### 8. GPU and Tolerations
- Do GPU requests equal limits (required for Guaranteed QoS)?
- Are tolerations on InferenceService (not ServingRuntime)?
- Is the default toleration `nvidia.com/gpu: Exists, NoSchedule`?
- Does GPU count match blueprint spec or NVIDIA documentation?

### 9. Annotations and Labels
- Do annotations and labels on ServingRuntime and InferenceService match the patterns in `nim-serving-patterns.md`?
- `networking.kserve.io/visibility: exposed` only when `externalRoute: true`; omit label entirely when false.

### 10. Environment Variables
- Are all required env vars present per `nim-serving-patterns.md`? Missing ones cause startup failures.

### 11. Model Format Consistency
- Does `supportedModelFormats[0].name` in ServingRuntime match `modelFormat.name` in InferenceService?
- Is model format unique per model (not shared)?
- Is `autoSelect: true` and `priority: 1` set?

## Self-Check Before Dispatching to Validation

The validation subagent checks specs mechanically. Before dispatching, verify these higher-level concerns:
- [ ] Every NIM model from the blueprint has resources generated (none missed)
- [ ] Existing NIM paths unchanged — additive only
- [ ] Endpoints rewired with port included (`service.port`)
- [ ] Fallback Secret template generated only when blueprint has no existing NGC secrets
