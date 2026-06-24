# Deploy State Output Schema

Write `{project_path}/.bp-rhoai/deploy-state/deploy-state.yaml` with this structure.

**Important:** `unhealthy_resources` and `healthy_resources` MUST be top-level fields so the main agent can read the actionable list first without parsing the full file.

```yaml
scan_timestamp: "2026-06-18T10:00:00Z"
namespace: myns
overall_status: healthy | degraded | failing

# Main agent reads these fields first
unhealthy_resources:
  - name: app-server
    kind: Deployment
    pod_status: CrashLoopBackOff
    brief_reason: "Container exit code 1 - permission denied"
    debug_attempts: 0
  - name: redis
    kind: StatefulSet
    pod_status: Pending
    brief_reason: "PVC not bound - storage class not found"
    debug_attempts: 0

healthy_resources: [postgresql, milvus, embedding-service]

# Full details for all resources (subagents read these when diagnosing)
resources:
  - name: postgresql
    kind: Deployment
    status: healthy
    details: "1/1 replicas ready"
    pod_status: Running
    pod_name: postgresql-7f8b9c-x4k2m
  - name: app-server
    kind: Deployment
    status: unhealthy
    details: "0/1 replicas ready, 3 restarts"
    pod_status: CrashLoopBackOff
    pod_name: app-server-6d4c8b-j9m3n
    last_error: "PermissionError: [Errno 13] Permission denied: '/app/data'"
  - name: redis
    kind: StatefulSet
    status: unhealthy
    details: "0/1 replicas ready, pod pending"
    pod_status: Pending
    pod_name: redis-0
    last_error: "PersistentVolumeClaim redis-data not bound"

# Resources expected but not found (compared against deploy-analysis expected_resources)
missing_resources:
  - name: app-route
    kind: Route
    expected: true
    note: "Route not created - may need manual creation or Helm value"

# Changes since last scan (empty on first scan)
changes_since_last_scan:
  - resource: postgresql
    previous_status: unhealthy
    current_status: healthy
    note: "Pod started successfully after fix"
```

## Update Rules

When updating an existing `{project_path}/.bp-rhoai/deploy-state/deploy-state.yaml`:
- Preserve `debug_attempts` counts from the previous state
- Update `changes_since_last_scan` by comparing current vs previous status
- Keep `unhealthy_resources` and `healthy_resources` lists accurate
