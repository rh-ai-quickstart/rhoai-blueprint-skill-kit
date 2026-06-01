---
name: jaeger-on-rhoai
description: Jaeger distributed tracing on RHOAI using official Jaeger Helm subchart with OpenShift security contexts
summary: "Deploys Jaeger distributed tracing on OpenShift using official Helm subchart as dependency, configured for restricted SCC compliance with ephemeral memory storage. Use all-in-one mode with memory storage for dev/test or moderate scale (<1000 traces/min); switch to distributed deployment with Elasticsearch backend for high scale (>1000 traces/min); migrate to Red Hat OpenShift distributed tracing platform for enterprise production with support and monitoring. Leave podSecurityContext fields (runAsUser, runAsGroup, fsGroup) empty because OpenShift SCC admission injects UIDs from namespace range and explicit values cause pod rejection; set `allowPrivilegeEscalation: false`, drop ALL capabilities, `seccompProfile: RuntimeDefault`; applications send traces to `http://{{ .Release.Name }}-jaeger-query:4318`. Memory storage is ephemeral (traces lost on restart) and causes pod eviction under high traffic; explicit UID/GID values conflict with namespace ranges; external Route exposes traces without authentication (disable in production via `jaegerRoute.enabled: false`)."
metadata:
  type: component
components: [jaeger]
deployment_types: [helm]
resource_types: [security-context]
architecture: []
source_examples:
  - blueprint: "pdf-to-podcast"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/pdf-to-podcast"
    fork_repo: "https://github.com/rh-ai-quickstart/pdf-to-podcast"
    notes: "Jaeger all-in-one deployment using official chart with restricted SCC compliance"
    approach: "A"
---

# Jaeger on RHOAI

## Overview

Jaeger is an open-source distributed tracing system used to monitor and troubleshoot microservices architectures. This pattern shows how to deploy Jaeger using the **official Jaeger Helm chart** as a subchart, configured for OpenShift restricted SCC compliance.

## Conversion Pattern

### Deployment Type: Helm Subchart

Jaeger is deployed as a **Helm dependency** (subchart) rather than a standalone deployment. Uses the official Jaeger Helm chart maintained by the Jaeger project.

### Helm Chart Dependency

**Chart.yaml dependency declaration**:

```yaml
dependencies:
  - name: jaeger
    version: "3.x.x"
    repository: "https://jaegertracing.github.io/helm-charts"
    condition: jaeger.enabled
```

**Install dependencies**:

```bash
cd helm/
helm dependency update
```

This downloads the official subchart to `helm/charts/jaeger-*.tgz`.

### Values Configuration

**Parent chart values.yaml** (configures Jaeger subchart for OpenShift):

```yaml
#############################################################################
# Jaeger configuration (using official Jaeger chart v3.x)
# Docs: https://github.com/jaegertracing/helm-charts
#############################################################################
jaeger:
  enabled: true
  provisionDataStore:
    cassandra: false
    elasticsearch: false
  allInOne:
    enabled: true
    image:
      registry: quay.io
      repository: jaegertracing/all-in-one
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 1Gi
    podSecurityContext:
      runAsUser:         # Empty: let OpenShift assign UID from namespace range
      runAsGroup:        # Empty: let OpenShift assign GID
      fsGroup:           # Empty: let OpenShift assign fsGroup
      seccompProfile:
        type: RuntimeDefault
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
  storage:
    type: memory
  agent:
    enabled: false
  collector:
    enabled: false
  query:
    enabled: false
```

### Key Configuration Decisions

**All-in-one deployment**:
- Uses `jaeger.allInOne.enabled: true` for simplified single-pod deployment
- Disables separate agent, collector, and query components
- Suitable for development and moderate-scale production

**Memory-based storage**:
- `storage.type: memory` means traces are stored in-memory (ephemeral)
- Traces are lost when pod restarts
- For production with persistence, use `elasticsearch` or `cassandra` storage

**No persistent data store**:
- `provisionDataStore.cassandra: false` and `provisionDataStore.elasticsearch: false`
- Avoids deploying heavy infrastructure components
- Reduces resource footprint for dev/test environments

### Security Context for OpenShift

**Critical for restricted SCC compliance**:

```yaml
podSecurityContext:
  runAsUser:         # MUST be empty - OpenShift assigns from namespace UID range
  runAsGroup:        # MUST be empty - OpenShift assigns from namespace GID range
  fsGroup:           # MUST be empty - OpenShift assigns
  seccompProfile:
    type: RuntimeDefault  # Required by restricted-v2 SCC

securityContext:
  allowPrivilegeEscalation: false  # Required by restricted-v2 SCC
  capabilities:
    drop:
      - ALL                         # Drop all Linux capabilities
```

**Why empty UID/GID values?**
- OpenShift automatically assigns UIDs from the namespace's allocated range
- Setting explicit `runAsUser: 1000` may conflict with namespace range and cause pod rejection
- Leaving empty allows OpenShift's SCC admission to inject correct values

### Service Discovery

The Jaeger subchart automatically creates Services with standard names:

- **Query UI**: `{{ .Release.Name }}-jaeger-query:16686` (web interface)
- **OTLP gRPC**: `{{ .Release.Name }}-jaeger-collector:4317` (if collector enabled)
- **OTLP HTTP**: `{{ .Release.Name }}-jaeger-collector:4318` (if collector enabled)

For **all-in-one mode**, the Service name is `{{ .Release.Name }}-jaeger-query`.

**Application instrumentation** (send traces to Jaeger):

```yaml
env:
  - name: OTLP_ENDPOINT
    value: "http://pdf-to-podcast-jaeger-query:4318"  # OTLP HTTP endpoint
```

**Helm helper function** (in parent chart's `_helpers.tpl`):

```yaml
{{/*
Jaeger OTLP endpoint for application services
*/}}
{{- define "pdf-to-podcast.jaegerOtlpEndpoint" -}}
http://{{ .Release.Name }}-jaeger-query:4318
{{- end }}
```

Usage in application deployment:

```yaml
env:
  - name: OTLP_ENDPOINT
    value: "{{ include "pdf-to-podcast.jaegerOtlpEndpoint" . }}"
```

### External Access via Route

To expose Jaeger UI externally for debugging:

**Parent chart Route template**:

```yaml
{{- if .Values.jaegerRoute.enabled }}
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: jaeger-ui
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "pdf-to-podcast.labels" . | nindent 4 }}
    app: jaeger
spec:
  {{- if .Values.jaegerRoute.host }}
  host: {{ .Values.jaegerRoute.host }}
  {{- end }}
  to:
    kind: Service
    name: {{ .Release.Name }}-jaeger-query
  port:
    targetPort: 16686  # Jaeger Query UI port
  {{- if .Values.jaegerRoute.tls.enabled }}
  tls:
    termination: {{ .Values.jaegerRoute.tls.termination }}
    {{- if .Values.jaegerRoute.tls.insecureEdgeTerminationPolicy }}
    insecureEdgeTerminationPolicy: {{ .Values.jaegerRoute.tls.insecureEdgeTerminationPolicy }}
    {{- end }}
  {{- end }}
{{- end }}
```

**Values for Route**:

```yaml
jaegerRoute:
  enabled: true  # Set to false in production for security
  host: ""  # Auto-generated by OpenShift if not specified
  tls:
    enabled: true
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

## Known Issues and Gotchas

### Issue: Pod fails with "unable to create user: permission denied"
- **Problem**: Explicit `runAsUser` set in podSecurityContext conflicts with OpenShift's namespace UID range
- **Solution**: Leave `runAsUser`, `runAsGroup`, and `fsGroup` empty (see Security Context section above)

### Issue: Traces not appearing in Jaeger UI
- **Problem**: Application sending traces to wrong endpoint or format
- **Solution**: 
  - Verify OTLP endpoint: `http://<release-name>-jaeger-query:4318` for HTTP, `:4317` for gRPC
  - Check application uses OpenTelemetry SDK with OTLP exporter
  - Test endpoint: `curl http://<service>:4318/v1/traces` (should return 405 Method Not Allowed, confirming endpoint is alive)

### Issue: Jaeger pod evicted due to memory pressure
- **Problem**: Memory storage fills up with traces in high-traffic scenarios
- **Solution**: 
  - Increase memory limits in `jaeger.allInOne.resources.limits.memory`
  - Configure trace sampling rate in application instrumentation (e.g., sample 10% of requests)
  - Consider Elasticsearch backend for production

### Issue: Traces lost after pod restart
- **Problem**: Using `storage.type: memory` (ephemeral)
- **Solution**: For production, use persistent storage:
  ```yaml
  jaeger:
    storage:
      type: elasticsearch  # or cassandra
    elasticsearch:
      host: my-elasticsearch:9200
  ```

## Dependencies

None - Jaeger is a standalone component. Applications integrate by:
1. Installing OpenTelemetry SDK
2. Configuring OTLP exporter to point to Jaeger endpoint
3. Instrumenting code with tracing spans

## Testing Notes

Verify Jaeger is running and accessible:

```bash
# Check Jaeger pod status
oc get pods -n $NAMESPACE | grep jaeger

# Check Service created
oc get svc -n $NAMESPACE | grep jaeger

# Port-forward to access UI locally
oc port-forward -n $NAMESPACE svc/pdf-to-podcast-jaeger-query 16686:16686
# Then visit http://localhost:16686 in browser

# Test OTLP endpoint from another pod
oc exec -n $NAMESPACE deployment/api-service -- curl -v http://pdf-to-podcast-jaeger-query:4318/v1/traces
# Expected: 405 Method Not Allowed (confirms endpoint is live)
```

**Verify traces are being collected**:

1. Access Jaeger UI via Route or port-forward
2. Select service from dropdown (e.g., "api-service")
3. Click "Find Traces"
4. Should see traces from instrumented services

## Production Considerations

### Security

- **Disable external Route in production**: Set `jaegerRoute.enabled: false` to prevent unauthorized access to traces
- **Use internal access only**: Access Jaeger UI via `oc port-forward` or internal tools
- **Implement authentication**: Official Jaeger chart v3.x doesn't include auth - consider using OAuth proxy or network policies

### Performance and Scale

**For moderate scale (< 1000 traces/min)**:
- All-in-one deployment with memory storage is sufficient
- Increase memory limits as needed

**For high scale (> 1000 traces/min)**:
- Use distributed deployment with separate collector, query, and storage
- Deploy Elasticsearch backend for persistence
- Configure trace sampling in applications (sample 1-10% of requests)

**Values for production scale**:

```yaml
jaeger:
  allInOne:
    enabled: false
  collector:
    enabled: true
    replicas: 3
  query:
    enabled: true
    replicas: 2
  storage:
    type: elasticsearch
  elasticsearch:
    host: elasticsearch:9200
    user: jaeger
    password: <from-secret>
```

### Storage Options

| Storage Type | Persistence | Scale | Use Case |
|--------------|-------------|-------|----------|
| **memory** | No (ephemeral) | Low | Dev/test, debugging |
| **badger** | Yes (local disk) | Low-Medium | Single-node production |
| **elasticsearch** | Yes (distributed) | High | Multi-node production |
| **cassandra** | Yes (distributed) | Very High | Large-scale production |

## Alternative: Red Hat OpenShift distributed tracing platform

For enterprise production, consider **Red Hat OpenShift distributed tracing platform** (based on Jaeger):
- Fully supported by Red Hat
- Integrated with OpenShift Console
- Includes monitoring and alerting
- Operator-based lifecycle management

**Migration path**:
1. Start with official Jaeger chart for development
2. Validate application instrumentation works
3. Migrate to Red Hat distributed tracing platform for production

## Related Patterns

- [[helm-subchart-integration]] - General pattern for using subcharts
- [[security-contexts-scc]] - Security context requirements for OpenShift
- [[opentelemetry-instrumentation]] - Application instrumentation patterns (if exists)
