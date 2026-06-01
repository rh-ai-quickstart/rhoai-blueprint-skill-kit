---
name: pod-affinity-rwo-pvc
description: Pod affinity pattern for sharing ReadWriteOnce PVCs between multiple pods on the same node
summary: "Solves the \"Multi-Attach error\" when multiple pods need to share a ReadWriteOnce PVC that can only attach to one node, preventing pods from mounting the same PVC across different nodes. Use for temp/ephemeral data sharing when RWX storage is unavailable or expensive; skip if pods require independent horizontal scaling or HA across nodes (both scale limitations). Add matching custom label (e.g., `pdf-temp-volume: shared`) to all sharing pods, then configure `podAffinity.requiredDuringSchedulingIgnoredDuringExecution` with `labelSelector` matching that label and `topologyKey: kubernetes.io/hostname` to force co-location. Replicas limited to 1 per deployment (cannot scale independently), node failure kills all affinity-linked pods, pods stuck pending if node lacks resources to fit all pods, and file permission conflicts between different UIDs require fsGroup (leave empty on OpenShift for auto-range assignment from namespace)."
metadata:
  type: resource-pattern
components: []
deployment_types: [helm]
resource_types: [storage]
architecture: []
source_examples:
  - blueprint: "pdf-to-podcast"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/pdf-to-podcast"
    fork_repo: "https://github.com/rh-ai-quickstart/pdf-to-podcast"
    notes: "Pod affinity ensures pdf-api and celery-worker share RWO PVC for temp storage"
    approach: "A"
---

# Pod Affinity for Sharing ReadWriteOnce PVCs

## Overview

**ReadWriteOnce (RWO)** PVCs can only be mounted by pods on a **single node**. When multiple pods need to share the same PVC, you must ensure they are scheduled to the same node using **pod affinity**. This pattern shows how to configure pod affinity to enable PVC sharing without requiring ReadWriteMany (RWX) storage.

## When to Use This Pattern

**Use this pattern when:**
- Multiple pods need to read/write the same PVC
- Your cluster doesn't have ReadWriteMany (RWX) storage class
- RWX storage is expensive or slow (e.g., NFS)
- The pods can operate correctly when co-located on one node
- Temporary/ephemeral data sharing (like temp file processing)

**Don't use this pattern when:**
- Pods need to scale independently across nodes (requires RWX)
- High availability requires pods on different nodes
- Network latency between pods matters (affinity increases network locality)

## Common Use Cases

1. **Shared temp storage**: Multiple services processing files through a shared directory
2. **Upload/download workflows**: One service receives uploads, another processes them
3. **Batch processing**: API service writes jobs, worker service reads them from shared volume
4. **Coordinated services**: Services that need direct filesystem access to shared artifacts

## The Problem

**What doesn't work - naive approach:**

```yaml
# Pod 1
volumeMounts:
  - name: shared-temp
    mountPath: /tmp/shared
volumes:
  - name: shared-temp
    persistentVolumeClaim:
      claimName: temp-storage  # RWO PVC

---
# Pod 2
volumeMounts:
  - name: shared-temp
    mountPath: /tmp/shared
volumes:
  - name: shared-temp
    persistentVolumeClaim:
      claimName: temp-storage  # Same RWO PVC
```

**Why it fails:**
- Both pods try to mount the same RWO PVC
- Kubernetes scheduler places Pod 1 on Node A
- Pod 2 might be scheduled to Node B
- RWO PVC can't attach to Node B (already attached to Node A)
- Pod 2 remains in `ContainerCreating` state with error: "Volume is already exclusively attached to one node"

## The Solution: Pod Affinity with Label Matching

### Step 1: Create the Shared PVC

```yaml
# templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pdf-temp
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "pdf-to-podcast.labels" . | nindent 4 }}
    app: pdf-processing
spec:
  accessModes:
    - ReadWriteOnce  # Only attachable to one node
  resources:
    requests:
      storage: 10Gi
  {{- if .Values.pdfTempStorage.storageClass }}
  storageClassName: {{ .Values.pdfTempStorage.storageClass }}
  {{- end }}
```

**Configuration in values.yaml:**

```yaml
pdfTempStorage:
  size: 10Gi
  storageClass: ""  # Use cluster default
  accessMode: ReadWriteOnce
```

### Step 2: Label Pods that Share the PVC

Add a **custom label** to all pods that will mount the same PVC:

```yaml
# templates/pdf-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pdf-api
  namespace: {{ .Values.global.namespace }}
spec:
  replicas: 1  # Must be 1 when using RWO with affinity
  selector:
    matchLabels:
      app: pdf-api
  template:
    metadata:
      labels:
        app: pdf-api
        pdf-temp-volume: shared  # Custom label for affinity matching
    spec:
      # ... rest of spec
```

```yaml
# templates/celery-worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
  namespace: {{ .Values.global.namespace }}
spec:
  replicas: 1  # Must be 1 when using RWO with affinity
  selector:
    matchLabels:
      app: celery-worker
  template:
    metadata:
      labels:
        app: celery-worker
        pdf-temp-volume: shared  # Same label value as pdf-api
    spec:
      # ... rest of spec
```

**Key points:**
- Both pods have label `pdf-temp-volume: shared`
- Label name is arbitrary (choose something meaningful)
- Label value must match across all sharing pods

### Step 3: Configure Pod Affinity

Add **required pod affinity** to ensure pods are scheduled to the same node:

```yaml
# templates/pdf-api-deployment.yaml
spec:
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: pdf-temp-volume
                operator: In
                values:
                - shared
            topologyKey: kubernetes.io/hostname
      containers:
      - name: pdf-api
        # ... container spec
        volumeMounts:
        - name: pdf-temp
          mountPath: /tmp/pdf_conversions
      volumes:
      - name: pdf-temp
        persistentVolumeClaim:
          claimName: pdf-temp
```

```yaml
# templates/celery-worker-deployment.yaml
spec:
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: pdf-temp-volume
                operator: In
                values:
                - shared
            topologyKey: kubernetes.io/hostname
      containers:
      - name: celery-worker
        # ... container spec
        volumeMounts:
        - name: pdf-temp
          mountPath: /tmp/pdf_conversions
      volumes:
      - name: pdf-temp
        persistentVolumeClaim:
          claimName: pdf-temp
```

**Pod affinity explanation:**
- `requiredDuringSchedulingIgnoredDuringExecution`: Hard requirement (pod won't schedule without it)
- `labelSelector.matchExpressions`: Find pods with label `pdf-temp-volume: shared`
- `topologyKey: kubernetes.io/hostname`: Co-locate on the same **hostname** (node)

**Alternative topology keys:**
- `kubernetes.io/hostname` - Same physical/virtual node (most common)
- `topology.kubernetes.io/zone` - Same availability zone (too broad for RWO)
- Custom topology key - Depends on cluster setup

### Step 4: Mount the PVC in Both Pods

Both pods mount the same PVC at the same path:

```yaml
volumeMounts:
  - name: pdf-temp
    mountPath: /tmp/pdf_conversions

volumes:
  - name: pdf-temp
    persistentVolumeClaim:
      claimName: pdf-temp  # Same PVC name
```

**Result:**
- pdf-api writes files to `/tmp/pdf_conversions/`
- celery-worker reads files from `/tmp/pdf_conversions/`
- Both access the same filesystem on the same node

## Complete Example: PDF Processing Pipeline

**Use case:** PDF upload service (pdf-api) writes files to shared temp storage, Celery worker picks them up for processing.

**PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pdf-temp
  namespace: pdf-to-podcast
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**pdf-api Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pdf-api
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: pdf-api
        pdf-temp-volume: shared
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: pdf-temp-volume
                operator: In
                values:
                - shared
            topologyKey: kubernetes.io/hostname
      containers:
      - name: pdf-api
        image: pdf-api:latest
        env:
        - name: TEMP_FILE_DIR
          value: "/tmp/pdf_conversions"
        volumeMounts:
        - name: pdf-temp
          mountPath: /tmp/pdf_conversions
      volumes:
      - name: pdf-temp
        persistentVolumeClaim:
          claimName: pdf-temp
```

**celery-worker Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: celery-worker
        pdf-temp-volume: shared
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: pdf-temp-volume
                operator: In
                values:
                - shared
            topologyKey: kubernetes.io/hostname
      containers:
      - name: celery-worker
        image: celery-worker:latest
        env:
        - name: TEMP_FILE_DIR
          value: "/tmp/pdf_conversions"
        volumeMounts:
        - name: pdf-temp
          mountPath: /tmp/pdf_conversions
      volumes:
      - name: pdf-temp
        persistentVolumeClaim:
          claimName: pdf-temp
```

**Workflow:**
1. User uploads PDF via pdf-api → saved to `/tmp/pdf_conversions/upload_123.pdf`
2. Celery task triggered with file path
3. celery-worker reads `/tmp/pdf_conversions/upload_123.pdf` (same filesystem)
4. Worker processes PDF, writes result to `/tmp/pdf_conversions/result_123.md`
5. pdf-api reads result and returns to user

## Scaling Limitations

**With RWO + Pod Affinity:**
- **Each sharing group limited to 1 replica per deployment**
- Cannot scale pdf-api or celery-worker independently
- All pods sharing the PVC run on the same node

**Scaling options:**

### Option 1: Vertical Scaling
Scale resources up on the single node:
```yaml
celeryWorker:
  replicas: 1
  resources:
    requests:
      cpu: 4000m
      memory: 16Gi
```

### Option 2: Partition Workloads
Create multiple independent PVC groups:
```yaml
# Group 1: pdf-temp-1 PVC
pdf-api-1 + celery-worker-1 (affinity label: pdf-temp-volume: group-1)

# Group 2: pdf-temp-2 PVC
pdf-api-2 + celery-worker-2 (affinity label: pdf-temp-volume: group-2)

# Load balancer distributes between pdf-api-1 and pdf-api-2
```

### Option 3: Migrate to RWX
If you need true independent scaling, use ReadWriteMany:
```yaml
spec:
  accessModes:
    - ReadWriteMany  # Can be mounted by pods on multiple nodes
```

Remove pod affinity (no longer needed).

**RWX trade-offs:**
- ✅ Pods can scale independently
- ✅ High availability (pods on different nodes)
- ❌ Requires RWX-capable storage (NFS, CephFS, Portworx)
- ❌ May have higher latency
- ❌ Often more expensive

## Known Issues and Gotchas

### Issue: Pod stuck in `Pending` state
**Cause:** No node has enough resources to fit all affinity-linked pods.

**Solution:**
- Check node capacity: `oc describe nodes | grep -A 5 "Allocated resources"`
- Reduce resource requests
- Add nodes to cluster
- Use `preferredDuringSchedulingIgnoredDuringExecution` instead of `required` (soft affinity)

### Issue: Second pod fails with "Multi-Attach error"
**Cause:** Pod affinity not configured correctly, pods scheduled to different nodes.

**Solution:**
- Verify both pods have the matching label (`pdf-temp-volume: shared`)
- Verify affinity block is in both deployments
- Check `topologyKey: kubernetes.io/hostname` (not zone)

### Issue: All pods go down when node fails
**Cause:** All affinity-linked pods are on the same node (by design).

**Solution:**
- For critical services, use RWX instead of RWO + affinity
- Implement failover: detect node failure, provision new PVC, restart pods
- Accept the risk for non-critical temporary data

### Issue: Pods scheduled but PVC won't attach
**Cause:** PVC already attached to a different node from previous deployment.

**Solution:**
- Delete all pods mounting the PVC
- Wait for PVC to detach (may take 1-2 minutes)
- Redeploy

**Quick fix:**
```bash
oc delete pod -l pdf-temp-volume=shared -n $NAMESPACE
oc get volumeattachment  # Check PVC detached
# Wait until no volumeattachment for the PVC
oc rollout restart deployment/pdf-api -n $NAMESPACE
```

### Issue: File permissions conflict between pods
**Cause:** Pods running as different UIDs write files with different ownership.

**Solution:** Use `fsGroup` security context to share group ownership:

```yaml
spec:
  template:
    spec:
      securityContext:
        fsGroup: 1000  # Shared GID for all pods
      containers:
      - name: pdf-api
        securityContext:
          runAsUser: 1001  # Different UIDs OK
```

Files created will be group-writable with GID 1000.

**OpenShift note:** Leave `fsGroup` empty - OpenShift assigns from namespace range:

```yaml
securityContext:
  fsGroup:  # Empty - let OpenShift assign
```

## Performance Considerations

### Advantages of RWO + Affinity

1. **Lower latency**: Filesystem access is local (no network)
2. **Higher throughput**: Local disk I/O typically faster than NFS
3. **Simpler setup**: Don't need to provision RWX storage class

### Disadvantages

1. **No horizontal scaling**: Replicas limited to 1
2. **Single point of failure**: Node failure takes down all pods
3. **Resource contention**: All pods compete for same node's CPU/memory

### When to Choose RWO + Affinity vs RWX

| Requirement | RWO + Affinity | RWX |
|-------------|----------------|-----|
| **Horizontal scaling** | ❌ Limited to 1 replica | ✅ Scale independently |
| **High availability** | ❌ All pods on one node | ✅ Pods on different nodes |
| **Performance** | ✅ Local disk speed | ⚠️ Network storage (slower) |
| **Simplicity** | ✅ No special storage needed | ❌ Requires RWX storage class |
| **Cost** | ✅ Standard block storage | ⚠️ RWX often more expensive |

**Rule of thumb:**
- **Use RWO + Affinity** for: Temp storage, ephemeral data, development, single-instance workloads
- **Use RWX** for: Production services requiring HA, horizontally scaled workloads, critical data

## Testing and Verification

### Verify Pod Affinity is Working

```bash
# Get pod node assignments
oc get pods -n $NAMESPACE -o wide | grep -E 'pdf-api|celery-worker'

# Should show:
# pdf-api-xxx       1/1  Running  ...  worker-node-01
# celery-worker-xxx 1/1  Running  ...  worker-node-01
# (same node name)
```

### Verify PVC is Shared

```bash
# Exec into pdf-api, write a file
oc exec -n $NAMESPACE deployment/pdf-api -- touch /tmp/pdf_conversions/test.txt

# Exec into celery-worker, check file exists
oc exec -n $NAMESPACE deployment/celery-worker -- ls -la /tmp/pdf_conversions/test.txt

# Should show the file (proves same filesystem)
```

### Verify PVC Attachment

```bash
# Check volume attachments
oc get volumeattachment | grep pdf-temp

# Should show ONE attachment (to one node only)
```

## Related Patterns

- [[rhoai-pvc-initialization]] - PVC initialization pattern for RHOAI notebooks
- [[security-contexts-scc]] - Security context requirements for shared storage
- [[gpu-allocation-openshift]] - Node affinity patterns for GPU workloads
