---
name: bp-add-vllm-support
description: >-
  Use when the user asks to add vLLM support, replace NIM with vLLM, add
  open-source inference, or add KServe-based model serving to a blueprint
  that already has RHOAI/Helm support.
argument-hint: <path-to-blueprint-directory>
allowed-tools: Bash, Read, Write, Edit, Agent, AskUserQuestion, WebFetch, WebSearch, mcp__plugin_context7_context7__query-docs, mcp__plugin_context7_context7__resolve-library-id
---

# Add vLLM Support to NVIDIA Blueprint

You are adding vLLM as an alternative model-serving backend to an NVIDIA Blueprint that already has RHOAI/Helm support. vLLM replaces NVIDIA NIM containers with open-source inference using KServe on OpenShift AI.

## Key Principles

- **Additive** — add vLLM serving alongside any existing paths (NIM Operator, API call, raw deployment). Never disable, remove, or wrap existing resources. Every generated template, every rewired endpoint, every values.yaml entry is a new branch — not a replacement.
- **Colocate secrets** — add HF token to the existing secrets template file if one exists; only create inline in `vllm-serving-runtime.yaml` if no secrets file exists. Do NOT use Helm `lookup` — guard with the model's enabled flag.
- **Disabled by default** — vLLM path is added with `enabled: false`; the user chooses which path to enable at deploy time.

**Skill base directory**: Use the path from the "Base directory for this skill" message to resolve relative file references below.

## Inputs

```bash
BLUEPRINT_DIR="${1:?}"          # required — path to the blueprint directory
HF_TOKEN_SECRET="hf-token"     # default secret name for HuggingFace token
```

## Goal

For each inference model in the blueprint, determine if it can run on vLLM and, if so, add a toggleable vLLM deployment path that:
- Defaults to **disabled** (existing deployment unchanged)
- Can be enabled per-model via `--set vllm.models.<name>.enabled=true`
- Uses KServe InferenceService + ServingRuntime on RHOAI
- Preserves NIM API compatibility via companion Services

## Scope

**Only targets inference models** — NOT databases, storage, or application services.

### Per-Model Outcomes

1. **Compatible** → Add vLLM + HuggingFace download path. Works for any model whose architecture is in vLLM's registry, regardless of original image source. Gated models need an HF token; non-gated models download without authentication.
2. **Not compatible** — Architecture not in vLLM's registry. Stays on current deployment. No toggle added.

---

## Workflow

### Phase 1: Blueprint Analysis

Delegate to `blueprint-analyzer` subagent:

```python
blueprint_info = Agent(
    description="Scan blueprint for RHOAI support and inference models",
    prompt=f"""
Read and follow instructions from:
{skill_base_dir}/subagents/blueprint-analyzer-prompt.md

Blueprint directory: {blueprint_dir}
"""
)
```

**Output**: JSON with `rhoaiSupport`, list of models (each with name, image, service name/port, GPU), and discovered vLLM image.

**If `rhoaiSupport` is false**: Abort → direct user to `/bp-convert-to-rhoai` first.

**If `vllmImage` repository/tag are null** (first run): default to `quay.io/modh/vllm:rhoai-2.25-cuda`.

**Done when:** `rhoaiSupport` is true and models list is non-empty. If no models found, print summary and stop.

---

### Phase 2: Model Compatibility & Image Selection

Delegate to `model-analyzer` subagent:

```python
compatibility = Agent(
    description="Check vLLM compatibility for each model",
    prompt=f"""
Read and follow instructions from:
{skill_base_dir}/subagents/model-analyzer-prompt.md

Skill base directory: {skill_base_dir}
Blueprint directory: {blueprint_dir}
Models: {models_list}
vLLM image: {vllm_image}:{vllm_tag}
"""
)
```

**Output**: Per-model verdict (COMPATIBLE/INCOMPATIBLE/UNKNOWN), modelType, image source, gated flag. Display the compatibility matrix to the user, then proceed with **only COMPATIBLE models**. INCOMPATIBLE and UNKNOWN models get no toggle, no template — they stay on their current deployment.

**Done when:** Every model has a verdict. Compatibility matrix shown to user. If 0 compatible models, print summary and stop — no templates to generate.

---

### Phase 3: User Decisions

**Use AskUserQuestion tool** for critical decisions before generating resources:

#### 3.1 Confirm Models and Resources

```
"Found {count} vLLM-compatible model(s) in this blueprint:

{for each compatible model:}
- {display_name}
  HuggingFace ID: {model_id}
  Model type: {model_type} (e.g., generate, embed)
  GPU: {gpu_count} x nvidia.com/gpu
  Gated: {yes/no}

Incompatible models (staying on current deployment): {list or 'none'}

Confirm these models and resource allocations? (or specify changes)"
```

#### Auto-resolved (no question needed)

The following are determined automatically. Both are configurable in `values.yaml` after generation:

- **GPU tolerations**: Default to `[]` (empty). User fills per their cluster taints.
- **Security context**: Default to `{}` in base values; OpenShift overlays set as needed.

---

### Phase 4: Generate vLLM Resources

**Read `reasoning-guardrails.md` before continuing.**

Load knowledge base: `knowledge-base/kserve-patterns.md`

Delegate to `vllm-resource-generator` subagent:

```python
vllm_resources = Agent(
    description="Generate vLLM ServingRuntime + InferenceService + companion Service templates",
    prompt=f"""
Read and follow instructions from:
{skill_base_dir}/subagents/vllm-resource-generator-prompt.md

**Generation context:**
- Blueprint directory: {blueprint_dir}
- Models: {compatible_models}
- User decisions: {user_decisions}
- Existing HF token secret: {existing_hf_secret}
- Knowledge base dir: {skill_base_dir}/knowledge-base/
"""
)
```

Apply the generated resources:
1. Write `templates/vllm-serving-runtime.yaml` (shared ServingRuntime)
2. Write `templates/vllm-<model>.yaml` for each model (InferenceService + companion Service)
3. Add `vllm:` section to `values.yaml`

**Done when:** Every COMPATIBLE model has a ServingRuntime reference, InferenceService, and companion Service template written. `values.yaml` updated with `vllm:` block.

---

### Phase 5: Endpoint Rewiring

Delegate to `endpoint-rewiring` subagent:

```python
endpoint_rewiring = Agent(
    description="Rewire model endpoints to vLLM companion Service URLs",
    prompt=f"""
Read and follow instructions from:
{skill_base_dir}/subagents/endpoint-rewiring-prompt.md

**Rewiring context:**
- Blueprint directory: {blueprint_dir}
- Models: {compatible_models}
"""
)
```

The subagent searches the blueprint for env vars or config that reference model service endpoints and adds an **additive** `vllm` conditional branch using the companion Service URL pattern. All existing branches unchanged.

**Done when:** Every original model endpoint has a `vllm` conditional branch. NetworkPolicy patched if present.

---

### Phase 6: Validation & Self-Check

Delegate to validation subagent (max 3 iterations):

```python
validation_report = Agent(
    description="Validate vLLM deployment resources",
    prompt=f"""
Read and follow instructions from:
{skill_base_dir}/subagents/validation-prompt.md

**Validation context:**
- Skill base directory: {skill_base_dir}
- Blueprint directory: {blueprint_dir}
- Compatible models: {compatible_models}
- Incompatible models: {incompatible_models}
- Files created: {files_created}
- Files modified: {files_modified}
"""
)
```

Review errors critically — template syntax errors and port mapping issues are blocking, missing icons are not.

After validation passes, **read `reasoning-guardrails.md`** and run the self-check checklist against the generated files and validation report. Fix any issues found before proceeding.

**Done when:** Validation report is PASS or WARNINGS-only, and all self-check items in reasoning-guardrails.md are satisfied.

---

### Phase 7: Documentation & Summary

#### 7.1 Update Blueprint OpenShift Documentation

Search the blueprint for OpenShift deployment docs (e.g., `openshift-deployment.md`). Add a **vLLM Alternative** section covering: how to enable vLLM, HF token prerequisite, toleration customization, and that vLLM requires RHOAI with KServe (NIM Operator is NOT required).

#### 7.2 Print Summary

Print summary including:
- Models analyzed (compatible count, incompatible count)
- Files created and modified
- Endpoints rewired
- Validation status
- **GPU tolerations** — default to `[]` (empty). Include command to discover correct values:
  ```bash
  oc get nodes -l nvidia.com/gpu.present=true \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
  ```
- **Deploy command** — show a ready-to-copy `helm install` with all required `--set` flags: enable each vLLM model, disable corresponding NIM model (`--set nimOperator.<model>.enabled=false`), HF token (only if gated models exist), and API key placeholder (only if the blueprint's secrets template requires a non-empty value — trace the fallback chain to find the root key).
- **Before deploying** checklist:
  - **Only if gated models:** HF token secret exists in the deployment namespace (pass via `--set vllm.huggingFaceToken=hf_xxx`)
  - GPU tolerations match your cluster
  - NIM Operator / NIM serving disabled for models served via vLLM

**Done when:** Docs updated and summary printed with deploy command.

---

## Guidelines

- Do NOT hardcode tolerations — leave empty array `[]`, user fills per their cluster taints
- HuggingFace 401/403 on gated model config.json is normal — mark `gated: true`, proceed

## Supporting Documents

### Main Agent Reads:
- `reasoning-guardrails.md` — post-validation self-check for cross-cutting concerns — **read after Phase 6 validation passes**
- `knowledge-base/vllm-compatibility.md` — architecture compatibility reference — **read during Phase 2 for any non-generation model**
- `knowledge-base/kserve-patterns.md` — ServingRuntime + InferenceService + companion Service templates — **read during Phase 4 before generating templates**

### Subagent-Only Documents (DO NOT READ):
- `subagents/blueprint-analyzer-prompt.md` — Phase 1: find inference models and extract details
- `subagents/model-analyzer-prompt.md` — Phase 2: model compatibility + image selection
- `subagents/vllm-resource-generator-prompt.md` — Phase 4: Helm template generation
- `subagents/endpoint-rewiring-prompt.md` — Phase 5: endpoint rewiring + NetworkPolicy
- `subagents/validation-prompt.md` — Phase 6: validation checks

Read main agent documents only when needed during the appropriate phases. Never read subagent prompt files — they're passed to subagents via Agent tool prompt to keep main context clean.
