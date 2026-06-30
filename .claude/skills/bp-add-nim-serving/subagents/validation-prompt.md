# NIM Deployment Validation

**Role**: You are validating generated NIM deployment resources for correctness before deployment.

**Mindset**: Check syntax, structure, and NIM-specific requirements. You're NOT deploying — just verifying files are valid.

---

## Your Task

Validate that generated NIM resources are syntactically correct and follow the verified patterns from the RHOAI NIM integration.

**Context you'll receive:**
- Blueprint directory path
- Deployment type (helm | oc-apply)
- List of NIM model names
- Files created and modified

---

## Validation Checks

### 1. File Structure

```bash
cd <blueprint-directory>

# Helm chart basics
test -f Chart.yaml || echo "ERROR: Chart.yaml missing"
test -f values.yaml || echo "ERROR: values.yaml missing"
test -d templates || echo "ERROR: templates/ directory missing"
```

### 2. Rebuild Dependencies

If the blueprint uses subcharts (packaged as `.tgz` in `charts/`), rebuild so Helm picks up the new templates:

```bash
helm dependency build .
```

### 3. Helm Lint

```bash
helm lint .
```

### 4. Template Rendering (Both Modes)

```bash
# nimServing enabled — NIM serving resources should appear
helm template test . --set nimServing.<model-key>.enabled=true > /tmp/rendered-nimserving.yaml || \
  echo "ERROR: Template rendering failed (nimServing enabled)"

# nimServing disabled (default) — NIM serving resources should NOT appear
helm template test . > /tmp/rendered-default.yaml || \
  echo "ERROR: Template rendering failed (default)"
```

Verify:
- NIM ServingRuntime/InferenceService/PVC appear in nimServing-enabled render
- NIM ServingRuntime/InferenceService/PVC do NOT appear in default render
- Existing NIM paths (NIM Operator, standalone) are unchanged in both renders

### 5. NIM-Specific Validation (CRITICAL)

For each NIM model in the rendered RHOAI output:

#### 5.1 ServingRuntime Volume Mounts (9 required)

```bash
# Count volume mounts per ServingRuntime
grep -A 200 "kind: ServingRuntime" /tmp/rendered-nimserving.yaml | grep "mountPath:" | head -9
```

Verify all 9 mount paths present:
- [ ] `/dev/shm`
- [ ] `/mnt/models/cache`
- [ ] `/opt/nim/workspace`
- [ ] `/.cache`
- [ ] `/opt/nim/nginx`
- [ ] `/opt/nim/generated_configs`
- [ ] `/opt/nim/.cache`
- [ ] `/opt/nim/.config`
- [ ] `/opt/nim/.triton`

**Missing mounts cause container crashes. This is the most critical check.**

#### 5.2 ServingRuntime Annotations

Verify these annotations exist on each ServingRuntime:
- [ ] `runtimes.opendatahub.io/nvidia-nim: "true"`
- [ ] `opendatahub.io/apiProtocol: REST`
- [ ] `opendatahub.io/template-display-name: NVIDIA NIM`
- [ ] `opendatahub.io/template-name: nvidia-nim-runtime`

#### 5.3 ServingRuntime Labels

- [ ] `opendatahub.io/dashboard: "true"`

#### 5.4 Secret References

- [ ] `NGC_API_KEY` env var `secretKeyRef.name` matches `nimServing.secrets.ngcApiSecret` value
- [ ] `imagePullSecrets` matches `nimServing.secrets.ngcPullSecret` value
- [ ] Secret names in templates are parameterized via `.Values.nimServing.secrets.*` (not hardcoded)
- [ ] If blueprint creates its own NGC secret, `nimServing.secrets.ngcApiSecret` defaults to that name

#### 5.5 InferenceService

- [ ] `serving.kserve.io/deploymentMode: RawDeployment` annotation
- [ ] `opendatahub.io/dashboard: "true"` label
- [ ] `networking.kserve.io/visibility: exposed` label present only when `externalRoute: true`; label absent when false
- [ ] `HOME=/.cache` env var
- [ ] `TRITON_CACHE_DIR=/.cache/triton` env var
- [ ] GPU requests == GPU limits (Guaranteed QoS)
- [ ] Tolerations present for GPU nodes
- [ ] `modelFormat.name` matches ServingRuntime's `supportedModelFormats[0].name`
- [ ] `runtime` references the correct ServingRuntime name
- [ ] Configurable `securityContext` block present (`{{- with $nim.securityContext }}`)
- [ ] Base values.yaml has `securityContext: {}` (empty default)
- [ ] If OpenShift overlay exists: `securityContext.fsGroup: 1000` is set

#### 5.6 PVC

- [ ] Named `<service-name>-cache`
- [ ] `ReadWriteOnce` access mode
- [ ] Storage size set
- [ ] `opendatahub.io/managed: "true"` label

### 6. Endpoint Rewiring

- [ ] Original NIM service references replaced with InferenceService URLs
- [ ] URLs use correct pattern: `http://<name>-predictor.<namespace>.svc.cluster.local:<port>`
- [ ] NIM serving branch added to endpoint conditionals (existing branches unchanged)

### 7. Values Schema

```bash
# Check NIM values exist
grep -q "nimServing:" values.yaml || echo "ERROR: nimServing section missing in values.yaml"

# Check secrets config
grep -q "ngcApiSecret" values.yaml || echo "ERROR: ngcApiSecret missing"
grep -q "ngcPullSecret" values.yaml || echo "ERROR: ngcPullSecret missing"

# Verify secret name matches an existing secret in the chart
# (If the chart creates ngc-api, nimServing.secrets.ngcApiSecret should default to ngc-api)
NGC_SECRET=$(grep "ngcApiSecret:" values.yaml | awk '{print $2}' | tr -d '"')
echo "INFO: NGC API secret name = $NGC_SECRET"

# Cross-check: does the blueprint create a Secret with a DIFFERENT name?
CHART_SECRETS=$(grep -rl "kind: Secret" templates/ 2>/dev/null | xargs grep -l "NGC_API_KEY" 2>/dev/null)
if [ -n "$CHART_SECRETS" ]; then
  CHART_SECRET_NAME=$(grep -A2 "kind: Secret" $CHART_SECRETS | grep "name:" | head -1 | awk '{print $2}' | tr -d '"')
  echo "INFO: Blueprint creates NGC secret = $CHART_SECRET_NAME"
  if [ "$CHART_SECRET_NAME" != "$NGC_SECRET" ] && [ "$CHART_SECRET_NAME" != "{{ .Values.nimServing.secrets.ngcApiSecret }}" ]; then
    echo "WARNING: nimServing.secrets.ngcApiSecret ($NGC_SECRET) may not match blueprint's secret ($CHART_SECRET_NAME)"
  fi
fi
```

### 8. Fallback Secrets Template (when blueprint doesn't create secrets)

If `templates/nim-secrets.yaml` exists (generated when blueprint has no NGC secrets):
- [ ] Template is conditional on `ngcApiKey` being set (`{{- if .Values.ngcApiKey }}`)
- [ ] Uses `lookup` to skip if secrets already exist
- [ ] Creates both Opaque secret (NGC_API_KEY) and dockerconfigjson secret (nvcr.io pull)
- [ ] Secret names match `nimServing.secrets.ngcApiSecret` and `nimServing.secrets.ngcPullSecret`
- [ ] `ngcApiKey: ""` exists in values.yaml

If `templates/nim-secrets.yaml` does NOT exist, verify that the blueprint already creates NGC secrets (the model analyzer should have detected them).

---

## Output Format

```markdown
# NIM Validation Report

**Status:** [PASS | WARNINGS | ERRORS]
**Checks:** X passed, X warnings, X errors

---

## Errors
<If none: "None">

**1. <Short description>**
- File: `<path>`
- Issue: <what's wrong>
- Fix: <how to resolve>

---

## Warnings
<If none: "None">

---

## Checks Passed

- ServingRuntime volume mounts (9/9)
- ServingRuntime annotations
- Secret references (not generated)
- InferenceService RawDeployment mode
- InferenceService env vars (HOME, TRITON_CACHE_DIR)
- GPU QoS (requests == limits)
- PVC configuration
- Helm rendering (both modes)
- Endpoint rewiring
- ... (list what passed)
```

**Keep it concise** — the main agent already knows the blueprint context.

## Guidelines

1. **Volume mount count is the #1 priority** — verify all 9 are present
2. **ERROR**: Blocks deployment (missing mount, wrong annotation, template syntax)
3. **WARNING**: Works but not ideal (missing display name, non-standard sizing)
4. **Both modes must render** — nimServing enabled AND disabled (default)
5. **If tool unavailable** — mark as SKIPPED, don't fail
