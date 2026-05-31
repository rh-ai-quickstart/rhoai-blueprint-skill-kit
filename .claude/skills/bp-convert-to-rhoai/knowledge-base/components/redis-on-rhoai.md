---
name: redis-on-rhoai
description: Redis deployment on RHOAI with OpenShift-compatible security contexts and init containers
metadata:
  type: component
components: [redis]
deployment_types: [helm]
resource_types: [storage, security-context]
architecture: []
source_examples:
  - blueprint: "data-flywheel"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/data-flywheel"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-data-flywheel"
    notes: "Redis with conditional OpenShift support, restricted SCC compliance"
    approach: "A"
  - blueprint: "pdf-to-podcast"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/pdf-to-podcast"
    fork_repo: "https://github.com/rh-ai-quickstart/pdf-to-podcast"
    notes: "Standalone Redis deployment for OpenShift without conditionals, always-on restricted SCC"
    approach: "B"
---

# Redis on RHOAI

## Overview

Redis is an in-memory data store commonly used as a message broker for Celery task queues in AI/ML pipelines. This pattern shows how to deploy Redis with OpenShift conditional support using Helm templates.

## Conversion Pattern

### OPENSHIFT_MODE Conditional Support

Uses Helm template conditionals with `.Values.openshift.enabled` to switch between standard Kubernetes and OpenShift configurations.

### Deployment Type: Helm

Redis is deployed as a Kubernetes Deployment with conditional security contexts and init containers.

### Security Context Requirements

**Pod-level security context** (when `openshift.enabled=true`):
```yaml
spec:
  automountServiceAccountToken: false
  {{- if .Values.openshift.enabled }}
  securityContext:
    {{- include "data-flywheel.podSecurityContext" . | nindent 8 }}
  {{- end }}
```

**Container-level security context** (when `openshift.enabled=true`):
```yaml
containers:
  - name: redis
    image: "{{ .Values.redis.image.repository }}:{{ .Values.redis.image.tag }}"
    imagePullPolicy: Always
    {{- if .Values.openshift.enabled }}
    securityContext:
      {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
    {{- end }}
```

**Helper function definitions** - See [[mongodb-on-rhoai#Helper function definitions]]

**Values configuration**:
```yaml
openshift:
  enabled: false
  securityContext:
    pod:
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
    container:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      capabilities:
        drop:
          - ALL
```

### Storage Configuration

Uses **emptyDir** volumes for ephemeral storage with conditional init containers:

**When OpenShift enabled**:
```yaml
initContainers:
  {{- if .Values.openshift.enabled }}
  - name: create-redis-data-folder
    image: {{ .Values.openshift.initContainer.image }}
    command: ['sh', '-c', 'mkdir -p /data']
    volumeMounts:
      - name: redis-data-volume
        mountPath: /data
    securityContext:
      {{- include "data-flywheel.containerSecurityContext" . | nindent 12 }}
  {{- else }}
  - name: create-redis-data-folder
    image: busybox:1.28
    command: ['sh', '-c', 'mkdir -p /data && chmod 755 /data']
    volumeMounts:
      - name: redis-data-volume
        mountPath: /data
  {{- end }}
```

**Volume mount and volume**:
```yaml
containers:
  - name: redis
    # ... other config ...
    volumeMounts:
      - name: redis-data-volume
        mountPath: /data
volumes:
  - name: redis-data-volume
    emptyDir: {}
```

**Why different init containers?**
- **OpenShift**: Uses Red Hat UBI minimal image (`registry.access.redhat.com/ubi8/ubi-minimal:latest`) and cannot run `chmod` due to restricted SCC
- **Standard Kubernetes**: Uses `busybox:1.28` which can run `chmod` for explicit permissions

### Container Image

Uses standard Redis from container registry:
```yaml
redis:
  image:
    repository: "redis"
    tag: "latest"
```

## Known Issues and Gotchas

### Issue: chmod fails in restricted SCC
- **Problem**: Init containers cannot run `chmod` when using restricted-v2 SCC
- **Solution**: Remove `chmod` command in OpenShift mode; rely on OpenShift's automatic UID/GID assignment from namespace range

### Issue: Redis data persistence
- **Problem**: Using emptyDir means Redis data is lost when pod restarts
- **Solution**: For production, consider using PersistentVolumeClaim instead of emptyDir. This pattern uses emptyDir for task queue scenarios where data loss is acceptable (Celery tasks can be retried).

## Dependencies

None - Redis is a standalone component.

## Testing Notes

Verify Redis is running and accessible:
```bash
# Check pod status
oc get pods -n $NAMESPACE | grep redis

# Test Redis connection from another pod
oc exec -n $NAMESPACE deployment/df-api -- redis-cli -h df-redis-service ping
# Expected: PONG
```

---

## Approach B: Standalone OpenShift-Only Deployment (from pdf-to-podcast)

### When to Use

When the blueprint conversion creates a **separate OpenShift deployment infrastructure** (e.g., in an `openshift/` directory) rather than adding conditionals to existing Helm charts. This approach is preferred when:
- The original deployment uses docker-compose (not Helm)
- You want to preserve the original deployment method unchanged
- The OpenShift version is a complete standalone port

### Differences from Approach A

| Aspect | Approach A (Conditional) | Approach B (Standalone) |
|--------|-------------------------|------------------------|
| **File location** | Modifies existing Helm chart | New `openshift/helm/` directory |
| **Conditionals** | Uses `{{- if .Values.openshift.enabled }}` | No conditionals - always OpenShift |
| **Security contexts** | Applied conditionally | Always applied (hardcoded) |
| **Init containers** | Conditional based on platform | Not needed - emptyDir works directly |
| **Original deployment** | Modified with flags | Completely unchanged |

### Conversion Pattern

**Standalone deployment template** with OpenShift security contexts always applied:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "pdf-to-podcast.labels" . | nindent 4 }}
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: redis
        image: "{{ .Values.redis.image.registry }}/{{ .Values.redis.image.repository }}:{{ .Values.redis.image.tag }}"
        imagePullPolicy: {{ include "pdf-to-podcast.imagePullPolicy" . }}
        ports:
        - containerPort: 6379
          name: redis
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        resources:
          {{- toYaml .Values.redis.resources | nindent 10 }}
        volumeMounts:
        - name: redis-data
          mountPath: /data
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: redis-data
        emptyDir: {}
```

**Values configuration** (OpenShift-specific, no platform toggle):

```yaml
redis:
  enabled: true
  image:
    registry: docker.io
    repository: redis
    tag: latest
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

**Service definition**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "pdf-to-podcast.labels" . | nindent 4 }}
    app: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
    protocol: TCP
    name: redis
  type: ClusterIP
```

### Key Characteristics

1. **No init containers needed**: emptyDir works directly with restricted SCC when proper security contexts are set
2. **Security contexts hardcoded**: Always uses `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, capabilities drop ALL
3. **Health probes**: TCP socket checks for liveness and readiness
4. **Resource limits**: Defined in values.yaml with sensible defaults
5. **Namespace templating**: Uses `{{ .Values.global.namespace }}` for flexibility

### Advantages of Approach B

- **Simpler templates**: No conditional logic to maintain
- **Separation of concerns**: Original deployment (docker-compose) untouched
- **Clear intent**: Templates are obviously OpenShift-specific
- **Easier testing**: No need to test both openshift.enabled true/false paths

### Disadvantages of Approach B

- **Code duplication**: Separate Helm chart means maintaining two deployment methods
- **Divergence risk**: Bug fixes in docker-compose may not propagate to Helm chart
- **Storage limitation**: Uses emptyDir (ephemeral) only - cannot preserve data across pod restarts

---

## Choosing Between Approaches

**Use Approach A (Conditional)** when:
- Original blueprint already uses Helm
- You want a single Helm chart for both platforms
- Need to toggle between local dev and OpenShift easily
- The blueprint will be maintained upstream

**Use Approach B (Standalone)** when:
- Original blueprint uses docker-compose (not Helm)
- Creating an OpenShift-specific deployment alongside original
- Want to preserve NVIDIA's original deployment method completely
- OpenShift deployment is a separate fork/variant
- Prioritize simplicity over unification

**Pattern observed**: pdf-to-podcast uses Approach B because the original is docker-compose-based, so a standalone Helm chart is the natural OpenShift adaptation without disrupting the local development workflow.

## Related Patterns

- [[mongodb-on-rhoai]] - Similar stateful service with conditional OpenShift support
- [[security-contexts-scc]] - Security context patterns for OpenShift
- [[helm-openshift-conditionals]] - Conditional deployment patterns for Approach A
