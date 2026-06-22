# E2E Results Output Schema

Write `/tmp/e2e-results.yaml` with this structure:

```yaml
test_plan_source: TEST-PLAN.md  # or wherever test steps were found
namespace: myns
timestamp: "2026-06-18T12:00:00Z"

summary:
  total_tests: 12
  passed: 10
  failed: 2
  skipped: 0

tests:
  - name: "PostgreSQL connectivity"
    section: "Infrastructure Verification"
    command: "oc exec deploy/app-server -n myns -- pg_isready -h postgresql-svc"
    expected: "accepting connections"
    actual: "postgresql-svc:5432 - accepting connections"
    status: pass

  - name: "Redis connectivity"
    section: "Infrastructure Verification"
    command: "oc exec deploy/app-server -n myns -- redis-cli -h redis-svc ping"
    expected: "PONG"
    actual: "PONG"
    status: pass

  - name: "API health endpoint"
    section: "Functional Verification"
    command: "curl -sk https://app-route-myns.apps.cluster.example.com/health"
    expected: "HTTP 200 with JSON status"
    actual: "503 Service Unavailable"
    status: fail
    error_detail: "App server returns 503 - upstream embedding service not responding on expected port"

  - name: "Embedding query"
    section: "Functional Verification"
    command: "curl -sk -X POST https://app-route-myns.apps.cluster.example.com/v1/embeddings -d '{\"text\": \"test\"}'"
    expected: "HTTP 200 with embedding vector"
    actual: "Skipped - API not available (depends on health endpoint)"
    status: skipped
    skip_reason: "Prerequisite test 'API health endpoint' failed"

# Only populated if tests failed
failure_summary: |
  2 tests failed in Functional Verification section:
  1. API health endpoint - app-server returns 503, embedding service not responding
  2. Embedding query - skipped due to API being unavailable
  Root issue appears to be embedding service port misconfiguration.
```

## Rules

- Execute ALL test steps from TEST-PLAN.md, not a subset
- All oc/curl commands MUST use `-n <namespace>`
- If a test depends on a previous test that failed, mark it as `skipped` with reason
- `failure_summary` provides a human-readable explanation of what went wrong — this is what the main agent will use to report to the user
