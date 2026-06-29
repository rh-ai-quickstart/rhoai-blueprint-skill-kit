---
name: bp-deploy-and-debug
description: Deploy a converted RHOAI blueprint to OpenShift, debug failures, and verify end-to-end
argument-hint: <path-to-project-directory> [namespace]
allowed-tools: Bash, Read, Write, Edit, Agent, AskUserQuestion, WebSearch
---

# RHOAI Blueprint Deploy & Debug Skill

You are deploying a converted RHOAI blueprint to an OpenShift cluster, debugging any failures systematically in dependency order, and verifying end-to-end functionality.

## Goal

Deploy the blueprint, get all resources healthy, and pass the full TEST-PLAN.md — with minimal changes that preserve the original application flow.

## Input

User provides:
- **Project path**: Local path to a blueprint directory that already has OpenShift support (YAML/Helm files, deployment docs)
- **Namespace** _(optional)_: Target OpenShift namespace. If omitted, the Cluster Access Validator subagent derives a unique default namespace (see Phase 1a).

## Critical Rules

1. **Every oc/helm command MUST use `-n <namespace>` explicitly** — never rely on current context
2. **Do NOT change the intention of the original application flow** — fix deployment/config so the existing architecture works on OpenShift
3. **Auto-apply** config/infra fixes: ports, service names, storage classes, security contexts, resource limits, env vars, image tags, probes, volumes, SCCs
4. **Auto-apply** source code changes explicitly related to OpenShift (inside `if openshift_mode` blocks, OCP-specific config)
5. **Ask user** for source code changes NOT explicitly related to OpenShift — use AskUserQuestion
6. **Project-specific deploy commands** — never hardcode generic helm/oc commands; use what the Project Analyzer discovers
7. **Max 3 fix attempts per resource per phase** — escalate to user after 3 failed attempts (health and e2e phases each get 3 attempts)

## Workflow

> **Note:** Some phases below spawn subagents via the Agent tool. Subagent prompt files in `subagents/` are loaded by those subagents — do not read them yourself.

### Phase 1: Pre-deployment Analysis

#### 1a. Spawn Cluster Access Validator Subagent

```python
result = Agent(
    description="Validate cluster access",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-deploy-and-debug/subagents/cluster-access-validator-prompt.md

Namespace: {namespace if provided, else ""}
"""
)
namespace = result.split("namespace:")[1].strip()  # extract from "namespace: <value>"
```

Verifies login and namespace existence — prompts user interactively if anything needs fixing. If no namespace was provided, the subagent derives a unique default. Returns `namespace: <value>` on success — this serves as the success marker for the subagent.

**Output validation**: If the subagent returns no text or the returned text does not contain `namespace:`, treat it as a validation failure and re-run the subagent (max 2 retries). If it still fails after retries, stop and report the cluster access failure to the user. Extract the namespace value from the `namespace:` line.

Use the returned namespace as `{namespace}` for all subsequent phases.

#### 1b. Create State Directory

```bash
if [ -d "{project_path}/.bp-rhoai/deploy-state" ]; then
  mv "{project_path}/.bp-rhoai/deploy-state" "{project_path}/.bp-rhoai/deploy-state.old.$(date +%Y%m%d-%H%M%S)"
fi
mkdir -p "{project_path}/.bp-rhoai/deploy-state"
```

#### 1c. Spawn Project Analyzer Subagent

```python
Agent(
    description="Analyze project for deployment",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-deploy-and-debug/subagents/project-analyzer-prompt.md

Project directory: {project_path}
Namespace: {namespace}
"""
)
```

**Output**: `{project_path}/.bp-rhoai/deploy-state/deploy-analysis.yaml` with deploy commands, expected resources, dependency order

Read the analysis result. Understand the deployment method, components, and dependency graph.

---

### Phase 2: Deploy

Spawn Deploy Executor subagent:

```python
Agent(
    description="Deploy to OpenShift",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-deploy-and-debug/subagents/deploy-executor-prompt.md

Namespace: {namespace}
Project path: {project_path}
"""
)
```

**Output validation**: If the subagent's return text does not contain `deploy-execute-status: success`, treat it as a deployment failure and re-run the subagent (max 2 retries). If it still fails after retries, stop and report the deployment failure to the user.

---

### Phase 3: Initial Health Scan

Spawn Health Scanner subagent:

```python
Agent(
    description="Scan namespace health",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-deploy-and-debug/subagents/health-scanner-prompt.md

Namespace: {namespace}
Expected resources file: {project_path}/.bp-rhoai/deploy-state/deploy-analysis.yaml (read ONLY the expected_resources field)
"""
)
```

**Output**: `{project_path}/.bp-rhoai/deploy-state/deploy-state.yaml`

Read `unhealthy_resources` field from the state file.
- If empty (all healthy) → skip to Phase 5
- If unhealthy resources exist → enter Phase 4

---

### Phase 4: Debug Loop

Read `dependency_order` from `{project_path}/.bp-rhoai/deploy-state/deploy-analysis.yaml` to sort unhealthy resources — fix leaves first (resources with no dependencies), then work up.

**For each unhealthy resource** (in dependency order):

#### 4a. Spawn Debugger Subagent

```python
Agent(
    description=f"Debug {resource_name}",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-deploy-and-debug/subagents/resource-debugger-prompt.md

Resource name: {resource_name}
Resource kind: {resource_kind}
Namespace: {namespace}
Project path: {project_path}
Phase: health
Current attempt number (attempt_number): {attempt_number}
"""
)
```

**Output**: `{project_path}/.bp-rhoai/deploy-state/debug-{resource_name}.yaml` — appended with `health_attempt_{N}` entry containing root cause and proposed fix

#### 4b. Spawn Fix Applier Subagent

Extract deploy commands to pass to the fix applier:

```bash
yq eval '.deploy_commands' {project_path}/.bp-rhoai/deploy-state/deploy-analysis.yaml
```

```python
Agent(
    description=f"Fix {resource_name}",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-deploy-and-debug/subagents/fix-applier-prompt.md

Debug report: {project_path}/.bp-rhoai/deploy-state/debug-{resource_name}.yaml
Namespace: {namespace}
Project path: {project_path}
Deploy commands: {deploy_commands}
Phase: health
Current attempt number (attempt_number): {attempt_number}
"""
)
```

**Output**: `{project_path}/.bp-rhoai/deploy-state/fix-{resource_name}.yaml` updated with `health_attempt_{N}` entry

#### 4c. Re-scan Health

Spawn Health Scanner subagent (same as Phase 3) to re-scan ALL resources.

**Output**: Updated `{project_path}/.bp-rhoai/deploy-state/deploy-state.yaml`

#### 4d. Evaluate Result

Read `unhealthy_resources` from updated state file:

- **Resource now healthy** → move to next unhealthy resource
- **Still unhealthy AND attempt < 3** → back to 4a with incremented attempt number
  - Debugger reads previous `health_attempt_*` entries from both `{project_path}/.bp-rhoai/deploy-state/debug-{resource_name}.yaml` and `{project_path}/.bp-rhoai/deploy-state/fix-{resource_name}.yaml`
- **Attempt = 3** → AskUserQuestion:
  ```
  "Resource {resource_name} ({resource_kind}) is still unhealthy after 3 fix attempts:

  Attempt 1: {issue} → {fix_applied} → {result}
  Attempt 2: {issue} → {fix_applied} → {result}
  Attempt 3: {issue} → {fix_applied} → {result}

  Options:
  A. Skip this resource and continue with others
  B. Provide guidance on how to fix this resource
  C. Stop deployment and investigate manually"
  ```

Continue to next unhealthy resource until all processed.

**Note:** Since every resource either gets fixed or is escalated to the user at attempt 3, reaching Phase 5 with unhealthy resources should be very rare (only if user chose to skip).

---

### Phase 5: E2E Testing & Debug

#### 5a. Run E2E Tests

Spawn E2E Tester subagent:

```python
Agent(
    description="Run E2E tests",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-deploy-and-debug/subagents/e2e-tester-prompt.md

Project path: {project_path}
Namespace: {namespace}
"""
)
```

**Output**: `{project_path}/.bp-rhoai/deploy-state/e2e-results.yaml`

Read the results:
- All tests pass → skip to Phase 6
- Failures → read `failure_summary` to identify which resource(s) are causing failures, enter 5b

#### 5b. E2E Debug Loop

Same debug→fix cycle as Phase 4, but with `Phase: e2e` and max 3 attempts per resource.

For each implicated resource (in dependency order):

1. Spawn Debugger subagent (same as 4a, but pass `Phase: e2e`)
2. Spawn Fix Applier subagent (same as 4b, but pass `Phase: e2e`)
3. Re-run E2E Tester (same as 5a) to check if the failure is resolved
4. Evaluate:
   - **E2E test now passes for this resource** → move to next implicated resource
   - **Still failing AND attempt < 3** → retry with incremented attempt number
   - **Attempt = 3** → escalate to user (same AskUserQuestion format as Phase 4d)

The debug/fix files (`{project_path}/.bp-rhoai/deploy-state/debug-{resource_name}.yaml`, `{project_path}/.bp-rhoai/deploy-state/fix-{resource_name}.yaml`) are the same files used in Phase 4. E2E attempts are written under `e2e_attempt_N` keys — separate from `health_attempt_N` keys, so full history is preserved.

---

### Phase 6: Final Report

**Read `output-templates/final-report-template.md` before continuing** for report format.

Generate and print the final report including:
- Deployment status
- Resources deployed and final state
- Issues found and fixes applied (read from `{project_path}/.bp-rhoai/deploy-state/fix-*.yaml` files)
- E2E test results (pass/fail per test)
- If E2E failures: clear description of what failed and why
- Files modified during debugging

---

## Important Guidelines

### DO:
- Always use `-n <namespace>` on every oc/helm command
- Use project-specific deploy commands from analysis (not generic templates)
- Fix resources in dependency order (leaves first)
- Read `unhealthy_resources` field first from state file
- Let subagents handle diagnosis and fixing — keep orchestration clean
- Escalate to user after 3 failed attempts per resource per phase (health and e2e are separate)
- Debug E2E failures using the same debug→fix cycle with `Phase: e2e`

### DON'T (never do any of these):
- Never read subagent prompt files in the main agent
- Never hardcode generic helm install / oc apply commands
- Never change the intention of the original application flow
- Never apply source code changes not related to OpenShift without user approval
- Never skip health scan after a fix (always re-scan full namespace)
- Never exceed 3 debug attempts per resource per phase
