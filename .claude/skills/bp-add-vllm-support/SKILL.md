---
name: bp-add-vllm-support
description: >-
  Use when the user asks to add vLLM support, replace NIM with vLLM, add
  open-source inference, or add KServe-based model serving to a blueprint
  that already has RHOAI/Helm support.
argument-hint: <path-to-blueprint-directory-or-git-url>
allowed-tools: Bash, Read, Write, Edit, Agent, AskUserQuestion, Skill, WebFetch, WebSearch, mcp__plugin_context7_context7__query-docs, mcp__plugin_context7_context7__resolve-library-id
---

# Add vLLM Support to NVIDIA Blueprint

You are adding vLLM as an alternative model-serving backend to an NVIDIA Blueprint that already has RHOAI/Helm support. vLLM replaces NVIDIA NIM containers with open-source inference using KServe on OpenShift AI.

## Key Principles

- **Additive** — add vLLM serving alongside any existing paths (NIM Operator, API call, raw deployment). Never disable, remove, or wrap existing resources. Every generated template, every rewired endpoint, every values.yaml entry is a new branch — not a replacement.
- **Colocate secrets** — add HF token to the existing secrets template file if one exists; only create inline in `vllm-serving-runtime.yaml` if no secrets file exists. Do NOT use Helm `lookup` — guard with the model's enabled flag.
- **Disabled by default** — vLLM path is added with `enabled: false`; the user chooses which path to enable at deploy time.

**Skill base directory**: Use the path from the "Base directory for this skill" message to resolve relative file references below.

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

HF_TOKEN_SECRET="hf-token"     # default secret name for HuggingFace token
```

**Output**: `blueprint_dir` — local path used by all subsequent phases

---

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
2. **Not compatible / unknown** → Architecture not in vLLM's registry, or registry could not be verified. Ask the user:
   - **A)** Deploy original via NIM serving → record for handoff; no vLLM toggle. After this skill finishes, invoke `/bp-add-nim-serving` via the Skill tool (see Phase 7.3).
   - **B)** Keep current deployment → no toggle (legacy behavior).

---

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

**Output**: Per-model verdict (COMPATIBLE/INCOMPATIBLE/UNKNOWN), modelType, image source, gated flag. Display the compatibility matrix to the user.

**Done when:** Every model has a verdict. Compatibility matrix shown to user. If INCOMPATIBLE/UNKNOWN models exist, continue to Phase 3.1 for fallback decisions.

---

### Phase 3: User Decisions

**Use AskUserQuestion tool** for critical decisions before generating resources:

#### 3.1 Fallback choice (per INCOMPATIBLE / UNKNOWN model)

Skip if all models are COMPATIBLE.

For **each** INCOMPATIBLE/UNKNOWN model, ask **one** `AskUserQuestion` with exactly two options:

1. `Deploy original via NIM serving (/bp-add-nim-serving)` — RHOAI NIM serving (ServingRuntime + InferenceService), **not** the existing NIM Operator / current path. Include even if NIM Operator is already present.
2. `Keep current deployment (no vLLM)` — no change, no toggle added.

Record each answer into:
- `nim_handoff_models` — user wants `/bp-add-nim-serving` (no vLLM toggle)
- `keep_current_models` — stay on current deployment (no vLLM toggle)

#### 3.2 Confirm Models and Resources

`compatible_models` = original COMPATIBLE models only.

If `compatible_models` is empty: skip Phases 4–6 and 7.1, print the Phase 7.2 summary (no deploy command), then run Phase 7.3 if `nim_handoff_models` is non-empty. Stop after that — no vLLM templates to generate.

```
"Found {count} model(s) to add as vLLM paths:

{for each model in compatible_models:}
- {display_name}
  HuggingFace ID: {model_id}
  Model type: {model_type} (e.g., generate, embed)
  GPU: {gpu_count} x nvidia.com/gpu
  Gated: {yes/no}

Models for NIM serving handoff: {nim_handoff_models or 'none'}
Models keeping current deployment: {keep_current_models or 'none'}

Confirm these models and resource allocations? (or specify changes)"
```

#### Auto-resolved (no question needed)

The following are determined automatically. Both are configurable in `values.yaml` after generation:

- **GPU tolerations**: Default to `[]` (empty). User fills per their cluster taints.
- **Security context**: Default to `{}` in base values; OpenShift overlays set as needed.

---

### Phase 4: Generate vLLM Resources

Delegate to `vllm-resource-generator` subagent with `compatible_models` from Phase 3. Do **not** generate vLLM resources for `nim_handoff_models` or `keep_current_models`.

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

**Done when:** Every model in `compatible_models` has a ServingRuntime reference, InferenceService, and companion Service template written. `values.yaml` updated with `vllm:` block.

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
- Models without vLLM toggle: {nim_handoff_models + keep_current_models}
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
- Models analyzed (compatible count, incompatible/unknown count)
- Models keeping current deployment, if any
- Files created and modified
- Endpoints rewired
- Validation status
- **NIM serving handoff (announce)** — if `nim_handoff_models` is non-empty, list the models that will be handled next:
  ```
  Next: invoking /bp-add-nim-serving for:
  - {model_name} ({original image / HF id})
  ```
- **GPU tolerations** — default to `[]` (empty). Include command to discover correct values:
  ```bash
  oc get nodes -l nvidia.com/gpu.present=true \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
  ```
- **Deploy command** — show a ready-to-copy `helm install` with all required `--set` flags: enable each vLLM model, disable corresponding NIM model (`--set nimOperator.<model>.enabled=false`), HF token (only if gated models exist), and API key placeholder (only if the blueprint's secrets template requires a non-empty value — trace the fallback chain to find the root key). Skip deploy-command details when no vLLM resources were generated.
- **Before deploying** checklist (only when vLLM resources were generated):
  - **Only if gated models:** HF token secret exists in the deployment namespace (pass via `--set vllm.huggingFaceToken=hf_xxx`)
  - GPU tolerations match your cluster
  - NIM Operator / NIM serving disabled for models served via vLLM

#### 7.3 NIM serving handoff (when requested)

If `nim_handoff_models` is **empty**: vLLM skill is done after 7.2.

If `nim_handoff_models` is **non-empty**: after the summary, **ask the user** via `AskUserQuestion`:

```
"The following models are incompatible with vLLM and were marked for NIM serving:
- {model_name} ({image or HF id})

Want me to run /bp-add-nim-serving now for these models?"
Options: "Yes, run it now" | "No, I'll run it later"
```

**If yes**: invoke `/bp-add-nim-serving` via the **Skill tool** with a soft handoff note:

```
Skill tool → bp-add-nim-serving
Arguments:
  {blueprint_dir}

  Handoff from bp-add-vllm-support (soft preference — NIM skill still rediscovers and confirms):
  - Prefer NIM serving for: {nim_handoff_models as name + image/HF id}
  - Already adding vLLM path (do not require NIM unless user asks): {compatible_models or 'none'}
  - Keeping current deployment (no NIM serving unless user asks): {keep_current_models or 'none'}
```

**If no**: print the command the user can run later:
```
/bp-add-nim-serving {blueprint_dir}
```

**Rules:**
- Invoke only after this skill's own work is finished (summary printed; Phases 4–6 skipped or completed).
- Do **not** call NIM subagents, copy NIM prompts, or hard-filter NIM discovery from this skill — `bp-add-nim-serving` runs its full standalone flow (discovery, user confirmation, generation, validation). The handoff note only biases confirmation defaults.

**Done when:** Docs updated (if vLLM resources exist), summary printed, and — when `nim_handoff_models` is non-empty — user has been asked and either `/bp-add-nim-serving` was invoked or the command was printed.

---

## Supporting Documents

### Main Agent Reads:
- `reasoning-guardrails.md`: Post-validation self-check for cross-cutting concerns — **Read at Phase 6 (after validation passes)**

### Subagent-Only Documents (DO NOT READ):
- `subagents/blueprint-analyzer-prompt.md`: Blueprint analysis instructions (Phase 1)
- `subagents/model-analyzer-prompt.md`: Model compatibility analysis instructions (Phase 2)
- `subagents/vllm-resource-generator-prompt.md`: Helm template generation instructions (Phase 4)
- `subagents/endpoint-rewiring-prompt.md`: Endpoint rewiring + NetworkPolicy instructions (Phase 5)
- `subagents/validation-prompt.md`: Post-implementation validation instructions (Phase 6)

Read main agent documents at the appropriate phase boundaries as instructed above.

**Note:** Never read subagent prompt files - they're passed to subagents via Agent tool prompt to keep main context clean.
