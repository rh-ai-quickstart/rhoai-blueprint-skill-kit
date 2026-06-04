---
name: helm-with-subcharts
description: Helm chart deployment pattern using subcharts for infrastructure components (Redis, MinIO, Jaeger) on RHOAI
summary: "Use this pattern when converting docker-compose-based NVIDIA Blueprints to OpenShift by creating a standalone openshift/ directory with a Helm chart that leverages certified subcharts for infrastructure (Redis, MinIO, Jaeger) while deploying custom application services as templates; use helm-openshift-conditionals instead if the blueprint already has Helm, or oc-apply manifests for simple blueprints. Critical Chart.yaml structure includes conditional dependencies (condition: component.enabled) with version ranges (0.5.x), declares subcharts in dependencies array with repository URLs (e.g., minio from rh-ai-quickstart, jaeger from official helm-charts), and auto-generates Chart.lock for reproducible builds; run helm dependency update to download subcharts to helm/charts/ directory (.gitignore *.tgz files). Override subchart security contexts to avoid OpenShift SCC conflicts by setting podSecurityContext.runAsUser to empty value (not explicit UID like 1000), explicitly specify storageClassName in volumeClaimTemplates if cluster lacks default, and structure values.yaml with global settings first, then infrastructure components with inline documentation, then custom services. Common failures include forgetting helm dependency update (causes \"missing dependencies\" error), subchart runAsUser conflicts with namespace UID range (override to empty), and subchart PVCs pending due to missing StorageClass declaration; prefer Red Hat certified charts over Bitnami for guaranteed OpenShift compatibility (**always use `latest` tag for Bitnami images**), but use custom templates for simple components like ephemeral Redis."
metadata:
  type: deployment
components: [redis, minio, jaeger]
deployment_types: [helm]
resource_types: []
architecture: []
source_examples:
  - blueprint: "pdf-to-podcast"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/pdf-to-podcast"
    fork_repo: "https://github.com/rh-ai-quickstart/pdf-to-podcast"
    notes: "Standalone OpenShift Helm chart with certified subcharts for infrastructure"
    approach: "A"
---

# Helm Deployment with Subcharts Pattern

## Overview

This pattern describes how to structure a Helm chart for RHOAI that leverages **certified subcharts** for infrastructure components (Redis, MinIO, Jaeger) while deploying custom application services as standalone templates. This approach balances:
- **Reuse**: Certified charts handle infrastructure complexity
- **Control**: Custom templates for application-specific services
- **Maintainability**: Subcharts receive upstream updates

This pattern is used when converting docker-compose-based NVIDIA Blueprints to OpenShift, creating a **standalone Helm chart** in an `openshift/` directory without modifying the original deployment method.

## When to Use This Pattern

**Use this pattern when:**
- Original blueprint uses docker-compose (not Helm)
- Blueprint includes standard infrastructure (Redis, MinIO, Jaeger, etc.)
- Creating a separate OpenShift deployment alongside original
- Want to leverage certified/maintained charts for infrastructure
- Need clean separation between infrastructure and application services

**Don't use this pattern when:**
- Original blueprint already has a Helm chart (use [[helm-openshift-conditionals]] instead)
- Blueprint is simple enough for oc-apply manifests
- All components are custom (no reusable infrastructure)

## Directory Structure

```
openshift/
├── README.md                         # Deployment documentation
├── deploy-helm.sh                    # One-command deployment script
├── undeploy-helm.sh                  # Cleanup script
├── build-images.sh                   # BuildConfig automation
├── secrets.env.template              # API keys template
├── frontend/                         # Custom Dockerfiles
│   └── Dockerfile
└── helm/                             # Helm chart
    ├── Chart.yaml                    # Chart metadata + subchart dependencies
    ├── Chart.lock                    # Locked dependency versions
    ├── values.yaml                   # Default configuration
    ├── values-prod.yaml              # Production overrides
    ├── charts/                       # Downloaded subcharts (gitignored)
    │   ├── minio-*.tgz
    │   └── jaeger-*.tgz
    └── templates/                    # K8s manifests
        ├── _helpers.tpl              # Template helper functions
        ├── namespace.yaml            # Namespace (optional)
        ├── secrets.yaml              # API keys secret
        ├── configmaps.yaml           # Shared config
        ├── pvc.yaml                  # Shared PVCs
        ├── redis-deployment.yaml     # Custom Redis (standalone)
        ├── redis-service.yaml
        ├── api-service-deployment.yaml      # Custom app services
        ├── agent-service-deployment.yaml
        ├── pdf-service-deployment.yaml
        ├── pdf-api-deployment.yaml
        ├── celery-worker-deployment.yaml
        ├── tts-service-deployment.yaml
        ├── frontend-deployment.yaml
        ├── services.yaml             # All app Services in one file
        └── routes.yaml               # OpenShift Routes
```

## Chart.yaml Structure

**Defines metadata and subchart dependencies**:

```yaml
apiVersion: v2
name: pdf-to-podcast
description: A Helm chart for deploying PDF-to-Podcast application on OpenShift
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - pdf
  - podcast
  - nvidia
  - llm
  - tts
  - openshift
  - rhoai

# Infrastructure dependencies
# Redis: Standalone deployment using official Redis image (see templates/redis-*.yaml)
# MinIO: Red Hat AI Quickstart chart (uses official MinIO image)
# Jaeger: Official Jaeger chart for distributed tracing
# For enterprise production, consider migrating to Operators from OperatorHub.
dependencies:
  - name: minio
    version: "0.5.x"
    repository: "https://rh-ai-quickstart.github.io/ai-architecture-charts"
    condition: minio.enabled
  - name: jaeger
    version: "3.x.x"
    repository: "https://jaegertracing.github.io/helm-charts"
    condition: jaeger.enabled
```

**Key points**:
- **Conditional dependencies**: Use `condition: <component>.enabled` so users can disable components
- **Version ranges**: `0.5.x` pins major/minor but allows patch updates
- **Documentation in comments**: Explain why each subchart is used and alternatives

## Values.yaml Structure

**Organized by component with clear sections**:

```yaml
# Global settings (shared across all components)
global:
  namespace: pdf-to-podcast
  createNamespace: false  # Set to true to create namespace
  security:
    allowInsecureImages: true  # For Quay.io registry

# Image registry (OpenShift internal registry)
imageRegistry: image-registry.openshift-image-registry.svc:5000/pdf-to-podcast

# Image pull policy
imagePullPolicy: Always

# API Keys (passed via --set during deployment)
apiKeys:
  nvidia: ""
  elevenlabs: ""

#############################################################################
# Redis configuration (standalone deployment using official Redis image)
# Matches docker-compose setup: redis:latest with no persistence
#############################################################################
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

#############################################################################
# MinIO configuration (using Red Hat AI Quickstart chart)
# Docs: https://github.com/rh-ai-quickstart/ai-architecture-charts
# Note: Application creates 'audio-results' bucket automatically on startup
#############################################################################
minio:
  enabled: true
  secret:
    user: minioadmin
    password: minioadmin  # CHANGE IN PRODUCTION
    host: minio
    port: "9000"
  service:
    type: ClusterIP
    port: 9090  # Console UI
    apiPort: 9000  # S3 API
  volumeClaimTemplates:
    - metadata:
        name: minio-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 2Gi

#############################################################################
# Jaeger configuration (using official Jaeger chart v3.x)
#############################################################################
jaeger:
  enabled: true
  allInOne:
    enabled: true
    podSecurityContext:
      runAsUser:        # Empty: let OpenShift assign
      runAsGroup:
      fsGroup:
      seccompProfile:
        type: RuntimeDefault
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
  storage:
    type: memory

# Application services (custom deployments)
celeryWorker:
  image:
    tag: "latest"
  replicas: 1
  gpu:
    enabled: false
    count: 1
  resources:
    requests:
      cpu: 1000m
      memory: 8Gi
    limits:
      cpu: 2000m
      memory: 10Gi

# ... similar blocks for other app services ...

# OpenShift Routes
frontendRoute:
  enabled: true
  host: ""  # Auto-generated if empty
  tls:
    enabled: true
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**Key organizational principles**:
1. **Global settings first** - namespace, security, image registry
2. **Infrastructure components** - Redis, MinIO, Jaeger with inline documentation
3. **Application services** - custom microservices
4. **Networking** - Routes configuration
5. **Comments as documentation** - explain non-obvious choices inline

## Dependency Management

**Install/update subcharts**:

```bash
cd helm/
helm dependency update
```

This downloads subcharts to `helm/charts/` directory.

**Chart.lock file** (auto-generated):

```yaml
dependencies:
- name: minio
  repository: https://rh-ai-quickstart.github.io/ai-architecture-charts
  version: 0.5.3
- name: jaeger
  repository: https://jaegertracing.github.io/helm-charts
  version: 3.3.1
digest: sha256:abc123...
generated: "2024-01-15T10:30:00Z"
```

**Commit Chart.lock to git** - ensures reproducible builds.

**.gitignore entry**:

```
helm/charts/*.tgz
```

Don't commit downloaded subcharts - they're regenerated from Chart.lock.

## Template Patterns

### Helper Functions (_helpers.tpl)

**Standard Helm helpers**:

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "pdf-to-podcast.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "pdf-to-podcast.labels" -}}
helm.sh/chart: {{ include "pdf-to-podcast.chart" . }}
{{ include "pdf-to-podcast.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

**Custom helpers for service discovery**:

```yaml
{{/*
Redis host for connection strings
*/}}
{{- define "pdf-to-podcast.redisHost" -}}
redis
{{- end }}

{{/*
Jaeger OTLP endpoint for application instrumentation
*/}}
{{- define "pdf-to-podcast.jaegerOtlpEndpoint" -}}
http://{{ .Release.Name }}-jaeger-query:4318
{{- end }}

{{/*
Image pull policy (IfNotPresent for tags, Always for 'latest')
*/}}
{{- define "pdf-to-podcast.imagePullPolicy" -}}
{{- if eq .Values.imagePullPolicy "Always" }}
Always
{{- else }}
IfNotPresent
{{- end }}
{{- end }}
```

### Custom Infrastructure Templates

**Redis (standalone, not subchart)**:

```yaml
# templates/redis-deployment.yaml
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

**Why standalone Redis instead of subchart?**
- Simpler for ephemeral caching use case
- No persistence needed (Celery tasks can retry)
- Avoids Bitnami subchart complexity
- Matches docker-compose behavior exactly

### Application Service Templates

**Pattern for custom microservices**:

```yaml
# templates/api-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "pdf-to-podcast.labels" . | nindent 4 }}
    app: api-service
spec:
  replicas: {{ .Values.apiService.replicas }}
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api-service
        image: "{{ .Values.imageRegistry }}/api-service:{{ .Values.apiService.image.tag }}"
        imagePullPolicy: {{ include "pdf-to-podcast.imagePullPolicy" . }}
        ports:
        - containerPort: 8002
          name: http
        env:
        - name: REDIS_URL
          value: "redis://{{ include "pdf-to-podcast.redisHost" . }}:6379"
        - name: MINIO_ENDPOINT
          value: "minio:9000"
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: user
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: password
        resources:
          {{- toYaml .Values.apiService.resources | nindent 10 }}
        livenessProbe:
          httpGet:
            path: /health
            port: 8002
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8002
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Consolidated Services Template

**All Services in one file** (reduces template clutter):

```yaml
# templates/services.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: {{ .Values.global.namespace }}
spec:
  selector:
    app: api-service
  ports:
  - port: 8002
    targetPort: 8002
    protocol: TCP
    name: http
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: agent-service
  namespace: {{ .Values.global.namespace }}
spec:
  selector:
    app: agent-service
  ports:
  - port: 8964
    targetPort: 8964
    protocol: TCP
    name: http
  type: ClusterIP
# ... more services ...
```

### Routes Template

**OpenShift Routes with conditional creation**:

```yaml
# templates/routes.yaml
{{- if .Values.frontendRoute.enabled }}
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: pdf-to-podcast-frontend
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "pdf-to-podcast.labels" . | nindent 4 }}
    app: frontend
spec:
  {{- if .Values.frontendRoute.host }}
  host: {{ .Values.frontendRoute.host }}
  {{- end }}
  to:
    kind: Service
    name: frontend
  port:
    targetPort: http
  {{- if .Values.frontendRoute.tls.enabled }}
  tls:
    termination: {{ .Values.frontendRoute.tls.termination }}
    insecureEdgeTerminationPolicy: {{ .Values.frontendRoute.tls.insecureEdgeTerminationPolicy }}
  {{- end }}
{{- end }}
```

## Deployment Automation

### deploy-helm.sh

```bash
#!/bin/bash
set -e

NAMESPACE="${OPENSHIFT_NAMESPACE:-pdf-to-podcast}"
RELEASE_NAME="pdf-to-podcast"

# Load secrets from secrets.env
if [ -f secrets.env ]; then
  source secrets.env
else
  echo "Error: secrets.env not found. Copy secrets.env.template and fill in values."
  exit 1
fi

# Create namespace if it doesn't exist
oc get namespace "$NAMESPACE" || oc create namespace "$NAMESPACE"

# Update Helm dependencies
cd helm
helm dependency update
cd ..

# Install/upgrade Helm chart
helm upgrade --install "$RELEASE_NAME" ./helm \
  --namespace "$NAMESPACE" \
  --set apiKeys.nvidia="$NVIDIA_API_KEY" \
  --set apiKeys.elevenlabs="$ELEVENLABS_API_KEY" \
  --wait \
  --timeout 10m

echo "Deployment complete!"
echo ""
echo "Frontend URL:"
oc get route pdf-to-podcast-frontend -n "$NAMESPACE" -o jsonpath='{.spec.host}' && echo
```

### build-images.sh

**OpenShift BuildConfig automation**:

```bash
#!/bin/bash
set -e

NAMESPACE="${OPENSHIFT_NAMESPACE:-pdf-to-podcast}"
SERVICES="api-service agent-service pdf-service pdf-api celery-worker tts-service frontend"

# Function to create BuildConfig for a service
build_service() {
  local SERVICE=$1
  local DOCKERFILE_PATH="services/${SERVICE}/Dockerfile"
  
  # Special case for frontend
  if [ "$SERVICE" = "frontend" ]; then
    DOCKERFILE_PATH="openshift/frontend/Dockerfile"
  fi
  
  echo "Building $SERVICE..."
  
  # Create BuildConfig
  oc process -f - <<EOF | oc apply -n "$NAMESPACE" -f -
apiVersion: v1
kind: Template
objects:
- apiVersion: build.openshift.io/v1
  kind: BuildConfig
  metadata:
    name: ${SERVICE}
  spec:
    output:
      to:
        kind: ImageStreamTag
        name: ${SERVICE}:latest
    source:
      type: Binary
      binary: {}
    strategy:
      type: Docker
      dockerStrategy:
        dockerfilePath: ${DOCKERFILE_PATH}
    triggers: []
EOF

  # Start binary build from repo root
  oc start-build "${SERVICE}" --from-dir=../ --follow -n "$NAMESPACE"
}

# Build all or specific service
if [ "$1" = "all" ]; then
  for SERVICE in $SERVICES; do
    build_service "$SERVICE"
  done
else
  build_service "$1"
fi
```

## Production Considerations

### Subchart Selection Criteria

**When choosing subcharts, prioritize**:
1. **Red Hat certified charts** - Guaranteed OpenShift compatibility (e.g., Red Hat AI Quickstart MinIO)
2. **Official upstream charts** - Direct from project maintainers (e.g., Jaeger)
3. **Bitnami charts** - Well-maintained, but may need SCC adjustments. **Always use `latest` tag** - specific Bitnami version tags are not available in free container registries
4. **Custom templates** - For simple components (e.g., standalone Redis)

### Migration Path to Operators

Helm charts are valid for production, but Red Hat recommends **Operators** for long-term enterprise deployments:

```
Development → Testing → Initial Production → Migrate to Operators (optional)
```

**When to migrate to Operators**:
- Need advanced HA/DR capabilities
- Require automated backups and restore
- Want integrated monitoring/alerting
- Prefer declarative lifecycle management

**Keep Helm charts** for:
- Development environments
- Testing environments
- Blueprints that change frequently
- Teams comfortable with Helm

## Known Issues and Gotchas

### Issue: Subchart security contexts fail on OpenShift
- **Problem**: Subchart sets explicit `runAsUser: 1000` that conflicts with namespace UID range
- **Solution**: Override in parent chart values:
  ```yaml
  jaeger:
    allInOne:
      podSecurityContext:
        runAsUser:   # Empty value
  ```

### Issue: Subchart PVCs use wrong StorageClass
- **Problem**: Cluster doesn't have default StorageClass, subchart PVC pending
- **Solution**: Explicitly set in values:
  ```yaml
  minio:
    volumeClaimTemplates:
      - spec:
          storageClassName: my-storage-class
  ```

### Issue: Subcharts not downloaded
- **Problem**: `helm install` fails with "chart requires missing dependencies"
- **Solution**: Run `helm dependency update` before install

## Related Patterns

- [[redis-on-rhoai]] - Standalone Redis deployment
- [[minio-on-rhoai]] - MinIO subchart configuration
- [[jaeger-on-rhoai]] - Jaeger subchart configuration
- [[helm-openshift-conditionals]] - Alternative pattern with conditional logic
- [[security-contexts-scc]] - Security context requirements
