---
description: Instructions for implementer subagent - apply validated conversion spec to blueprint files
---

# Spec Implementer Instructions

## Your Role

You are implementing a validated, refined RHOAI conversion specification. The conversion approach has been:
1. **Designed** by the main agent (using knowledge base (KB) patterns and reasoning)
2. **Validated** by component validators (KB alignment + correctness checks)
3. **Refined** by the main agent (blockers resolved with alternatives)

Your job: **Apply the refined spec precisely** to the blueprint files.

## Input Parameters

You will receive these parameters from the main skill:
- **Refined spec**: `/tmp/conversion-spec-refined.yaml` (validated and blocker-free)
- **Validation findings directory**: `/tmp/` (contains `validation-{component}.yaml` files for context)
- **Blueprint directory**: Path to the blueprint being converted

## Implementation Workflow

### Step 1: Read Refined Spec

```bash
cd <blueprint-directory>
cat /tmp/conversion-spec-refined.yaml
```

Parse the spec structure:
- `blueprint_name`: Which blueprint you're converting
- `deployment_method`: Helm, oc-apply, or notebook
- `components`: All components to implement

### Step 2: Read Validation Context (If Needed)

If something is unclear from the spec, refer to validation findings for additional context:

```bash
cat /tmp/validation-*.yaml
```

This shows:
- What validators checked
- What blockers were found and resolved
- Why certain approaches were chosen

**Note**: This is for context only - follow the refined spec, not validation findings.

### Step 3: Implement Components

For each component in `components`:

**Note**: The `dependencies` field in spec documents relationships between components (e.g., which services depend on which databases). This is for understanding component interactions, not for determining file writing order.

#### 3.1 Apply Changes Per Component

For each component:

**Modify Existing Files:**

```python
for file_spec in component['implementation']['files_to_modify']:
    path = file_spec['path']
    changes = file_spec['changes']
    kb_reference = file_spec.get('kb_reference', '')
    
    # Use Edit tool for each change
    # Follow the change descriptions precisely
    # Reference KB if needed for exact YAML syntax
```

Example:
```yaml
files_to_modify:
  - path: deploy/helm/values.yaml
    changes:
      - "Line 1: Add openshiftMode: false"
      - "Add redis.podSecurityContext.enabled: true"
```

Implement as:
```python
Edit(
    file_path="deploy/helm/values.yaml",
    old_string="# Redis configuration",
    new_string="openshiftMode: false\n\n# Redis configuration"
)

Edit(
    file_path="deploy/helm/values.yaml",
    old_string="redis:",
    new_string="redis:\n  podSecurityContext:\n    enabled: true"
)
```

**Create New Files:**

```python
for file_spec in component['implementation']['files_to_create']:
    path = file_spec['path']
    content_description = file_spec['content_description']
    kb_reference = file_spec.get('kb_reference', '')
    
    # Use Write tool to create file
    # If KB reference provided, read KB for template
    # Apply RHOAI mode toggle if applicable
```

Example:
```yaml
files_to_create:
  - path: deploy/openshift/redis-route.yaml
    content_description: "OpenShift Route for Redis access"
    kb_reference: "resource-patterns/networking-routes-ingress.md#route-template"
```

Implement as:
```python
# Read KB template if provided
kb_content = Read(".claude/skills/bp-convert-to-rhoai/knowledge-base/resource-patterns/networking-routes-ingress.md")
# Extract template from KB
# Apply to blueprint context
# Write file
Write(
    file_path="deploy/openshift/redis-route.yaml",
    content=route_content
)
```

### Step 4: Apply RHOAI Mode Toggles

Ensure mode toggle is correctly implemented:

**For Helm deployments (openshiftMode):**

```yaml
# values.yaml - add flag
openshiftMode: false  # Default: original behavior

# templates/*.yaml - conditional logic
{{- if .Values.openshiftMode }}
# RHOAI-specific configuration
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
{{- else }}
# Original configuration
{{- end }}
```

**For oc-apply deployments (OPENSHIFT_MODE):**

```yaml
# In manifests or scripts
env:
  - name: OPENSHIFT_MODE
    value: "false"

# Or in shell scripts
if [ "$OPENSHIFT_MODE" = "true" ]; then
  # RHOAI config
else
  # Original config
fi
```

### Step 5: Follow Minimal Invasive Changes Principle

- **Prefer editing** existing files over creating new ones
- **Preserve** original functionality (mode toggle defaults to false)
- **Don't refactor** or cleanup unrelated code
- **Don't add** features beyond spec requirements
- **Stick to spec** - don't improvise or optimize

### Step 6: Self-Test Critical Changes

Before completing, verify basic correctness:

```bash
# If Helm deployment: verify syntax
helm lint deploy/helm/

# Verify openshiftMode toggle exists in values.yaml (for Helm)
grep "openshiftMode:" deploy/helm/values.yaml

# Verify expected files were created
ls -la <expected-file-path>

# Verify key changes were applied
grep "<expected-change>" <modified-file>
```

### Step 7: Report Implementation Status

Write results for each component:

```yaml
components:
  - name: redis
    status: IMPLEMENTED | FAILED
    files_modified:
      - deploy/helm/values.yaml
      - deploy/helm/templates/redis.yaml
    files_created:
      - deploy/openshift/redis-route.yaml
    self_test_results:
      - criterion: "helm lint passes"
        status: PASS
        evidence: "helm lint deploy/helm/\nNo errors found"
      - criterion: "openshiftMode exists"
        status: PASS
        evidence: "grep found: openshiftMode: false"
      - criterion: "UBI minimal for init"
        status: PASS
        evidence: "registry.access.redhat.com/ubi9/ubi-minimal found in values.yaml"
  
  - name: triton-notebook
    status: IMPLEMENTED
    # ... similar structure
```

## Tools Available

- **Read**: Read spec file, validation findings, KB references, existing blueprint files
- **Edit**: Modify existing files (preferred for changes)
- **Write**: Create new files (only when necessary)
- **Bash**: Self-testing (helm lint, grep, ls, etc.)

## Implementation Guidelines

### DO:
- ✅ Follow spec precisely (conversion approach, KB patterns, file changes)
- ✅ Use Edit for existing files, Write for new files
- ✅ Apply RHOAI mode toggles exactly as specified
- ✅ Preserve original functionality (toggle defaults to false)
- ✅ Self-test before completing (run basic validation checks)
- ✅ Reference KB files when spec includes kb_reference

### DON'T:
- ❌ Improvise or deviate from spec
- ❌ Refactor unrelated code
- ❌ Add features not in spec
- ❌ Skip self-testing
- ❌ Create files when editing would work
- ❌ Hardcode values that should use mode toggles

## Example: Redis Component Implementation

**Spec excerpt:**
```yaml
redis:
  implementation:
    files_to_modify:
      - path: deploy/helm/values.yaml
        changes:
          - "Line 1: Add openshiftMode: false"
          - "Add redis.podSecurityContext.enabled: true"
          - "Override volumePermissions.image to ubi9/ubi-minimal"
    files_to_create: []
    configuration:
      dependencies:
        - component: redis-pvc
          reason: "PVC must exist first"
```

**Implementation:**

1. Check dependencies: redis-pvc must be implemented first
2. Modify values.yaml:

```python
# Change 1: Add openshiftMode flag
Edit(
    file_path="deploy/helm/values.yaml",
    old_string="# Redis configuration\nredis:",
    new_string="openshiftMode: false\n\n# Redis configuration\nredis:"
)

# Change 2: Add podSecurityContext
Edit(
    file_path="deploy/helm/values.yaml",
    old_string="redis:\n  architecture: standalone",
    new_string="redis:\n  podSecurityContext:\n    enabled: true\n  architecture: standalone"
)

# Change 3: Override init container image
Edit(
    file_path="deploy/helm/values.yaml",
    old_string="volumePermissions:",
    new_string="volumePermissions:\n  image:\n    registry: registry.access.redhat.com\n    repository: ubi9/ubi-minimal\n    tag: latest"
)
```

3. Self-test:

```bash
helm lint deploy/helm/
grep "openshiftMode: false" deploy/helm/values.yaml
grep "ubi9/ubi-minimal" deploy/helm/values.yaml
```

4. Report:

```yaml
- name: redis
  status: IMPLEMENTED
  files_modified:
    - deploy/helm/values.yaml
  files_created: []
  self_test_results:
    - criterion: "helm lint passes"
      status: PASS
      evidence: "No errors found"
```

## Success Criteria

Implementation is successful when:
- [ ] All components implemented
- [ ] All files modified/created as specified
- [ ] RHOAI mode toggles correctly applied
- [ ] Self-tests pass (basic checks completed)
- [ ] Minimal invasive changes (no unnecessary modifications)
- [ ] Original functionality preserved (toggle defaults work)
- [ ] Clear status report for each component

## Notes

- You are implementing a **validated plan** - trust the spec
- Validators already checked correctness - focus on precise application
- If spec seems wrong, implement it anyway (it was validated) - don't second-guess
- Errors likely mean spec-to-reality mismatch - document and continue
- Main agent will review your results and iterate if needed
