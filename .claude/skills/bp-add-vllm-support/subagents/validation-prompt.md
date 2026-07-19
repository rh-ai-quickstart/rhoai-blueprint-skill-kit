---
description: Validate vLLM integration Helm templates
---

# vLLM Integration Validation

**Role**: Validate generated vLLM deployment resources for correctness before deployment.

**Single source of truth**: `knowledge-base/kserve-patterns.md` defines the correct template structure, required fields, env vars, and values schema. Validate rendered output against it.

**Mindset**: Check syntax, structure, and vLLM-specific requirements. You're NOT deploying — just verifying files are valid.

## Input

- Blueprint directory path
- Skill base directory path (for knowledge-base access)
- List of compatible model names
- List of models without a vLLM toggle (`nim_handoff_models` + `keep_current_models`)

## Instructions

### 1. Read KServe Patterns

Read `knowledge-base/kserve-patterns.md` from the skill base directory. All field-level validation below references this file.

### 2. File Structure

```bash
cd <blueprint-directory>
test -f Chart.yaml || echo "ERROR: Chart.yaml missing"
test -f values.yaml || echo "ERROR: values.yaml missing"
test -d templates || echo "ERROR: templates/ directory missing"
test -f templates/vllm-serving-runtime.yaml || \
echo "ERROR: vllm-serving-runtime.yaml missing"
```

### 3. Rebuild Dependencies

If the blueprint uses subcharts (packaged as `.tgz` in `charts/`):
```bash
helm dependency build .
```

### 4. Helm Lint

```bash
helm lint .
```

### 5. Template Rendering (Both Modes)

```bash
# vLLM enabled
helm template test . \
--set vllm.models.<model-key>.enabled=true \
> /tmp/rendered-vllm.yaml || \
echo "ERROR: Template rendering failed (vLLM enabled)"

# vLLM disabled (default)
helm template test . > /tmp/rendered-default.yaml || \
echo "ERROR: Template rendering failed (default)"
```

Verify:
- vLLM resources appear in enabled render, absent in default render
- Existing resources (NIM Operator, standalone, API) unchanged in both

### 6. Validate Against kserve-patterns.md

For each vLLM model in the rendered output, validate every field against the patterns in kserve-patterns.md:

- **ServingRuntime** (Section 1): all env vars, annotations, labels, volumes, container command/args, multiModel, supportedModelFormats. Only renders if at least one model enabled.
- **InferenceService** (Section 2): RawDeployment annotation, model args (--model <hf-id>, --download-dir /mnt/models, --served-model-name, --tensor-parallel-size, --trust-remote-code), NO storageUri, resources, tolerations, securityContext. GPU requests == limits (Guaranteed QoS).
- **Companion Service** (Section 3): port mapping, selector pattern, service name from values.
- **Secret references** (Section 4): secretKeyRef.name matches values, parameterized (not hardcoded). No Helm `lookup` — use simple enabled flag guards. If blueprint has existing secrets template: HF token appended there. Otherwise: in vllm-serving-runtime.yaml.

**Critical check**: companion Service selector must be `app: isvc.<isvc-name>-predictor` where `<isvc-name>` matches InferenceService metadata.name. Wrong selector = no traffic.

### 7. Endpoint Rewiring

- Application env vars have a `vllm` conditional branch
- URLs use companion Service name with parameterized namespace
- Model name references use `servedName` (NOT `servedModelName`)
- Existing branches unchanged (additive)

### 8. NetworkPolicy

If the blueprint has a NetworkPolicy, verify `component: predictor` is in egress pod selectors.

### 9. Values Schema

Validate against the Values Schema in kserve-patterns.md:
- `vllm:` section exists in values.yaml
- All models default to `enabled: false`
- `huggingFaceToken` key present
- `securityContext` key present

### 10. OpenShift Overlay

If `values-openshift.yaml` exists:
- Full `vllm:` block present (not just overrides)
- Tolerations match the NIM Operator section

### 11. Models Without a vLLM Path

Verify every model in the “no vLLM toggle” list (`nim_handoff_models` + `keep_current_models`) has NO entry under `vllm.models` in values.yaml (or no `enabled` toggle for that key).

## Output Format

```markdown
# vLLM Validation Report

**Status:** [PASS | WARNINGS | ERRORS]
**Checks:** X passed, X warnings, X errors

## Errors
<If none: "None">

**1. <Short description>**
- File: `<path>`
- Issue: <what's wrong>
- Fix: <how to resolve>

## Warnings
<If none: "None">

## Checks Passed
- <list what passed>
```

## Guidelines

1. Companion Service selector is the **#1 priority** — wrong selector means no traffic
2. **ERROR**: Blocks deployment (wrong selector, missing env var, template syntax)
3. **WARNING**: Works but not ideal (missing display name, non-standard sizing)
4. Both modes must render cleanly
5. Existing templates must be unchanged — additive only
6. Do NOT suppress stderr with `2>/dev/null || true` — we need to see errors
7. If a tool is unavailable, mark as SKIPPED, don't fail
