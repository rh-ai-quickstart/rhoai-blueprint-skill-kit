---
name: bp-convert-to-rhoai
description: Convert an NVIDIA Blueprint to RHOAI-compatible version using proven patterns
argument-hint: <path-to-blueprint-directory>
allowed-tools: Bash, Read, Write, Edit, Agent, mcp__plugin_context7_context7__query-docs, mcp__plugin_context7_context7__resolve-library-id, AskUserQuestion
---

# NVIDIA Blueprint to RHOAI Conversion Skill

You are converting an NVIDIA Blueprint to run on Red Hat OpenShift AI (RHOAI) by applying proven patterns from the knowledge base and following Red Hat best practices.

## Goal

Generate minimal, conditional modifications to the blueprint that:
- Enable deployment on RHOAI via conditional mode toggles
- Preserve original functionality (blueprint works on both platforms)
- Apply proven patterns from previously converted blueprints
- Follow OpenShift and RHOAI best practices

## Input

User provides a **local path** to an NVIDIA Blueprint directory (not a GitHub URL).

## Conversion Philosophy

- **Minimal invasive changes**: Modify minimally, prefer editing over creating
- **Conditional support**: Use RHOAI mode toggles (see below) to preserve original functionality
- **Right tool for deployment**:
  - Simple notebooks → `oc apply` manifests with `OPENSHIFT_MODE` env var
  - Multi-service/docker-compose → Helm charts with `openshiftMode` value flag
  - When using Helm → all resources must be Helm-managed templates
- **Pattern reuse**: 80% from knowledge base, 20% custom reasoning

## RHOAI Mode Toggles

**For Helm deployments:**
- Add `openshiftMode: false` flag in values.yaml (default keeps original behavior)
- Use Helm conditionals: `{{- if .Values.openshiftMode }}`
- Enable at deploy time: `helm install ... --set openshiftMode=true`

**For oc apply deployments:**
- Check `OPENSHIFT_MODE` environment variable in manifests/scripts
- Original behavior when unset or false
- Enable at deploy time: `export OPENSHIFT_MODE=true; oc apply ...`

## Workflow

### Phase 1: Blueprint Analysis

Delegate to blueprint analyzer subagent to extract feature vector.

```python
BLUEPRINT_FEATURES_SCHEMA = {
    "type": "object",
    "properties": {
        "components": {"type": "array", "items": {"type": "string"}},
        "architecture": {"type": "string"},
        "deployment_types": {"type": "array", "items": {"type": "string"}},
        "resource_types_needed": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["components", "architecture", "deployment_types", "resource_types_needed"]
}

blueprint_features = Agent(
    description="Analyze blueprint structure",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-convert-to-rhoai/subagents/blueprint-analyzer-prompt.md

Blueprint directory: {blueprint_dir}

Navigate to directory, analyze structure, and extract feature vector.
Return JSON matching the schema from instructions.
""",
    schema=BLUEPRINT_FEATURES_SCHEMA
)
```

**Output**: `blueprint_features` dict with components, architecture, deployment_types, resource_types_needed

---

### Phase 2: Knowledge Retrieval

**Think like an engineer leveraging previous work**: Load knowledge files that match what you discovered about this blueprint. Don't load everything—that creates noise. Don't load too little—you'd miss proven patterns.

#### 2.1 Retrieve Knowledge Summaries as XML

Delegate to knowledge scorer subagent to score files, rank by relevance, and extract summaries from frontmatter.

```python
KB_DIR = ".claude/skills/bp-convert-to-rhoai/knowledge-base"

knowledge_summaries_xml = Agent(
    description="Retrieve knowledge summaries",
    prompt=f"""
Blueprint features: {blueprint_features}
Knowledge base directory: {KB_DIR}

Read and follow instructions from:
.claude/skills/bp-convert-to-rhoai/subagents/knowledge-scorer-prompt.md
"""
)
```

#### 2.2 Apply Knowledge from Summaries

**The subagent returns XML** with this structure:
```xml
<knowledge-base>
  <file>
    <name>component-name</name>
    <path>category/filename.md</path>
    <category>component|deployment-type|resource-pattern|architecture|integration</category>
    <summary>Proven pattern summary from frontmatter...</summary>
  </file>
  ...
</knowledge-base>
```

The XML contains summaries extracted from proven conversion patterns. **You now have the conversion experience of engineers who successfully converted previous blueprints.**

**What the summaries provide:**
- When to use which approach (e.g., "Use Approach A when original uses Helm... use Approach B when original uses docker-compose...")
- Critical configuration changes required (e.g., "pod-level runAsNonRoot: true + container allowPrivilegeEscalation: false + capabilities drop ALL")
- Common gotchas and their solutions (e.g., "chmod fails in restricted-v2 SCC - rely on automatic UID/GID assignment")
- Security context requirements for OpenShift
- Storage and resource allocation patterns

**Use this loaded knowledge during reasoning (Phase 3)** to make informed conversion decisions.

#### 2.3 Fetch Additional Details When Needed

Each `<file>` entry includes a `<path>` field pointing to the full knowledge file. **If a summary indicates a relevant pattern but you need more detail** (exact YAML code, step-by-step instructions, detailed approach comparison), read from that file.

**Guiding principle**: Summaries give you the "what" and "when" - full files give you the "how" with exact implementation. Fetch details when summary triggers a question that needs implementation specifics to answer.

---

### Phase 3: Dynamic Reasoning with Guardrails

**Read `reasoning-guardrails.md` before continuing** to understand concern areas.

#### 3.1 Think Freely About Conversion

Reason about how to convert each component. Questions should emerge organically from analysis, not from a fixed template.

**Example reasoning flow:**
```
"I see Triton Inference Server in docker-compose..."
  ↓ Question: "How should Triton be deployed on RHOAI?"
  ↓ Consult knowledge: triton-on-rhoai.md
  ↓ Answer: Needs GPU, anyuid SCC, dedicated node pool pattern

"Triton requires GPU allocation..."
  ↓ Question: "What's the OpenShift pattern for GPU resources?"
  ↓ Consult knowledge: gpu-allocation-openshift.md
  ↓ Answer: nodeSelector + tolerations + nvidia.com/gpu resource limits

"Blueprint also has Milvus..."
  ↓ Question: "Integration requirements for Triton + Milvus?"
  ↓ Consult knowledge: triton-milvus-integration.md (if exists)
  ↓ Answer: Service mesh config + init order requirements

Continue reasoning organically...
```

#### 3.2 Check Guardrails

Before proceeding to user decisions, verify all concern areas from `reasoning-guardrails.md` were addressed:
- Resource allocation (GPU, CPU, memory, storage)
- Security contexts and SCCs
- Networking (routes, services, DNS)
- Persistent storage (PVCs, access modes)
- Inter-service dependencies
- Initialization order
- Secrets and config management
- Image registries and pull secrets
- Health checks and probes
- Resource quotas and limits

If any concern feels unaddressed, reason about it explicitly before proceeding.

#### 3.3 Query Context7 (When Knowledge Gaps Exist)

If encountering technologies not in knowledge base:

```python
# Example: Blueprint uses FUSE mounts, not documented
library_id = resolve_library_id("Red Hat OpenShift", query)
docs = query_docs(library_id, "OpenShift FUSE mount security context constraints best practices")
apply_pattern_from_docs(docs)
```

**When to use:** Component/technology not in knowledge base, need Red Hat official guidance.

#### 3.4 GitHub Source Lookup (When Knowledge Insufficient)

If knowledge file lacks implementation details:
1. Check knowledge file's `source_examples` in frontmatter
2. Clone/fetch referenced repository
3. Navigate to relevant files (e.g., templates/services/triton.yaml)
4. Read actual implementation
5. Apply similar pattern to new blueprint

**When to trigger:** Knowledge summary insufficient, need exact YAML/code.

---

### Phase 4: User Decision Points

**Use AskUserQuestion tool** to gather critical decisions before generating modifications.

#### 4.1 Model Deployment Strategy (Critical)

If blueprint deploys NIM models, present options:

```
"This blueprint deploys the following NIM models:
- <model-name> (requires <GPU-count> x <GPU-type>, ~<VRAM> VRAM)
- ...

Deployment options:
A. Local deployment (requires GPU resources in your cluster)
   - Pros: Lower latency, data privacy, no API costs
   - Cons: Requires <count> GPUs, complex setup, higher infrastructure costs
   
B. NVIDIA hosted API (uses NVIDIA's infrastructure)
   - Pros: No GPU requirements, simple setup, pay-per-use
   - Cons: API costs, latency, requires internet, data leaves cluster
   
C. Hybrid (critical models local, others via API)
   - Pros: Balance between performance and cost
   - Cons: More complex configuration

Which deployment strategy do you prefer?"
```

#### 4.2 Multiple Valid Patterns

If knowledge base shows multiple approaches for same component:

```
"I found 2 patterns for deploying <component> with <requirement>:

Pattern A (from <blueprint-name>): <approach>
- Pros: <benefits>
- Cons: <tradeoffs>

Pattern B (from <blueprint-name>): <approach>
- Pros: <benefits>
- Cons: <tradeoffs>

Which approach fits your cluster setup?"
```

#### 4.3 Resource Constraints

If blueprint requires significant resources and user's cluster may not support:

```
"This blueprint requires:
- <GPU-count> x <GPU-type>
- <RAM> total memory
- <storage> persistent storage

Do you have these resources available in your OpenShift cluster?
- Yes, proceed with full local deployment
- Partially, discuss alternatives (hybrid, resource reduction)
- No, prefer NVIDIA hosted models via API"
```

#### 4.4 Cluster Storage and Configuration

If blueprint requires shared storage (PVC with RWM access mode) or has cluster-specific configuration needs:

```
"This blueprint uses/may benefit from ReadWriteMany (RWM) Persistent Volume Claims for:
- <component> - <reason> (Required/Optional)
- ...

Does your OpenShift cluster have RWM storage class available (e.g., NFS, CephFS, GlusterFS)?
- Yes, I have RWM storage available
- No, only RWO (ReadWriteOnce) storage available
- Not sure, need to check"

[If other cluster-specific questions arise during analysis:]
"Additional cluster configuration questions:
- <question-about-ingress/networking/security/etc>
- <question-about-available-operators/capabilities>
- ..."
```

**When to ask:**
- Multiple valid approaches exist in knowledge base
- Decision depends on user's environment (cluster, policies, resources)
- Model deployment strategy needed
- Knowledge base doesn't indicate clear preference
- Blueprint requires or could benefit from RWM storage
- Cluster-specific capabilities needed (ingress, networking, security policies, available operators)

---

### Phase 5: Generate Modifications

#### 5.1 Minimal Invasive Changes

**Prefer editing existing files over creating new ones:**

```bash
# Good: Modify existing values.yaml
Edit existing Helm values.yaml to add openshiftMode flag

# Avoid: Creating new file when edit would work
Don't create values-openshift.yaml when you can add conditional to values.yaml
```

**Create new files only when necessary:**
- OpenShift Routes (no docker-compose equivalent)
- RHOAI-specific manifests for oc apply deployments (when blueprint has no equivalent)

#### 5.2 Apply RHOAI Mode Patterns

**For Helm charts:**
```yaml
# values.yaml - add flag
openshiftMode: false  # Default: original behavior

# templates/deployment.yaml - conditional logic
spec:
  template:
    spec:
      {{- if .Values.openshiftMode }}
      # RHOAI-specific configuration
      securityContext:
        runAsUser: 0
      nodeSelector:
        nvidia.com/gpu.present: "true"
      {{- else }}
      # Original configuration
      securityContext: {}
      {{- end }}
```

**For docker-compose or oc apply deployments:**
```yaml
# Environment variable check
if [ "$OPENSHIFT_MODE" = "true" ]; then
  # RHOAI-specific configuration
else
  # Original configuration
fi
```

Or in YAML manifests:
```yaml
# Use environment variable substitution or envsubst
command: ["/bin/sh", "-c"]
args:
  - |
    if [ "$OPENSHIFT_MODE" = "true" ]; then
      # RHOAI config
    else
      # Original config
    fi
```

#### 5.3 Apply Component Patterns

For each component, apply patterns from loaded knowledge files:
- Security contexts from `components/<name>-on-rhoai.md`
- Storage from `resource-patterns/storage-pvc-patterns.md`
- GPU allocation from `resource-patterns/gpu-allocation-openshift.md`
- Networking from `resource-patterns/networking-routes-ingress.md`
- Integration patterns if applicable

#### 5.4 Deployment Method Selection

Based on blueprint structure:

**Simple notebook:**
- Generate `oc apply` manifests
- Create: notebook-deployment.yaml, pvc.yaml (if storage), route.yaml (if external access)
- Use `OPENSHIFT_MODE` environment variable for conditionals

**Multi-service or docker-compose:**
- Generate or modify Helm chart
- Ensure ALL resources are Helm-managed templates (no standalone `oc apply`)
- Use `openshiftMode` value flag for conditionals
- Create: Chart.yaml, values.yaml, templates/ directory

---

### Phase 6: Generate Documentation

Delegate to documentation generator subagent to create TEST-PLAN.md, RHOAI-CONVERSION.md, and update README.md.

```python
DOCUMENTATION_SUMMARY_SCHEMA = {
    "type": "object",
    "properties": {
        "files_created": {"type": "array", "items": {"type": "string"}},
        "files_modified": {"type": "array", "items": {"type": "string"}},
        "test_plan_sections": {"type": "array", "items": {"type": "string"}},
        "conversion_doc_sections": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["files_created", "files_modified", "test_plan_sections", "conversion_doc_sections"]
}

doc_summary = Agent(
    description="Generate conversion documentation",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-convert-to-rhoai/subagents/documentation-generator-prompt.md

**Documentation context:**
- Blueprint directory: {blueprint_dir}
- Deployment method: {deployment_method}
- RHOAI mode toggle: {rhoai_mode_toggle}
- Components: {components}
- Patterns applied: {patterns_applied}
- User decisions: {user_decisions}
- Modified files: {modified_files}
- Created files: {created_files}
- Knowledge sources: {knowledge_sources}

Generate TEST-PLAN.md, RHOAI-CONVERSION.md, and update README.md.
Return summary JSON matching schema from instructions.
""",
    schema=DOCUMENTATION_SUMMARY_SCHEMA
)
```

**Output**: Documentation files created in blueprint directory

---

### Phase 6.5: Automated Validation

Spawn validation subagent to verify generated files. Iterate until clean or max 3 attempts.

**IMPORTANT: Do NOT read `subagents/validation-prompt.md` yourself - it's only for the subagent to read.**

#### 6.5.1 Spawn Validation Subagent

```python
validation_report = Agent(
    description="Validate RHOAI conversion outputs",
    prompt=f"""
Read and follow validation instructions from:
.claude/skills/bp-convert-to-rhoai/subagents/validation-prompt.md

**Validation context:**
- Blueprint directory: {blueprint_dir}
- Deployment method: {deployment_method}
- RHOAI mode toggle: {rhoai_mode_toggle}
- Components: {components_list}

Navigate to blueprint directory and run validation checks.
Return report using format from instructions.
"""
)
```

#### 6.5.2 Review and Fix Issues

```python
validation_report = <subagent-response>
errors = extract_errors(validation_report)

# Review errors (filter false positives)
real_errors = [e for e in errors if is_blocking(e)]

# Fix real errors
if real_errors:
    for error in real_errors:
        attempt_fix(error)
    
    # Re-validate (max 3 iterations)
    if iteration < 3:
        run_validation_again()
```

**Review criteria:**
- Missing icon/labels? → Not blocking
- Template syntax error? → Blocking, fix it
- Missing required file? → Blocking, fix it

#### 6.5.3 Document Results

Append to RHOAI-CONVERSION.md:

```markdown
## Validation

**Status:** {PASSED | WARNINGS | ERRORS}
**Iterations:** {count}

### Issues Fixed
{list if any}

### Warnings
{list if any}

### Manual Fixes Required
{list if any}
```

Save full report: `VALIDATION-REPORT.md`

---

### Phase 7: Output Summary

**Read `output-templates.md` before continuing** for summary report template.

Print comprehensive summary using Conversion Summary Report template from `output-templates.md`, including:
- Blueprint name and architecture
- Deployment method and RHOAI mode toggle used
- Components converted
- Patterns applied
- User decisions made
- Files modified/created
- Knowledge sources used
- Context7 queries (if any)
- **Validation status** (PASSED / PASSED WITH WARNINGS / INCOMPLETE)
- **Validation iterations** (how many validation runs)
- **Issues fixed automatically** (count and summary)
- **Manual fixes required** (if any)
- Next steps for user

---

## Supporting Documents

### Main Agent Reads:
- `reasoning-guardrails.md`: Concern areas to check during reasoning - **Read at Phase 3**
- `output-templates.md`: Templates for summary report - **Read at Phase 7**
- `knowledge-base/README.md`: Knowledge base structure and usage

### Subagent-Only Documents (DO NOT READ):
- `subagents/blueprint-analyzer-prompt.md`: Blueprint analysis instructions
- `subagents/knowledge-scorer-prompt.md`: Knowledge scoring instructions
- `subagents/documentation-generator-prompt.md`: Documentation generation instructions (reads output-templates.md)
- `subagents/validation-prompt.md`: Validation instructions

Read main agent documents at the appropriate phase boundaries as instructed above ("before continuing").

**Note:** Never read subagent prompt files - they're passed to subagents via Agent tool prompt to keep main context clean.

## Important Guidelines

### DO:
- ✅ Read supporting docs at phase boundaries before continuing
- ✅ Apply proven patterns from knowledge base
- ✅ Make minimal, conditional modifications
- ✅ Preserve original functionality with mode toggles
- ✅ Ask user for deployment strategy decisions
- ✅ Generate comprehensive test plan
- ✅ Use Edit tool for modifying existing files
- ✅ Check guardrails coverage before completing
- ✅ Run automated validation via subagent before finalizing
- ✅ Review validation errors critically (check for false positives)
- ✅ Iterate validation until critical issues resolved

### DON'T:
- ❌ Load all knowledge upfront (only load what's relevant)
- ❌ Create new files when editing existing would work
- ❌ Modify without conditional RHOAI mode toggles
- ❌ Skip user decisions on model deployment strategy
- ❌ Forget to document which knowledge sources were used
- ❌ Skip validation phase (always validate before completing)
- ❌ Accept all validation errors blindly (review for false positives)
- ❌ Deliver output with unresolved critical validation errors

## Error Handling

If conversion encounters issues:
- Document what couldn't be converted automatically
- Explain why (missing pattern, complex custom logic, etc.)
- Provide manual steps for user to complete
- Suggest adding pattern to knowledge base for future
