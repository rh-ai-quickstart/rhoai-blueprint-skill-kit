# NIM Prerequisites and Configuration

Secrets, storage, and environment variables needed for NIM serving templates.

---

## Secrets

NIM deployment requires two secrets in the target namespace: an Opaque secret with `NGC_API_KEY` and a dockerconfigjson secret for pulling images from `nvcr.io`.

### How Secrets Are Provisioned

**If the blueprint already creates NGC secrets** (e.g., `ngc-api` from `--set ngcApiKey`):
- The skill detects the existing secret during Phase 1
- NIM serving templates reference that secret name
- Works out of the box with `helm install`

**If the blueprint does NOT create NGC secrets:**
- The skill generates a fallback Secret template (`templates/nim-secrets.yaml`) that creates both secrets from `{{ .Values.ngcApiKey }}`
- User passes `--set ngcApiKey=nvapi-xxx` at install time
- Uses `lookup` to skip creation if secrets already exist

### Secret Names

| Secret Purpose | Blueprint creates | Fallback name | Data key |
|---------------|------------------|---------------|----------|
| NGC API key | `ngc-api` (or similar) | `nvidia-nim-secrets` | `NGC_API_KEY` |
| Image pull | `ngc-secret` | `ngc-secret` | dockerconfigjson |

**NIM serving templates reference secrets by configurable name** (`nimServing.secrets.ngcApiSecret` and `nimServing.secrets.ngcPullSecret`). The skill detects what the blueprint already creates and defaults to those names.

---

## OpenShift SecurityContextConstraints (SCC)

NIM containers run as UID 1000 (GID 0). On OpenShift, the pod's `fsGroup` determines PVC ownership. The interaction between SCCs and `fsGroup` is critical:

| SCC | fsGroup policy | What happens | Result |
|-----|---------------|--------------|--------|
| `restricted-v2` (default) | `MustRunAs` | Auto-assigns fsGroup from namespace UID range | Works |
| Custom SCC (from `bp-convert-to-rhoai`) | `RunAsAny` | Permits any fsGroup but does NOT auto-assign | Fails without explicit `fsGroup` in pod spec |

The InferenceService must include a configurable `securityContext` block (empty default `{}`). The OpenShift overlay sets `fsGroup: 1000`.

## Storage (PVC)

### Sizing Reference

Actual PVC sizing should come from the blueprint's existing specs or NVIDIA documentation for the specific model. The table below is for reference only — always confirm with the user.

| Model Type | Typical Range | Notes |
|------------|--------------|-------|
| LLM (8B) | 80-100Gi | Single GPU model |
| LLM (70B) | 150-200Gi | Multi-GPU model |
| Embedding | 30-50Gi | Smaller models |
| Reranking | 30-50Gi | Smaller models |
| VLM | 80-100Gi | Video/image models |

### Storage Notes

- **ReadWriteOnce** is sufficient — each NIM runs single-replica by default
- **storageClassName** can be left empty to use cluster default
- PVC is mounted at `/mnt/models/cache` in the ServingRuntime
- Model data is cached in PVC — subsequent pod restarts skip download
- PVCs persist after helm uninstall (use `oc delete pvc` to clean up)

---

## Optional Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `NIM_SERVED_MODEL_NAME` | Model name | Override served model name |
| `NIM_HTTP_API_PORT` | `8000` | Override API port (default 8000) |

