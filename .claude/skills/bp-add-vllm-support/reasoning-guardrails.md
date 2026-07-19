# vLLM Post-Validation Self-Check

Cross-cutting concerns to verify after validation completes, before proceeding to documentation. Template-level specs live in `knowledge-base/kserve-patterns.md` — this checklist covers higher-level reasoning that mechanical validation cannot catch.

## Concerns

### 1. Version-Specific Compatibility
- Was the architecture checked against `registry.py` for the **exact deployed vLLM version**, not docs.vllm.ai or upstream `main`?
- Was `registry.py` actually fetched and read, not answered from memory? Can the subagent quote the exact dict name where the architecture was found?
- If the registry fetch failed, is the verdict `UNKNOWN` (never `COMPATIBLE`)?
- Are models with `trust_remote_code` or `auto_map` in config.json flagged with `trustRemoteCode: true`?
- Are gated models (HuggingFace 401/403) marked `gated: true` and still treated as compatible?
- Was the HuggingFace ID verified by fetching the actual repo, not derived from the NIM image path? NIM and HuggingFace use different naming conventions.

### 2. Model-Type Flag Discovery
- Was the flag name (`--task` vs `--runner`) discovered from `arg_utils.py` for the deployed version, NOT hardcoded?
- Are the accepted values discovered at runtime, not assumed?
- If both `--runner` and `--task` exist, was `--runner` preferred (newer)?
- Is the flag in `extraArgs`, not baked into the template?
- Is the fallback `["--task", "embed"]` used ONLY when discovery fails?

### 3. Companion Service Selector
- Does every companion Service use selector `app: isvc.<isvc-name>-predictor`?
- Does `<isvc-name>` exactly match the InferenceService `metadata.name`?
- Wrong selector = no traffic reaches vLLM. This is the #1 cause of silent failures.

### 4. HF Token Secret
- If an existing `secrets.yaml` / `secret.yaml` exists in `templates/`, is the HF token appended there (not in a separate file)?
- Is the secret guarded by the model's enabled flag (`{{- if .Values.vllm.models.<key>.enabled }}`)?
- Does the ServingRuntime's `secretKeyRef.name` match `vllm.servingRuntime.secrets.huggingFaceToken`?
- Is Helm `lookup` absent? Do NOT use `lookup` to detect existing secrets — it breaks `helm template` and adds unnecessary complexity.

### 5. Additive Invariant
- Are existing NIM/deployment paths completely unchanged?
- Is every rewired endpoint an additive conditional branch (vLLM first, existing as else)?
- Do companion Service names differ from NIM service names? (NIM Operator owns its Services — reusing names causes `invalid ownership metadata` errors.)

### 6. Namespace Convention
- Does the vLLM endpoint branch use the same namespace pattern as existing branches? Charts may use a helper (e.g., `{{ include "chartname.namespace" . }}`) instead of `{{ .Release.Namespace }}`. Check `_helpers.tpl` and existing URL branches.

### 7. Anti-Patterns to Catch
- Does the InferenceService have `--download-dir /vllm/model`? vLLM downloads the model itself — there is no `storageUri` or KServe storage-initializer. Do NOT use `/mnt/models` — KServe auto-injects a volume at that path, causing a duplicate mount error.
- Is `HF_HUB_OFFLINE` set to `"0"` in the ServingRuntime env? The RHOAI vLLM image sets `HF_HUB_OFFLINE=1` by default (assumes storage-initializer downloads the model). We must override it to `0` so vLLM can download from HuggingFace Hub directly.
- Is `storageUri` absent from the InferenceService? We do NOT use KServe's storage-initializer — vLLM handles model download directly.
- Is `--tensor-parallel-size` using positional `index` syntax (`{{ index $model.resources.limits "nvidia.com/gpu" | quote }}`), NOT pipe syntax (`$model.resources.limits | index "nvidia.com/gpu"`)? Pipe syntax produces wrong output silently.

### 8. Completeness
- Does every model in `compatible_models` have all three resources generated (ServingRuntime ref, InferenceService, companion Service)?
- Do models in `nim_handoff_models` and `keep_current_models` have NO vLLM toggle in values? Do not generate vLLM for NIM-handoff models “just in case.”
- Were ALL model endpoint references found during rewiring for models that received a vLLM path? Check env vars, config files, and Helm values — missed references mean some code paths still point at the old service when vLLM is enabled.
- For template-level field correctness (model args, tensor-parallel-size, env vars, port mapping), defer to kserve-patterns.md — the validation subagent checks those mechanically.

### 9. Incompatible-Model Fallback
- For each INCOMPATIBLE/UNKNOWN model, did Phase 3.1 `AskUserQuestion` include **both** options: `Deploy original via NIM serving (/bp-add-nim-serving)` **and** `Keep current deployment (no vLLM)`? Neither may be omitted.
- If the user chose NIM handoff, was the user asked whether to run `/bp-add-nim-serving` now (Phase 7.3)? If yes: invoked via Skill tool with soft handoff note. If no: command printed for later use.
- Was NIM left to run its own full flow (no calling NIM subagents from this skill)?

## Self-Check Checklist

Before proceeding to Phase 7:
- [ ] Version-specific compatibility verified (not upstream main, not docs.vllm.ai)
- [ ] Model-type flags discovered at runtime, not hardcoded
- [ ] Companion Service selectors match InferenceService names exactly
- [ ] Existing paths unchanged — additive only
- [ ] HF token placed in existing secrets template if one exists; no Helm `lookup`
- [ ] Has `--download-dir /vllm/model` in InferenceService args, no `storageUri`
- [ ] `--tensor-parallel-size` uses positional `index` syntax (not pipe)
- [ ] Every model in `compatible_models` has all three resources; NIM-handoff and keep-current models have no toggle
- [ ] Phase 3.1 always offered NIM serving handoff + keep-current (not dropped)
- [ ] NIM handoff announced in summary when user chose that path
- [ ] Phase 7.3 asked whether to run `/bp-add-nim-serving` now when `nim_handoff_models` non-empty; Skill invoke only on yes, else command printed
- [ ] No blocking errors remain from validation report
