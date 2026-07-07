---
name: bp-add-nim-serving
description: Use when a blueprint contains NIM models that need to be deployed via RHOAI NIM serving
argument-hint: <path-to-blueprint-directory-or-git-url>
allowed-tools: Bash, Read, Write, Edit, Agent, AskUserQuestion
---

# Deploy Blueprint NIM Models via RHOAI NIM Integration

You are deploying NVIDIA NIM models found in a blueprint using RHOAI's native NIM serving integration — ServingRuntime + InferenceService + PVC. This replicates what the RHOAI dashboard does when deploying NIM models, but as Helm templates so the entire blueprint deploys with one `helm install`.

## Key Principles

- **Additive** — add NIM serving alongside any existing NIM paths (NIM Operator, standalone Deployment). Never disable, remove, or wrap existing paths. Every generated resource, every rewired endpoint, every values.yaml entry is a new branch — not a replacement.
- **Secrets** — detect what the blueprint already creates and reference those. Generate a fallback Secret template only when the blueprint creates no NGC secrets.
- **Disabled by default** — NIM serving path is added with `enabled: false`; the user chooses which path to enable.

## Input

User provides either a **local path** to a blueprint directory OR a **git URL** (HTTPS/SSH).

## Workflow

### Phase 0: Input Resolution

Determine whether user input is a local path or a git URL. If it's a git URL, clone it to a well-known local directory.

```bash
INPUT="<user-provided-input>"

if echo "$INPUT" | grep -qE '^(https?://|git@|git://)'; then
  REPO_NAME=$(basename "$INPUT" .git)
  mkdir -p ~/rhoai-blueprints
  git clone "$INPUT" ~/rhoai-blueprints/"$REPO_NAME"
  blueprint_dir=~/rhoai-blueprints/"$REPO_NAME"
else
  blueprint_dir="$INPUT"
fi
```

**Output**: `blueprint_dir` — local path used by all subsequent phases

---

### Phase 1: NIM Model Discovery

Delegate to `nim-model-analyzer` subagent to scan the blueprint.

```python
nim_models = Agent(
    description="Scan blueprint for NIM models",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-add-nim-serving/subagents/nim-model-analyzer-prompt.md

Blueprint directory: {blueprint_dir}
"""
)
```

**Output**: List of NIM models with image, GPU, PVC, port, and service name details.

**Guard**: If `deployment_type` is `docker-compose`, stop and tell the user:
> "This blueprint hasn't been converted to a Helm chart yet. Run `/bp-convert-to-rhoai` first to create the Helm chart, then re-run `/bp-add-nim-serving` to add NIM serving resources."

---

### Phase 2: User Decisions

**Use AskUserQuestion tool** for critical decisions:

#### 2.1 Confirm Models and Resources

```
"Found {count} NIM model(s) in this blueprint:

{for each model:}
- {display_name} ({type})
  Image: {image}:{tag}
  GPU: {gpu_count} x nvidia.com/gpu
  CPU: {cpu_request} request / {cpu_limit} limit
  Memory: {memory_request} request / {memory_limit} limit
  PVC: {pvc_size}

Total GPU requirement: {total_gpus}
Total storage: {total_storage}

Confirm these models and resource allocations? (or specify changes)"
```

#### Auto-resolved (no question needed)

The following are determined automatically from the blueprint. If not found, sensible defaults are used. Both are configurable in `values.yaml` after generation.

- **Storage class**: Use blueprint's existing StorageClass if found, otherwise cluster default (empty storageClassName)
- **GPU tolerations**: Use blueprint's existing tolerations if found, otherwise standard `nvidia.com/gpu` with `NoSchedule`

---

### Phase 3: NIM Resource Generation

Delegate to `nim-resource-generator` subagent:

```python
nim_resources = Agent(
    description="Generate NIM ServingRuntime + InferenceService + PVC templates",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-add-nim-serving/subagents/nim-resource-generator-prompt.md

**Generation context:**
- Blueprint directory: {blueprint_dir}
- Deployment type: {deployment_type}
- Models: {nim_models}
- User decisions: {user_decisions}
- Existing secrets: {existing_secrets}
""",
)
```

**Completion**: Every model from Phase 1 has a corresponding ServingRuntime + InferenceService + PVC, all wrapped in `{{- if .Values.nimServing.<model>.enabled }}` conditionals, disabled by default.

---

### Phase 4: Endpoint Rewiring

Delegate to `endpoint-rewiring` subagent:

```python
endpoint_rewiring = Agent(
    description="Rewire NIM endpoints to InferenceService URLs",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-add-nim-serving/subagents/endpoint-rewiring-prompt.md

**Rewiring context:**
- Blueprint directory: {blueprint_dir}
- Deployment type: {deployment_type}
- Models: {nim_models}
""",
)
```

The subagent searches the blueprint for env vars or config that reference NIM service endpoints and adds an **additive** `nimServing` conditional branch using the InferenceService URL pattern. All existing branches unchanged.

**Completion**: Every original NIM endpoint (service names, URLs, env vars) has a `nimServing` conditional branch.

---

### Phase 5: Validation

Delegate to validation subagent (max 3 iterations):

```python
validation_report = Agent(
    description="Validate NIM deployment resources",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-add-nim-serving/subagents/validation-prompt.md

**Validation context:**
- Blueprint directory: {blueprint_dir}
- Deployment type: {deployment_type}
- Models: {model_names}
- Files created: {files_created}
- Files modified: {files_modified}
""",
)
```

Fix errors and re-validate until clean or max 3 attempts.

After validation passes, **read `reasoning-guardrails.md`** and run the self-check checklist against the generated files and validation report. Fix any issues found before proceeding.

---

### Phase 6: Documentation & Summary

#### 6.1 Update Blueprint OpenShift Documentation

Search the blueprint for OpenShift deployment docs (e.g., `openshift-deployment.md`). Add a **NIM Serving Alternative** section covering:
- How to enable NIM serving (`--set nimServing.<model>.enabled=true`)
- NGC secret prerequisites (either existing chart secrets or `--set ngcApiKey=nvapi-xxx`)
- Storage class and GPU toleration customization
- That NIM serving requires RHOAI with KServe (NIM Operator is NOT required for this path)

#### 6.2 Print Summary

Print summary including:
- Models deployed (name, type, GPU, PVC)
- Resources generated (ServingRuntime, InferenceService, PVC per model)
- Endpoints rewired
- Validation status
- **Cluster-specific configuration** — these defaults **must be verified against the target cluster** before deploying:

  | Setting | Default | When to Change |
  |---------|---------|----------------|
  | **Storage class** | `""` (cluster default) | Set to your cluster's storage class if the default doesn't support `ReadWriteOnce` block storage (e.g., `gp3-csi`, `ocs-storagecluster-ceph-rbd`, `managed-premium`) |
  | **GPU tolerations** | `nvidia.com/gpu: Exists, NoSchedule` | Change if your GPU nodes use different taint keys (e.g., `gpu-type`, `accelerator`, or a custom key) |

  Include commands to discover the correct values:
  ```bash
  # Find available storage classes
  oc get storageclass

  # Find GPU node taints
  oc get nodes -l nvidia.com/gpu.present=true \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
  ```

  Show how to override at install time via `--set` flags and mention the values.yaml paths for direct editing.
- **Deploy command** — show a ready-to-copy `helm install` with all required `--set` flags: enable each NIM serving model (`--set nimServing.<model>.enabled=true`), disable corresponding NIM Operator model (`--set nimOperator.<model>.enabled=false`), NGC API key (only if the skill generated a fallback Secret template — `--set ngcApiKey=nvapi-xxx`).
- **Before deploying** checklist:
  - GPU Operator installed with available GPU nodes
  - NGC secrets exist in namespace (either created by the blueprint's chart or via `--set ngcApiKey=nvapi-xxx` if the skill generated a fallback Secret template)
  - Verify storage class and GPU tolerations match your cluster (see above)

---

## Supporting Documents

### Main Agent Reads:
- `reasoning-guardrails.md`: Post-validation self-check for NIM-specific concerns — **Read at Phase 5 (after validation passes)**

### Subagent-Only Documents (DO NOT READ):
- `subagents/nim-model-analyzer-prompt.md`: NIM model discovery instructions (Phase 1)
- `subagents/nim-resource-generator-prompt.md`: Resource generation instructions (Phase 3)
- `subagents/endpoint-rewiring-prompt.md`: Endpoint rewiring instructions (Phase 4)
- `subagents/validation-prompt.md`: Post-implementation validation instructions (Phase 5)

Read main agent documents at the appropriate phase boundaries as instructed above.

**Note:** Never read subagent prompt files or knowledge-base files - they're passed to subagents via Agent tool prompt to keep main context clean.
