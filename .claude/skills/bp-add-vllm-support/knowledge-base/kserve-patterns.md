---
name: kserve-patterns
description: KServe ServingRuntime + InferenceService + companion Service templates for vLLM on RHOAI
summary: |
  Complete Helm template patterns for deploying vLLM models on RHOAI via KServe.
  Covers the three-resource pattern (ServingRuntime + InferenceService + companion Service),
  vLLM-native model download via --model + --download-dir, tensor-parallel-size derivation
  from GPU limits, HF token secret placement, and the full values.yaml schema. Companion
  Services bridge the port gap (8000→8080) so applications don't need code changes.
metadata:
  type: resource-pattern
resource_types: [model-serving, kserve, gpu]
deployment_types: [helm]
---

# KServe Patterns for vLLM on RHOAI

## Architecture

A **companion Service** is a ClusterIP Service that bridges port 8000 (NIM-compatible) → 8080 (vLLM internal), so applications don't need code changes.

```
Application → companion Service (port 8000) → vLLM pod (port 8080)
                                               ↑ managed by KServe InferenceService
                                               ↑ image defined by ServingRuntime
                                               ↑ vLLM downloads model from HuggingFace on startup
```

## Key Patterns

- **--model**: Pass the HuggingFace model ID directly (e.g., `--model nvidia/Llama-3_3-Nemotron-Super-49B-v1_5`). vLLM downloads the model itself using its built-in HuggingFace client. No `storageUri` or KServe storage-initializer needed.
- **--download-dir /vllm/model**: Download location backed by a `vllm-home` emptyDir volume mounted at `/vllm`. Do NOT use `/mnt/models` — KServe auto-injects a volume at that path, causing a duplicate mount error. Model is re-downloaded on pod restart.
- **HF_TOKEN on ServingRuntime**: For gated models, the HF token env var on the ServingRuntime goes directly to the vLLM container that does the download — no init container propagation issues.
- **--tensor-parallel-size**: Derive from `nvidia.com/gpu` resource limit. vLLM requires this flag explicitly — it does not auto-detect GPU count.
- **HOME=/vllm**: Set `HOME=/vllm` in the ServingRuntime env, backed by the `vllm-home` emptyDir. Required for OpenShift where the default home directory is not writable.

## 1. ServingRuntime (Shared — One Per Blueprint)

```yaml
{{- if $anyVllmModelEnabled }}
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: {{ .Values.vllm.servingRuntime.name }}
  annotations:
    openshift.io/display-name: vLLM NVIDIA GPU ServingRuntime for KServe
    opendatahub.io/recommended-accelerators: '["nvidia.com/gpu"]'
    opendatahub.io/runtime-version: {{ .Values.vllm.servingRuntime.image.tag | quote }}
  labels:
    opendatahub.io/dashboard: "true"
spec:
  annotations:
    opendatahub.io/kserve-runtime: vllm
    opendatahub.io/apiProtocol: REST
    opendatahub.io/template-display-name: vLLM ServingRuntime for KServe (GPU)
    opendatahub.io/template-name: vllm-runtime-gpu
    serving.knative.dev/progress-deadline: 60m
    prometheus.io/path: /metrics
    prometheus.io/port: "8080"
  containers:
  - command:
    - python3
    - -m
    - vllm.entrypoints.openai.api_server
    args:
    - --port=8080
    env:
    - name: HOME
      value: /vllm
    - name: HF_HOME
      value: /vllm/hf_home
    - name: VLLM_SKIP_WARMUP
      value: "true"
    - name: HF_HUB_OFFLINE
      value: "0"
    - name: HF_TOKEN
      valueFrom:
        secretKeyRef:
          key: HF_TOKEN
          name: {{ .Values.vllm.servingRuntime.secrets.huggingFaceToken }}
    image: {{ .Values.vllm.servingRuntime.image.repository }}:{{ .Values.vllm.servingRuntime.image.tag }}
    name: kserve-container
    ports:
    - containerPort: 8080
      protocol: TCP
    volumeMounts:
    - name: shm
      mountPath: /dev/shm
    - name: vllm-home
      mountPath: /vllm
  multiModel: false
  supportedModelFormats:
  - autoSelect: true
    name: vLLM
  volumes:
  - name: shm
    emptyDir:
      medium: Memory
      sizeLimit: 2Gi
  - name: vllm-home
    emptyDir: {}
{{- end }}
```

## 2. InferenceService (One Per Model)

```yaml
{{- $model := .Values.vllm.models.<modelKey> }}
{{- $isvcName := printf "vllm-%s" "<model-name>" }}
{{- if $model.enabled }}
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: {{ $isvcName }}
  annotations:
    serving.kserve.io/deploymentMode: RawDeployment
    openshift.io/display-name: {{ $isvcName }}
    serving.knative.openshift.io/enablePassthrough: "true"
  labels:
    opendatahub.io/dashboard: "true"
    networking.knative.dev/visibility: cluster-local
spec:
  predictor:
    {{- with $model.securityContext }}
    securityContext:
{{ toYaml . | nindent 6 }}
    {{- end }}
    minReplicas: {{ $model.replicas | default 1 }}
    maxReplicas: {{ $model.replicas | default 1 }}
    model:
      modelFormat:
        name: vLLM
      name: ""
      runtime: {{ .Values.vllm.servingRuntime.name }}
      resources:
{{ toYaml $model.resources | nindent 8 }}
      args:
      - --model
      - {{ $model.modelId }}
      - --download-dir
      - /vllm/model
      - --served-model-name
      - {{ $model.servedName }}
      - --tensor-parallel-size
      - {{ index $model.resources.limits "nvidia.com/gpu" | quote }}
      {{- if $model.trustRemoteCode }}
      - --trust-remote-code
      {{- end }}
      {{- range $model.extraArgs }}
      - {{ . | quote }}
      {{- end }}
    {{- with $model.tolerations }}
    tolerations:
{{ toYaml . | nindent 4 }}
    {{- end }}
{{- end }}
```

## 3. Companion Service (One Per Model)

```yaml
{{- $model := .Values.vllm.models.<modelKey> }}
{{- if $model.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ $model.service.name }}
  labels:
    app: {{ $model.service.name }}
spec:
  type: ClusterIP
  ports:
  - port: 8000
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: isvc.{{ $isvcName }}-predictor
{{- end }}
```

## 4. HuggingFace Token Secret

The ServingRuntime references an HF token secret via `secretKeyRef` for the vLLM container. vLLM downloads the model directly using its built-in HuggingFace client, so the HF token reaches the container that does the download — no ClusterStorageContainer CR or storage-initializer workaround needed.

- **Public models** — no token needed, vLLM downloads without authentication.
- **Gated models** — the `HF_TOKEN` env var on the ServingRuntime is sufficient. vLLM uses it to authenticate with HuggingFace Hub.

**Always generate the secret block**, guarded by the model's enabled flag. The user controls the secret name via `vllm.servingRuntime.secrets.huggingFaceToken` in values — if they have a pre-existing secret on the cluster, they set that value to the existing secret name.

Do NOT use Helm `lookup` to detect existing secrets at deploy time — it doesn't work with `helm template` and adds unnecessary complexity.

Place the secret in the existing secrets template if one exists, otherwise in `vllm-serving-runtime.yaml`.

```yaml
{{- if .Values.vllm.models.<modelKey>.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.vllm.servingRuntime.secrets.huggingFaceToken }}
type: Opaque
stringData:
  HF_TOKEN: {{ .Values.vllm.huggingFaceToken | default "" | quote }}
{{- end }}
```

## Values Schema

```yaml
vllm:
  huggingFaceToken: ""    # only when generating fallback secret; omit if blueprint already has one
  servingRuntime:
    name: vllm-serving-runtime
    image:
      repository: <discovered-image>
      tag: "<discovered-tag>"
    secrets:
      huggingFaceToken: "hf-token"    # secret name — set to existing secret name if blueprint has one

  models:
    <model-key>:
      enabled: false
      replicas: 1
      service:
        name: "<model-key>-vllm"
      modelId: "<hf-org/model-name>"
      servedName: "<api-model-name>"
      trustRemoteCode: false
      extraArgs: []           # version-specific and user args (e.g., --task embed, --dtype fp8)
      resources:
        limits:
          nvidia.com/gpu: 1
          memory: "64Gi"
          cpu: "16"
        requests:
          nvidia.com/gpu: 1
          memory: "32Gi"
          cpu: "8"
      tolerations: []
      securityContext: {}
```
