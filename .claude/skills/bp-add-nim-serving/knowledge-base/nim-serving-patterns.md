# NIM Serving Deployment Patterns

Verified patterns from a live RHOAI cluster with NIM integration enabled.

## ServingRuntime Pattern

Each NIM model gets its own ServingRuntime, instantiated per model with the specific image and model format.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  annotations:
    opendatahub.io/apiProtocol: REST
    opendatahub.io/recommended-accelerators: '["nvidia.com/gpu"]'
    opendatahub.io/template-display-name: NVIDIA NIM
    opendatahub.io/template-name: nvidia-nim-runtime
    openshift.io/display-name: {{ $nim.displayName }}
    runtimes.opendatahub.io/nvidia-nim: "true"
  labels:
    opendatahub.io/dashboard: "true"
  name: {{ $nim.service.name }}
spec:
  annotations:
    prometheus.io/path: /metrics
    prometheus.io/port: "{{ $nim.service.port }}"
  containers:
    - name: kserve-container
      image: "{{ $nim.image.repository }}:{{ $nim.image.tag }}"
      ports:
        - containerPort: {{ $nim.service.port }}
          protocol: TCP
      env:
        - name: NIM_CACHE_PATH
          value: /mnt/models/cache
        - name: NGC_API_KEY
          valueFrom:
            secretKeyRef:
              key: NGC_API_KEY
              name: {{ .Values.nimServing.secrets.ngcApiSecret }}
      volumeMounts:
        - mountPath: /dev/shm
          name: shm
        - mountPath: /mnt/models/cache
          name: nim-pvc
        - mountPath: /opt/nim/workspace
          name: nim-workspace
        - mountPath: /.cache
          name: nim-cache
        - mountPath: /opt/nim/nginx
          name: nim-nginx
        - mountPath: /opt/nim/generated_configs
          name: nim-generated-configs
        - mountPath: /opt/nim/.cache
          name: nim-dot-cache
        - mountPath: /opt/nim/.config
          name: nim-dot-config
        - mountPath: /opt/nim/.triton
          name: nim-dot-triton
  imagePullSecrets:
    - name: {{ .Values.nimServing.secrets.ngcPullSecret }}
  multiModel: false
  protocolVersions:
    - grpc-v2
    - v2
  supportedModelFormats:
    - name: {{ $nim.modelFormat }}
      autoSelect: true
      priority: 1
      version: "{{ $nim.image.tag }}"
  volumes:
    - name: nim-pvc
      persistentVolumeClaim:
        claimName: {{ $nim.service.name }}-cache
    - name: nim-workspace
      emptyDir: {}
    - name: nim-cache
      emptyDir: {}
    - name: nim-nginx
      emptyDir: {}
    - name: nim-generated-configs
      emptyDir: {}
    - name: nim-dot-cache
      emptyDir: {}
    - name: nim-dot-config
      emptyDir: {}
    - name: nim-dot-triton
      emptyDir: {}
    - name: shm
      emptyDir:
        medium: Memory
        sizeLimit: 2Gi
```

### Critical Notes

- **9 volume mounts** are required. Missing mounts cause container crashes.
- **`runtimes.opendatahub.io/nvidia-nim: "true"`** — required for NIM-specific metrics dashboard.
- **`opendatahub.io/dashboard: "true"`** label — required for dashboard visibility.
- **`imagePullSecrets`** — references pre-existing `ngc-secret`, NOT created by the chart.
- **`multiModel: false`** — each NIM serves exactly one model.

## InferenceService Pattern

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  annotations:
    openshift.io/display-name: {{ $nim.displayName }}
    serving.kserve.io/deploymentMode: RawDeployment
  labels:
    {{- if $nim.externalRoute }}
    networking.kserve.io/visibility: exposed
    {{- end }}
    opendatahub.io/dashboard: "true"
  name: {{ $nim.service.name }}
spec:
  predictor:
    automountServiceAccountToken: false
    {{- with $nim.securityContext }}
    securityContext:
{{ toYaml . | nindent 6 }}
    {{- end }}
    minReplicas: {{ $nim.replicas | default 1 }}
    maxReplicas: {{ $nim.maxReplicas | default 1 }}
    model:
      modelFormat:
        name: {{ $nim.modelFormat }}
      runtime: {{ $nim.service.name }}
      resources:
{{ toYaml $nim.resources | nindent 8 }}
      env:
        - name: HOME
          value: /.cache
        - name: TRITON_CACHE_DIR
          value: /.cache/triton
    {{- with $nim.tolerations }}
    tolerations:
{{ toYaml . | nindent 6 }}
    {{- end }}
```

### Critical Notes

- **`deploymentMode: RawDeployment`** — RHOAI NIM uses raw Kubernetes Deployments, NOT Knative serverless.
- **`securityContext`** — NIM containers run as UID 1000. On OpenShift with a custom SCC (`fsGroup: RunAsAny`), the SCC permits any fsGroup but does NOT auto-assign one. Without an explicit `fsGroup` in the pod spec, the PVC is owned by root and model download fails with "Permission denied". The `securityContext` block is configurable via values — set `fsGroup` in the OpenShift overlay (`values-openshift.yaml`), not the base values, since vanilla Kubernetes doesn't need it.
- **`HOME=/.cache`** and **`TRITON_CACHE_DIR=/.cache/triton`** — required for writable directories.
- **GPU requests MUST equal limits** — required for Guaranteed QoS class.
- **Tolerations go on InferenceService**, not ServingRuntime.
- **`modelFormat.name`** must match `supportedModelFormats[0].name` in ServingRuntime.
- **Visibility label** — controlled by `externalRoute` boolean in values. When `true`, sets `networking.kserve.io/visibility: exposed` to create an external Route. When `false`, the label is **omitted entirely** (not set to "cluster-local"). This matches odh-dashboard behavior: the controller only checks for "exposed"; absence = cluster-local.

## PVC Pattern

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ $nim.service.name }}-cache
  labels:
    opendatahub.io/managed: "true"
spec:
  accessModes:
    - ReadWriteOnce
  {{- if $nim.storage.storageClass }}
  storageClassName: {{ $nim.storage.storageClass }}
  {{- end }}
  resources:
    requests:
      storage: {{ $nim.storage.size | default "100Gi" }}
```

## InferenceService Endpoint URL Pattern

Once ready, the InferenceService exposes:
- **Internal**: `http://<isvc-name>-predictor.<namespace>.svc.cluster.local:<container-port>`
- **External** (if `externalRoute: true`): `https://<isvc-name>-<namespace>.apps.<cluster-domain>`

**Port is required for internal URLs.** KServe creates a headless service (`ClusterIP: None`) for the predictor. Headless services resolve directly to the pod IP, so the service-level port remapping (80→8000) does not apply. Clients must connect on the container port (default 8000) explicitly. Make the port configurable via values (`service.port`).

Use the internal URL for service-to-service communication within the blueprint.
