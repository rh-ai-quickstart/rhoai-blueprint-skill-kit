---
name: minio-on-rhoai
description: MinIO object storage deployment on RHOAI using Red Hat AI Quickstart subchart
metadata:
  type: component
components: [minio]
deployment_types: [helm]
resource_types: [storage]
architecture: []
source_examples:
  - blueprint: "pdf-to-podcast"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/pdf-to-podcast"
    fork_repo: "https://github.com/rh-ai-quickstart/pdf-to-podcast"
    notes: "MinIO using Red Hat AI Quickstart subchart for RHOAI ecosystem integration"
    approach: "A"
---

# MinIO on RHOAI

## Overview

MinIO is an S3-compatible object storage system commonly used for storing ML artifacts, audio files, PDFs, and other binary data. This pattern shows how to deploy MinIO using the **Red Hat AI Quickstart subchart**, which is certified for RHOAI and follows OpenShift best practices.

## Conversion Pattern

### Deployment Type: Helm Subchart

MinIO is deployed as a **Helm dependency** (subchart) rather than a standalone deployment. This leverages certified, maintained charts from the RHOAI ecosystem.

### Helm Chart Dependency

**Chart.yaml dependency declaration**:

```yaml
dependencies:
  - name: minio
    version: "0.5.x"
    repository: "https://rh-ai-quickstart.github.io/ai-architecture-charts"
    condition: minio.enabled
```

**Install dependencies**:

```bash
cd helm/
helm dependency update
```

This downloads the subchart to `helm/charts/minio-*.tgz`.

### Values Configuration

**Parent chart values.yaml** (configures MinIO subchart):

```yaml
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
    apiPort: 9000  # S3 API (what services connect to)
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
```

### Service Discovery

**MinIO service is created automatically by the subchart**. Application services connect using:

```yaml
env:
  - name: MINIO_ENDPOINT
    value: "minio:9000"  # S3 API endpoint
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
```

**MinIO connection URL format**:
- **S3 API**: `http://minio:9000` (used by application services)
- **Console UI**: `http://minio:9090` (admin interface)

### Automatic Bucket Creation

The Red Hat AI Quickstart MinIO chart **does not auto-create buckets**. Applications must create buckets programmatically on startup.

**Python example** (from pdf-to-podcast APIService):

```python
from minio import Minio
import os

minio_client = Minio(
    os.getenv("MINIO_ENDPOINT", "minio:9000"),
    access_key=os.getenv("MINIO_ACCESS_KEY", "minioadmin"),
    secret_key=os.getenv("MINIO_SECRET_KEY", "minioadmin"),
    secure=False  # Use True for HTTPS in production
)

# Create bucket if it doesn't exist
bucket_name = "audio-results"
if not minio_client.bucket_exists(bucket_name):
    minio_client.make_bucket(bucket_name)
```

### Storage Configuration

MinIO uses **volumeClaimTemplates** for persistent storage:

```yaml
volumeClaimTemplates:
  - metadata:
      name: minio-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Gi
      # Optional: specify storageClass
      # storageClassName: my-storage-class
```

**Default behavior**:
- Uses cluster's default StorageClass if not specified
- ReadWriteOnce access mode (single node attachment)
- PVC is managed by the MinIO StatefulSet

### External Access via Route (Optional)

To expose MinIO Console UI externally:

**Parent chart Route template**:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-console
  namespace: {{ .Values.global.namespace }}
spec:
  to:
    kind: Service
    name: minio
  port:
    targetPort: 9090  # Console UI port
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**Note**: The subchart creates the Service; you only need to create the Route in your parent chart.

## Known Issues and Gotchas

### Issue: Bucket not found errors on first startup
- **Problem**: Application tries to access MinIO bucket before it's created
- **Solution**: Add bucket creation logic to application startup code (see "Automatic Bucket Creation" above)

### Issue: Credentials not passed correctly
- **Problem**: MinIO subchart expects specific secret structure
- **Solution**: Use subchart's `.Values.minio.secret` format with `user`, `password`, `host`, `port` keys

### Issue: Port confusion (9000 vs 9090)
- **Problem**: Connecting to wrong port (Console UI vs S3 API)
- **Solution**: 
  - **Port 9000** = S3 API (for boto3, minio-py, AWS SDK)
  - **Port 9090** = Console web UI (for browser access)
  - Applications should always use port 9000

### Issue: Storage class not available
- **Problem**: Cluster doesn't have a default StorageClass
- **Solution**: Explicitly set `volumeClaimTemplates[0].spec.storageClassName` in values.yaml

## Dependencies

None - MinIO is a standalone service. However, it's typically used by:
- API services for file uploads/downloads
- ML pipelines for artifact storage
- TTS services for audio file storage

## Testing Notes

Verify MinIO is running and accessible:

```bash
# Check MinIO pod status
oc get pods -n $NAMESPACE | grep minio

# Check PVC created
oc get pvc -n $NAMESPACE | grep minio

# Port-forward to access Console UI locally
oc port-forward -n $NAMESPACE svc/minio 9090:9090
# Then visit http://localhost:9090 in browser

# Test S3 API from another pod
oc exec -n $NAMESPACE deployment/api-service -- curl http://minio:9000/minio/health/live
# Expected: HTTP 200
```

**Test bucket creation** from a Python pod:

```bash
oc exec -n $NAMESPACE deployment/api-service -- python3 -c "
from minio import Minio
client = Minio('minio:9000', access_key='minioadmin', secret_key='minioadmin', secure=False)
print('Buckets:', list(client.list_buckets()))
"
```

## Production Considerations

### Security

- **Change default credentials**: Override `minio.secret.user` and `minio.secret.password` in production
- **Enable TLS**: Set `secure=True` in MinIO client connections and configure TLS certificates
- **Use Secrets**: Store credentials in OpenShift Secrets, reference via `secretKeyRef`

### High Availability

For production HA, consider:
- **Multi-node MinIO**: Red Hat AI Quickstart chart supports distributed mode
- **Increase replicas**: Set `.Values.minio.replicas` for fault tolerance
- **Use dedicated storage**: Configure high-performance StorageClass

### Storage Sizing

Estimate based on use case:
- **Audio files**: ~1-5 MB per podcast episode
- **PDFs**: ~1-10 MB per document
- **ML models**: 100 MB - 10 GB per model
- **Add 30% buffer** for growth

Example for 1000 podcasts:
```yaml
volumeClaimTemplates:
  - spec:
      resources:
        requests:
          storage: 10Gi  # 1000 podcasts × 5 MB × 1.3 buffer ≈ 6.5 GB → round to 10 GB
```

## Alternative: OpenShift Data Foundation

For enterprise production, consider **OpenShift Data Foundation (ODF)** instead of MinIO:
- Native S3-compatible object storage
- Integrated with OpenShift
- Advanced HA/DR capabilities
- Red Hat support

**Migration path**:
1. Start with MinIO subchart for development
2. Validate application works
3. Migrate to ODF for production if advanced features needed

## Related Patterns

- [[helm-subchart-integration]] - General pattern for using subcharts
- [[security-contexts-scc]] - Security context requirements
- [[rhoai-pvc-initialization]] - PVC initialization patterns
