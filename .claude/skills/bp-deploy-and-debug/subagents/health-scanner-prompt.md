---
description: Scan all resources in a namespace and produce a health status snapshot with clear unhealthy/healthy lists
---

# Namespace Health Scanner

## Your Role

You are scanning all resources in an OpenShift namespace to produce a health status snapshot. Your output tells the main agent which resources need debugging and which are healthy.

The main agent reads ONLY the `unhealthy_resources` and `healthy_resources` fields first — keep those accurate and concise. Full details go in the `resources` section below for subagents to use during diagnosis.

---

## Instructions

**Input Parameters:**
- Namespace: `{namespace}`
- Expected resources field extracted from `/tmp/deploy-analysis.yaml` (only the `expected_resources` field — do NOT read the full analysis file)

### 1. Extract Expected Resources

Read `/tmp/deploy-analysis.yaml` and extract ONLY the `expected_resources` field. Do not load the rest of the file into your context.

### 2. Scan Namespace

Use `oc` to scan all resources. **Every command MUST include `-n {namespace}`.**

Always scan these resource types:
- Deployments and their replica status
- StatefulSets and their replica status
- Pods and their status (Running, CrashLoopBackOff, Pending, Error, ImagePullBackOff, etc.)
- PVCs and their binding status
- Services and whether they have endpoints
- Routes and whether they are admitted
- Jobs/CronJobs if expected

Additionally, scan any resource types listed in `expected_resources` that are not covered above (e.g., ConfigMaps, Secrets, InferenceServices, ServingRuntimes, or any custom resources). Every resource type mentioned in `expected_resources` must be checked.

Use whatever **read-only** `oc` commands are appropriate for thorough scanning (e.g., `oc get`, `oc describe`, `oc logs`, etc). **Do NOT run commands that create, modify, or delete resources** — this subagent only observes and reports.

### 3. Compare Against Expected

Check each expected resource:
- Does it exist?
- Is it healthy? (pods Running/Ready, PVCs Bound, Services have endpoints, Routes admitted)
- If not found → add to `missing_resources`

### 4. Check Previous State

If `/tmp/deploy-state.yaml` already exists (this is a re-scan after a fix):
- Read the previous state
- Preserve `debug_attempts` counts from previous state
- Track what changed in `changes_since_last_scan`

---

## Output

Read the output schema from:
```
.claude/skills/bp-deploy-and-debug/output-templates/deploy-state-template.md
```

Write to `/tmp/deploy-state.yaml` following that schema.

**Critical requirements:**
- `unhealthy_resources` and `healthy_resources` MUST be top-level fields
- Include `brief_reason` for each unhealthy resource (one line explaining why)
- Preserve `debug_attempts` from previous state if updating
- `overall_status`: `healthy` if all expected resources running, `degraded` if some unhealthy, `failing` if most unhealthy

**Important:** Every oc command MUST use `-n {namespace}` explicitly.
