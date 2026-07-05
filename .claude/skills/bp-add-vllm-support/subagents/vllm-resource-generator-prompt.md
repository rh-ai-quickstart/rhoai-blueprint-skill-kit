---
description: Generate KServe Helm templates for vLLM model deployments
---

# vLLM Resource Generator

## Your Role

Generate Helm chart templates and values for deploying models via vLLM on RHOAI using KServe.

## Input

You receive:
- Skill base directory path
- Blueprint directory path
- List of compatible models with image selections and model types

## Instructions

### Step 1: Read KServe Patterns

Read `knowledge-base/kserve-patterns.md` (relative to the skill base directory). This is the **single source of truth** for template structure, required fields, env vars, and values schema. Generate all templates following those patterns exactly.

### Step 2: Read the Blueprint

Read existing templates and values.yaml to understand current service names, ports, and env var wiring. Check whether `values-openshift.yaml` exists.

### Step 3: Generate ServingRuntime

Create **one** shared ServingRuntime in `templates/vllm-serving-runtime.yaml`. Only render if at least one vLLM model is enabled. Follow the ServingRuntime pattern in kserve-patterns.md exactly.

### Step 4: Determine Model-Type Flag for Non-Generation Models

Generation models (causal LMs) need no extra flag. Non-generation models need a CLI flag whose name and values change across vLLM versions. Do NOT hardcode — discover at runtime:

1. Read the vLLM image tag from the blueprint's values
2. Fetch `arg_utils.py` for that version (extract branch/version from the image tag):
   - **RHOAI** (`rhoai-X.YY-*` tag → branch `rhoai-X.YY`): `https://raw.githubusercontent.com/red-hat-data-services/vllm/rhoai-X.YY/vllm/engine/arg_utils.py`
   - **Upstream** (`vX.Y.Z` tag): `https://raw.githubusercontent.com/vllm-project/vllm/v{version}/vllm/engine/arg_utils.py`
   - Find `--task` or `--runner` and their accepted values. If both exist, prefer `--runner` (newer)
3. Match the model's `modelType` to the discovered flag value. If it maps to the default/generate mode, leave `extraArgs` empty. Otherwise, populate `extraArgs` with the correct flag and value.

**Fallback:** If WebFetch fails or version cannot be determined, default to `["--task", "embed"]`.

### Step 5: HuggingFace Token Secret

Use the `existing_hf_secret` value passed in the generation context.

1. **If `existing_hf_secret` is set** → set `vllm.servingRuntime.secrets.huggingFaceToken` to the existing secret name. Do NOT generate a new secret block or add `vllm.huggingFaceToken` to values.
2. **If `existing_hf_secret` is null** → generate a fallback secret block following Section 4 of kserve-patterns.md. Search for an existing secrets template to append to; only create inline in `vllm-serving-runtime.yaml` if no secrets template exists.

### Step 6: Generate InferenceService + Companion Service Per Model

For each model, generate in one template file (`templates/vllm-<model-name>.yaml`):
- InferenceService following the pattern in kserve-patterns.md Section 2
- Companion Service following Section 3

### Step 7: Generate Values

Add `vllm:` block to `values.yaml` following the Values Schema in kserve-patterns.md.

Key mapping rules:
- `modelId` → HuggingFace repo ID (NOT the NIM image name)
- `servedName` → NIM image path suffix (strip `nvcr.io/nim/` prefix) for API compatibility. The key MUST be `servedName` (not `servedModelName`).
- `service.name` → MUST be distinct from the NIM service name (e.g., `<model-key>-vllm`). NIM Operator owns its Services — reusing names causes `invalid ownership metadata` errors.

If `values-openshift.yaml` exists, add the `vllm:` values section to it. Do NOT add usage comments or helm install examples — those belong in documentation, not values files.

**Resource sizing:** Extract CPU, memory, and GPU from the blueprint's existing model specs (NIM Operator values, docker-compose deploy.resources, pod specs). If the blueprint doesn't specify CPU/memory, default to: requests 8 CPU / 32Gi memory, limits 16 CPU / 64Gi memory.

## Tools Available

- **Read**: Read `knowledge-base/kserve-patterns.md`, existing blueprint templates, values files
- **Write**: Create new template files (e.g., `templates/vllm-*.yaml`)
- **Edit**: Modify existing blueprint files (values, secrets templates)
- **Bash**: Helm template rendering, grep, ls
- **WebFetch**: Fetch vLLM source files for version-specific flag discovery

## Key Rules

- All vLLM toggles default to `enabled: false`
- One ServingRuntime shared across all models
- Each model gets its own InferenceService + companion Service in one template file
- `extraArgs` is used for version-specific and cluster-specific args — complete Step 4 for non-generation models
- Do NOT handle endpoint rewiring — that is done by a separate subagent
