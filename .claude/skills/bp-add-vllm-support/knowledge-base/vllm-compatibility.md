---
name: vllm-compatibility
description: How to determine vLLM model compatibility and model-type configuration
summary: |
  Defines how to check model compatibility against vLLM's ModelRegistry (the source of truth,
  not the docs page). Compatibility is version-specific — must fetch registry.py for the exact
  deployed vLLM version. Also covers model-type flag discovery for non-generation models
  (flag name and values change across versions — must be discovered at runtime from arg_utils.py).
metadata:
  type: component
resource_types: [model-serving, gpu]
components: [vllm]
---

# vLLM Model Compatibility

## Determining Compatibility

vLLM's `ModelRegistry` in `registry.py` is the source of truth. It maps architecture strings (from HuggingFace `config.json`) to vLLM implementations. If the architecture isn't in the registry, the model won't load.

Compatibility is **version-specific** — check the registry for the exact deployed version:
- **RHOAI** (`rhoai-X.YY-*` tag → branch `rhoai-X.YY`): `https://raw.githubusercontent.com/red-hat-data-services/vllm/rhoai-X.YY/vllm/model_executor/models/registry.py`
  - If 404 (some newer branches are build-only), discover the upstream vLLM version from the branch's Dockerfile and fetch from the upstream repo at that tag.
- **Upstream** (`vX.Y.Z` tag): `https://raw.githubusercontent.com/vllm-project/vllm/v{version}/vllm/model_executor/models/registry.py`

Do NOT use `docs.vllm.ai` — the docs page reflects upstream main and is incomplete. Do NOT use HuggingFace model cards — they may reference newer vLLM versions. Do NOT check upstream `main` — architectures on `main` may not be in the deployed version. **If the fetch fails or the version cannot be determined, the verdict MUST be UNKNOWN, never COMPATIBLE.**

## Model Type Flag (Non-Generation Models)

Generation models (causal LMs) are the default mode — no extra CLI flag needed. All other model types (embedding, reranking, classification, etc.) require a flag to tell vLLM how to run them.

The flag name and accepted values change across vLLM versions (`--task` → `--runner`). The skill must discover both the flag and the correct value at runtime from `arg_utils.py` for the deployed version. Do NOT hardcode flag names or values — the set of model types vLLM supports grows across versions.

Discovery steps:
1. Read the vLLM image and tag from the blueprint's values (e.g., `vllm.servingRuntime.image`)
2. Fetch `arg_utils.py` for that version (same repo URL patterns as registry.py above) to find the flag (`--task` or `--runner`) and its accepted values. If both exist, prefer `--runner` (newer)
3. Match the model's `model_type` from HuggingFace `config.json` to the correct flag value
4. Place the discovered flag in `extraArgs` — never bake it into the template

**Fallback**: If version discovery fails (fetch error, unknown image format), default to `["--task", "embed"]` — it works on older versions which are more commonly deployed. This is a safety net, not a shortcut; always attempt discovery first.

The template must NOT have a dedicated `runner` or `task` field. All version-specific args go through `extraArgs`.
