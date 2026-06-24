# Final Report Template

Print this report to the user at Phase 6. Adapt sections based on actual results — skip sections that don't apply.

```markdown
# Deployment Report: {blueprint_name}

**Namespace:** {namespace}
**Deployment Method:** {helm | oc-apply | custom}
**Status:** {SUCCESS | PARTIAL SUCCESS | E2E FAILURES}

---

## Deployment Summary

| Resource | Kind | Status | Fix Applied |
|----------|------|--------|-------------|
| postgresql | Deployment | Running | - |
| redis | StatefulSet | Running | Fixed: storage class mismatch |
| app-server | Deployment | Running | Fixed: SCC + service name |
| embedding-svc | Deployment | Running | - |

## Issues Found & Fixed

### {resource-name} — {phase} (Attempt {N})
- **Issue:** {brief issue description}
- **Root Cause:** {root cause}
- **Fix:** {what was changed}
- **Files Modified:** {list}

{repeat for each resource that needed fixes — group by phase: health fixes first, then e2e fixes}

## Unresolved Issues
{list any resources where user chose to skip, or issues that couldn't be fixed}
{empty if all resolved}

## E2E Test Results

**Source:** {TEST-PLAN.md}
**Result:** {X}/{Y} tests passed

| Test | Status |
|------|--------|
| PostgreSQL connectivity | PASS |
| Redis connectivity | PASS |
| API health endpoint | FAIL - embedding service port mismatch |

{if failures}
### Failure Details
{clear description of what failed and likely root cause}
{/if}

## Files Modified During Debug
- `deploy/helm/templates/app-server.yaml` — added emptyDir volume, fixed env var
- `deploy/helm/values.yaml` — updated storage class name

## Deploy Report
Full state saved to: `{project_path}/.rhoai/deploy-report.yaml`
```

## Copy State to Project

At the end, copy `/tmp/deploy-state.yaml` to `{project_path}/.rhoai/deploy-report.yaml`.
Create the `.rhoai/` directory if it doesn't exist.
