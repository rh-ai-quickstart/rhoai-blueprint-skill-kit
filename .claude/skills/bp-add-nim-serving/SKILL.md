---
name: bp-add-nim-serving
description: Use when a blueprint contains NIM models that need to be deployed via RHOAI NIM serving
argument-hint: <path-to-blueprint-directory>
allowed-tools: Bash, Read, Write, Edit, Agent, AskUserQuestion
---

# Deploy Blueprint NIM Models via RHOAI NIM Integration

You are deploying NVIDIA NIM models found in a blueprint using RHOAI's native NIM serving integration — ServingRuntime + InferenceService + PVC. This replicates what the RHOAI dashboard does when deploying NIM models, but as Helm templates so the entire blueprint deploys with one `helm install`.

## Key Principles

- **Additive** — add NIM serving alongside any existing NIM paths (NIM Operator, standalone Deployment). Never disable, remove, or wrap existing paths. Every generated resource, every rewired endpoint, every values.yaml entry is a new branch — not a replacement.
- **Secrets** — detect what the blueprint already creates and reference those. Generate a fallback Secret template only when the blueprint creates no NGC secrets.
- **Disabled by default** — NIM serving path is added with `enabled: false`; the user chooses which path to enable.

## Input

User provides a **local path** to an NVIDIA Blueprint directory that contains NIM model references — either local NIM containers (`nvcr.io/nim/*` images in docker-compose, Helm values, or templates) or NVIDIA API calls (`integrate.api.nvidia.com` endpoints).

## Workflow

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

**Read `reasoning-guardrails.md` before continuing.**

Load knowledge base: `nim-serving-patterns.md`, `nim-prerequisites.md`

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
- Knowledge base dir: .claude/skills/bp-add-nim-serving/knowledge-base/
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

#### 5.1 Validation

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

#### 5.2 Summary Report

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
  oc get nodes -l nvidia.com/gpu.present=true -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
  ```

  Show how to override at install time via `--set` flags and mention the values.yaml paths for direct editing.
- **Before deploying** checklist:
  - GPU Operator installed with available GPU nodes
  - NGC secrets exist in namespace (either created by the blueprint's chart or via `--set ngcApiKey=nvapi-xxx` if the skill generated a fallback Secret template)
  - Verify storage class and GPU tolerations match your cluster (see above)

---

## Supporting Documents

### Main Agent Reads:
- `reasoning-guardrails.md`: NIM-specific concern areas — **Read at Phase 3**
- `knowledge-base/nim-serving-patterns.md`: Verified ServingRuntime + InferenceService YAML
- `knowledge-base/nim-prerequisites.md`: Secrets, SCC, storage, env vars

### Subagent-Only Documents (DO NOT READ):
- `subagents/nim-model-analyzer-prompt.md`
- `subagents/nim-resource-generator-prompt.md`
- `subagents/endpoint-rewiring-prompt.md`
- `subagents/validation-prompt.md`
