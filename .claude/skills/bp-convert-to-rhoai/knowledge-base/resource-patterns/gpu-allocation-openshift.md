---
type: resource-pattern
components: []
deployment_types: [helm]
resource_types: [gpu]
architecture: []
source_examples:
  - blueprint: "video-search-and-summarization"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization"
    fork_repo: "https://github.com/rh-ai-quickstart/nvidia-video-search-and-summarization"
    notes: "Demonstrates GPU resource allocation and tolerations for NIM services and VSS application"
    approach: "A"
  - blueprint: "pdf-to-podcast"
    source_repo: "https://github.com/NVIDIA-AI-Blueprints/pdf-to-podcast"
    fork_repo: "https://github.com/rh-ai-quickstart/pdf-to-podcast"
    notes: "Optional GPU acceleration for Celery worker with conditional Helm values"
    approach: "B"
---

# GPU Resource Allocation on OpenShift

## Overview

OpenShift GPU nodes typically have taints that prevent non-GPU workloads from scheduling on expensive GPU hardware. This pattern shows how to configure GPU resource requests/limits and tolerations to schedule GPU workloads correctly.

## When to Use

- Workloads requiring NVIDIA GPUs (inference, training, video processing)
- Scheduling on nodes with `nvidia.com/gpu` taints
- Multi-GPU or fractional GPU allocation
- Preventing GPU workloads from running on CPU-only nodes

## Prerequisites

- NVIDIA GPU Operator installed on OpenShift cluster
- GPU nodes labeled with `nvidia.com/gpu.present=true`
- Nodes may have taints like `nvidia.com/gpu:NoSchedule`

**Verify GPU availability:**
```bash
oc get nodes -l nvidia.com/gpu.present=true
oc describe node <gpu-node> | grep -A 5 "Allocatable"
```

Expected output:
```
Allocatable:
  nvidia.com/gpu: 1  (or higher)
```

## GPU Resource Allocation Pattern

### 1. GPU Resource Requests and Limits

Specify GPU count in pod resources:

**For NIM services (via NIM Operator):**
```yaml
nimOperator:
  nim-llm:
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
```

**For standard Deployments/Pods:**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1
```

**Key Points:**
- Use `nvidia.com/gpu` resource name (not `nvidia.com/gpu.count` or variations)
- Set both `requests` and `limits` to same value for guaranteed allocation
- GPUs are whole-unit resources (no fractional allocation unless using MIG)

### 2. GPU Tolerations

Add tolerations to schedule on nodes with GPU taints:

**For NIM services:**
```yaml
nimOperator:
  nim-llm:
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

**For standard workloads:**
```yaml
spec:
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
```

**Common GPU Taints:**

| Taint Key | Effect | Operator | Meaning |
|-----------|--------|----------|---------|
| `nvidia.com/gpu` | NoSchedule | Exists | Node has GPUs, only GPU pods allowed |
| `nvidia.com/gpu` | NoExecute | Exists | Evict non-GPU pods from GPU nodes |
| Custom (e.g., `gpu-workload`) | NoSchedule | Equal:value | Cluster-specific GPU taint |

### 3. Node Selector (Optional)

Force scheduling on GPU nodes:

```yaml
nodeSelector:
  nvidia.com/gpu.present: "true"
```

**When to Use:**
- When tolerations alone don't guarantee GPU node scheduling
- When you need specific GPU types (L40S vs A100 vs H100)

**Example with GPU type:**
```yaml
nodeSelector:
  nvidia.com/gpu.product: NVIDIA-L40S
```

### 4. Example: VSS Application GPU Configuration

**values-openshift.yaml:**
```yaml
vss:
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
    # Add cluster-specific tolerations as needed:
    # - key: <your-gpu-taint-key>
    #   operator: Equal
    #   value: "true"
    #   effect: NoSchedule
  resources:
    limits:
      nvidia.com/gpu: 1
```

### 5. Example: NIM GPU Configuration

**values-openshift.yaml:**
```yaml
nimOperator:
  nim-llm:
    enabled: true
    replicas: 1
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    # Optional: specific GPU type
    nodeSelector: {}
    
  nemo-embedding:
    enabled: true
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

## Multi-GPU Allocation

### Multiple GPUs for One Pod

For models requiring tensor parallelism:

```yaml
resources:
  limits:
    nvidia.com/gpu: 4
  requests:
    nvidia.com/gpu: 4
```

**Example: LLM with 4-GPU tensor parallelism:**
```yaml
nimOperator:
  nim-llm:
    resources:
      limits:
        nvidia.com/gpu: 4
      requests:
        nvidia.com/gpu: 4
```

**Scheduling Constraint:** All GPUs must be on the same node unless using multi-node training frameworks.

### Multiple Single-GPU Pods

For horizontal scaling (e.g., multiple inference replicas):

```yaml
nimOperator:
  nim-llm:
    replicas: 4  # 4 pods, each with 1 GPU
    resources:
      limits:
        nvidia.com/gpu: 1
```

**Total GPUs:** `replicas × nvidia.com/gpu` (4 GPUs in example)

**Scheduling:** Pods can spread across multiple GPU nodes

## Checking GPU Node Taints

Identify GPU taints in your cluster:

```bash
# List all GPU nodes and their taints
oc get nodes -l nvidia.com/gpu.present=true -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Example output:
```
NAME                                       TAINTS
ip-10-0-1-100.ec2.internal                [map[effect:NoSchedule key:nvidia.com/gpu value:present]]
ip-10-0-1-101.ec2.internal                [map[effect:NoSchedule key:nvidia.com/gpu value:present]]
```

**Extracting taint details:**
```bash
oc describe node <gpu-node-name> | grep Taints
```

Example output:
```
Taints:             nvidia.com/gpu=present:NoSchedule
```

**Corresponding toleration:**
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: present
    effect: NoSchedule
```

**Or use wildcard:**
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

## GPU Models and Requirements

Based on video-search-and-summarization blueprint:

| Component | Model | GPU Count | VRAM Required | Reason |
|-----------|-------|-----------|---------------|--------|
| nim-llm | llama-3.1-8b-instruct | 1 | 16 GB | LLM inference |
| nim-llm (70B) | llama-3.1-70b-instruct | 4 | 320 GB (4×80GB) | Large LLM with tensor parallelism |
| nemo-embedding | llama-3.2-nv-embedqa-1b-v2 | 1 | 8 GB | Vector embedding |
| nemo-rerank | llama-3.2-nv-rerankqa-1b-v2 | 1 | 8 GB | Document reranking |
| vss | Cosmos-Reason2-8B (int4_awq) | 1 | 22 GB | Vision-language model (quantized) |
| vss (fp16) | Cosmos-Reason2-8B | 2 | 44 GB (2×22GB) | Vision-language model (full precision) |

**Minimum GPU hardware:**
- **L40S** (46 GB VRAM) ✅ Supports all single-GPU components
- **A100 40GB** ✅ Supports all single-GPU components
- **A10G** (22 GB VRAM) ❌ Insufficient for Cosmos-Reason2-8B

**Total GPU count:**
- **4 GPUs minimum:** 1 LLM + 1 embedding + 1 reranking + 1 VLM
- **8 GPUs for 70B LLM:** 4 LLM + 1 embedding + 1 reranking + 2 VLM

## Known Issues and Gotchas

### Issue: Pod Stuck in Pending Due to GPU Taint

**Problem:** Pod remains `Pending` with event:
```
0/N nodes are available: N node(s) had untolerated taint {nvidia.com/gpu: present}
```

**Cause:** Missing toleration for GPU node taint.

**Solution:** Add toleration matching the node taint:
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

### Issue: Insufficient GPUs Available

**Problem:** Pod stuck in `Pending` with event:
```
0/N nodes are available: N Insufficient nvidia.com/gpu
```

**Causes:**
1. Not enough GPU nodes in cluster
2. GPUs already allocated to other pods
3. Requesting more GPUs than available per node

**Debugging:**
```bash
# Check GPU availability across nodes
oc describe nodes -l nvidia.com/gpu.present=true | grep -A 5 "Allocated resources"

# Check which pods are using GPUs
oc get pods -A -o json | jq '.items[] | select(.spec.containers[].resources.limits."nvidia.com/gpu" != null) | {name: .metadata.name, namespace: .metadata.namespace, gpus: .spec.containers[].resources.limits."nvidia.com/gpu"}'
```

**Solutions:**
- Scale down or delete other GPU workloads
- Add more GPU nodes
- Reduce GPU request count

### Issue: GPU Not Detected in Pod

**Problem:** Pod schedules but can't access GPU (nvidia-smi fails).

**Causes:**
1. NVIDIA GPU Operator not installed or misconfigured
2. GPU device plugin not running
3. Pod missing resource request

**Debugging:**
```bash
# Check GPU operator pods
oc get pods -n nvidia-gpu-operator

# Verify device plugin is running
oc get pods -n nvidia-gpu-operator -l app=nvidia-device-plugin-daemonset

# Test GPU access in pod
oc exec <pod-name> -- nvidia-smi
```

**Solution:** Ensure GPU Operator is installed and healthy

### Issue: MIG (Multi-Instance GPU) Configuration

**Problem:** Cluster uses MIG but pods can't access GPU slices.

**Solution:** Use MIG resource names instead of `nvidia.com/gpu`:
```yaml
resources:
  limits:
    nvidia.com/mig-1g.5gb: 1  # MIG profile name
```

**MIG profile examples:**
- `nvidia.com/mig-1g.5gb`: 1 GPU instance, 5 GB memory
- `nvidia.com/mig-2g.10gb`: 2 GPU instances, 10 GB memory
- `nvidia.com/mig-3g.20gb`: 3 GPU instances, 20 GB memory

### Issue: Cluster-Specific Taints Not Documented

**Problem:** Pods still pending after adding `nvidia.com/gpu` toleration.

**Cause:** Cluster administrators added custom GPU taints.

**Solution:** Check actual taints and add them to values documentation:
```bash
oc describe nodes -l nvidia.com/gpu.present=true | grep Taints
```

Update values file comments:
```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  # Add cluster-specific tolerations as needed:
  # - key: <your-gpu-taint-key>
  #   operator: Equal
  #   value: "true"
  #   effect: NoSchedule
```

## Environment Variables for GPU Configuration

Some workloads need GPU-related environment variables:

### CUDA Visible Devices

Limit which GPUs are visible to the application:

```yaml
env:
  - name: CUDA_VISIBLE_DEVICES
    value: "0"  # Only GPU 0 visible
```

**When to Use:** Multi-GPU nodes where you want to isolate workloads

### NVIDIA Driver Capabilities

Enable specific GPU capabilities:

```yaml
env:
  - name: NVIDIA_DRIVER_CAPABILITIES
    value: "compute,utility"
```

**Common values:**
- `compute`: CUDA compute
- `utility`: nvidia-smi
- `graphics`: OpenGL
- `video`: Video encode/decode
- `all`: All capabilities

## Testing Notes

### Verify GPU Nodes
```bash
oc get nodes -l nvidia.com/gpu.present=true
```

### Check GPU Allocatable Resources
```bash
oc describe node <gpu-node> | grep nvidia.com/gpu
```

Expected output:
```
  nvidia.com/gpu: 1
  nvidia.com/gpu: 1
```

### Check GPU Taints
```bash
oc describe node <gpu-node> | grep Taints
```

### Verify Pod GPU Allocation
```bash
oc describe pod <pod-name> -n <namespace> | grep nvidia.com/gpu
```

Expected output:
```
    Limits:
      nvidia.com/gpu: 1
    Requests:
      nvidia.com/gpu: 1
```

### Test GPU Access in Pod
```bash
oc exec <pod-name> -n <namespace> -- nvidia-smi
```

Expected: GPU listing and utilization

### Check GPU Device Plugin
```bash
oc get pods -n nvidia-gpu-operator -l app=nvidia-device-plugin-daemonset
```

Should show DaemonSet pods running on all GPU nodes

## File Organization

```
deploy/helm/
└── <chart-name>/
    └── values.yaml
        ├── vss:
        │   ├── tolerations: [...]
        │   └── resources:
        │         limits:
        │           nvidia.com/gpu: 1
        └── nimOperator:
              <nim-name>:
                ├── tolerations: [...]
                └── resources:
                      limits:
                        nvidia.com/gpu: 1
```

## Best Practices

1. **Set Both Requests and Limits**: Always set `requests` equal to `limits` for GPUs
2. **Use Generic Tolerations**: Prefer `operator: Exists` over specific values for portability
3. **Document Cluster Taints**: Include placeholder comments for cluster-specific taints
4. **Verify GPU Requirements**: Test actual VRAM usage before documenting minimum requirements
5. **Single GPU Per Pod**: Default to 1 GPU unless tensor parallelism is required
6. **Node Selector as Optional**: Don't force specific GPU types unless necessary
7. **Test on Multiple Node Types**: Verify tolerations work across L40S, A100, H100, etc.
8. **Monitor GPU Utilization**: Use `nvidia-smi` in pods to verify GPU is actually used

## Conversion Checklist

When adding GPU support to a blueprint:

- [ ] Identify which components require GPUs
- [ ] Add `nvidia.com/gpu` resource requests/limits to each GPU component
- [ ] Add `nvidia.com/gpu` toleration to each GPU component
- [ ] Document minimum GPU requirements (type, VRAM, count)
- [ ] Add placeholder comments for cluster-specific taints
- [ ] Test scheduling on GPU nodes
- [ ] Verify GPU access in pods (`nvidia-smi`)
- [ ] Document total GPU count required for deployment
- [ ] Add nodeSelector configuration (optional, commented out)
- [ ] Test with different GPU node types (L40S, A100, H100)
- [ ] Update README with GPU prerequisites
- [ ] Add GPU verification commands to deployment guide

---

## Approach B: Optional GPU Acceleration (from pdf-to-podcast)

### When to Use

When GPU acceleration is **optional** rather than required - the workload can run on CPU but benefits from GPU acceleration. This pattern uses **conditional Helm values** to enable/disable GPU resources.

**Use cases:**
- PDF processing with optional GPU-accelerated OCR
- Image processing with CPU fallback
- ML inference that supports both CPU and GPU
- Development/testing environments without GPUs

### Differences from Approach A

| Aspect | Approach A (Required GPU) | Approach B (Optional GPU) |
|--------|---------------------------|---------------------------|
| **GPU requirement** | Mandatory | Optional (disabled by default) |
| **Values structure** | Direct resource limits | Nested `gpu.enabled` flag |
| **Scheduling** | Always requires GPU node | Conditional GPU scheduling |
| **Fallback** | Pod won't schedule without GPU | Runs on CPU if GPU disabled |
| **Configuration** | Static | User-controlled via Helm values |

### Pattern Implementation

**Values.yaml structure** with GPU as optional feature:

```yaml
celeryWorker:
  image:
    tag: "latest"
  replicas: 1
  gpu:
    enabled: false  # Default to CPU-only
    count: 1        # GPU count when enabled
  resources:
    requests:
      cpu: 1000m
      memory: 8Gi
    limits:
      cpu: 2000m
      memory: 10Gi
```

**Deployment template** with conditional GPU allocation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
  namespace: {{ .Values.global.namespace }}
spec:
  replicas: {{ .Values.celeryWorker.replicas }}
  selector:
    matchLabels:
      app: celery-worker
  template:
    metadata:
      labels:
        app: celery-worker
    spec:
      containers:
      - name: celery-worker
        image: "{{ .Values.imageRegistry }}/celery-worker:{{ .Values.celeryWorker.image.tag }}"
        env:
        - name: CELERY_BROKER_URL
          value: "redis://{{ include "pdf-to-podcast.redisHost" . }}:6379/0"
        resources:
          {{- toYaml .Values.celeryWorker.resources | nindent 10 }}
          {{- if .Values.celeryWorker.gpu.enabled }}
          limits:
            nvidia.com/gpu: {{ .Values.celeryWorker.gpu.count }}
          requests:
            nvidia.com/gpu: {{ .Values.celeryWorker.gpu.count }}
          {{- end }}
      {{- if .Values.celeryWorker.gpu.enabled }}
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      {{- end }}
```

**Key features:**
1. **Conditional GPU resources**: `{{- if .Values.celeryWorker.gpu.enabled }}`
2. **Conditional nodeSelector**: Only added when GPU enabled
3. **Conditional tolerations**: Only added when GPU enabled
4. **Base resources always present**: CPU/memory limits apply regardless

### Enabling GPU at Deployment

**Deploy with GPU enabled:**

```bash
helm install pdf-to-podcast ./helm \
  --set celeryWorker.gpu.enabled=true \
  --set celeryWorker.gpu.count=1
```

**Or create custom values file:**

```yaml
# values-gpu.yaml
celeryWorker:
  gpu:
    enabled: true
    count: 1
  resources:
    requests:
      cpu: 2000m  # Increase CPU for GPU workload
      memory: 16Gi
    limits:
      cpu: 4000m
      memory: 32Gi
```

Deploy:

```bash
helm install pdf-to-podcast ./helm -f values-gpu.yaml
```

### Application-Level GPU Detection

The application should detect GPU availability at runtime:

**Python example:**

```python
import torch
import os

# Check if GPU is requested via Helm values
GPU_ENABLED = os.getenv("GPU_ENABLED", "false").lower() == "true"

# Detect GPU at runtime
if GPU_ENABLED and torch.cuda.is_available():
    device = "cuda"
    print(f"GPU detected: {torch.cuda.get_device_name(0)}")
else:
    device = "cpu"
    print("Running on CPU")

# Use device in ML operations
model = model.to(device)
```

**Environment variable injection** (add to deployment):

```yaml
containers:
- name: celery-worker
  env:
  - name: GPU_ENABLED
    value: {{ .Values.celeryWorker.gpu.enabled | quote }}
```

### Multi-Component Optional GPU

For blueprints with multiple services that support GPU:

```yaml
# values.yaml
celeryWorker:
  gpu:
    enabled: false
    count: 1

pdfProcessor:
  gpu:
    enabled: false
    count: 1

# Enable GPU for specific components
helm install ... \
  --set celeryWorker.gpu.enabled=true \
  --set pdfProcessor.gpu.enabled=false
```

### Advantages of Optional GPU Pattern

1. **Flexibility**: Same chart works for GPU and CPU-only clusters
2. **Cost optimization**: Disable GPU in dev/test to save costs
3. **Progressive rollout**: Start CPU-only, migrate to GPU later
4. **Mixed deployments**: Some replicas on GPU, others on CPU (requires multiple releases)

### Disadvantages

1. **More complex templates**: Conditional logic increases complexity
2. **Testing burden**: Must test both GPU-enabled and GPU-disabled paths
3. **Performance variance**: CPU vs GPU performance may differ significantly
4. **Feature drift**: GPU-only features may not work in CPU mode

### Testing Both Modes

**Test CPU mode:**

```bash
helm install pdf-to-podcast-cpu ./helm \
  --set celeryWorker.gpu.enabled=false

# Verify pod scheduled on non-GPU node
oc get pod -l app=celery-worker -o wide
```

**Test GPU mode:**

```bash
helm install pdf-to-podcast-gpu ./helm \
  --set celeryWorker.gpu.enabled=true

# Verify pod scheduled on GPU node
oc get pod -l app=celery-worker -o wide

# Verify GPU accessible
oc exec deployment/celery-worker -- nvidia-smi
```

## Choosing Between Approaches

**Use Approach A (Required GPU)** when:
- GPU is mandatory for functionality (no CPU fallback)
- All production deployments have GPUs
- Simplicity preferred over flexibility
- Examples: NIM services, GPU-only ML models

**Use Approach B (Optional GPU)** when:
- GPU accelerates but isn't required
- Supporting both GPU and CPU-only environments
- Progressive GPU adoption (start CPU, add GPU later)
- Cost optimization important (disable GPU in dev/test)
- Examples: OCR, image processing, hybrid CPU/GPU workloads

## Related Patterns

- [[pod-affinity-rwo-pvc]] - Pod affinity for GPU workloads sharing storage
- [[security-contexts-scc]] - Security contexts for GPU pods
- [[helm-openshift-conditionals]] - Conditional Helm template patterns
