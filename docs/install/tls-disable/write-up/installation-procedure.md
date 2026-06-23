# Disabling TLS for LLMInferenceService Deployments

In some environments, TLS between the inference gateway and model servers is handled at the platform level. For example, when Istio mutual TLS (mTLS) is enabled across the service mesh, the built-in TLS that LLMInferenceService provides is redundant. In these scenarios, you can disable the built-in TLS to simplify your deployment and avoid double encryption.

When you disable TLS, the llmisvc controller automatically adjusts the following behaviors:

- Health and readiness probes switch from HTTPS to HTTP.
- The Endpoint Picker (EPP) inference scheduler runs with `--secure-serving=false` and stops passing `--cert-path`, `--enable-cert-reload`, and `--model-server-metrics-scheme=https`.
- In disaggregated (prefill/decode) deployments, the routing proxy runs with `--secure-proxy=false` and stops passing `--decoder-use-tls` and `--prefiller-use-tls`.
- Istio DestinationRules for TLS origination are deleted (if they exist).
- The workload service port name changes from `https` to `http`.
- The LLMInferenceService status URL uses the `http://` scheme.
- vLLM SSL arguments (`--enable-ssl-refresh`, `--ssl-certfile`, `--ssl-keyfile`) are not injected into the container command.

Self-signed TLS certificates are still generated regardless of this setting. This is by design and does not affect the behavior of the deployment when TLS is disabled.

## Prerequisites

- You have an OpenShift cluster running version 4.19.9 or later.

- You have installed the OpenShift CLI (`oc`).

- You have logged in as a user with cluster-admin privileges.

- You have installed {productname-long} {vernum}.

- A `DataScienceCluster` (DSC) and `DataScienceClusterInitialization` (DSCI) exist in your cluster with the `llmisvc-controller-manager` and `kserve-controller-manager` enabled.

- You have already deployed an LLMInferenceService resource, or you are preparing to deploy one.

## Configuration Methods

You can disable TLS using either of the following methods. Choose the method that best fits your environment.

### Method 1: ConfigMap (Direct)

This method sets the flag directly in the `inferenceservice-config` ConfigMap. Use this when you manage the ConfigMap manually or need to verify the behavior quickly.

1. Patch the `inferenceservice-config` ConfigMap to set `enableLLMInferenceServiceTLS` to `false` in the `ingress` section:

    ```bash
    oc patch configmap inferenceservice-config \
      -n redhat-ods-applications \
      --type merge \
      -p '{"data":{"ingress":"{\"enableLLMInferenceServiceTLS\": false}"}}'
    ```

2. Restart the llmisvc controller so it picks up the new configuration:

    ```bash
    oc rollout restart deployment llmisvc-controller-manager \
      -n redhat-ods-applications
    ```

3. Wait for the controller to become ready:

    ```bash
    oc rollout status deployment llmisvc-controller-manager \
      -n redhat-ods-applications
    ```

    You should see output similar to:

    ```text
    deployment "llmisvc-controller-manager" successfully rolled out
    ```

### Method 2: Kserve Custom Resource (Recommended for managed environments)

When the kserve-module operator manages your KServe installation, you can set the TLS toggle through the `Kserve` custom resource. The operator propagates this value into the `inferenceservice-config` ConfigMap automatically.

1. Edit the `Kserve` custom resource:

    ```bash
    oc edit kserve default-kserve -n redhat-ods-applications
    ```

2. Set `enableLLMInferenceServiceTLS` to `false` in the spec:

    ```yaml
    apiVersion: kserve.opendatahub.io/v1alpha1
    kind: Kserve
    metadata:
      name: default-kserve
    spec:
      enableLLMInferenceServiceTLS: false
      # ... other fields
    ```

3. Save and exit. The kserve-module operator updates the ConfigMap and restarts the controller automatically.

## Deploying an LLMInferenceService with TLS Disabled

After you disable TLS using one of the methods above, deploy your LLMInferenceService as usual. No changes are needed to the LLMInferenceService CR itself. The controller reads the ConfigMap at reconciliation time and applies the correct behavior.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: sample-llm-inference-service
  namespace: my-namespace
spec:
  replicas: 2
  model:
    uri: hf://RedHatAI/Qwen3-8B-FP8-dynamic
    name: RedHatAI/Qwen3-8B-FP8-dynamic
  router:
    route: {}
    gateway: {}
    scheduler: {}
  template:
    containers:
      - name: main
        resources:
          limits:
            cpu: '4'
            memory: 32Gi
            nvidia.com/gpu: "1"
          requests:
            cpu: '2'
            memory: 16Gi
            nvidia.com/gpu: "1"
```

There is no per-CR override for TLS. The setting applies globally to all LLMInferenceService resources managed by the controller.

## Verification

After the controller reconciles, verify that TLS is disabled by checking the following:

### 1. Check probe schemes

The health and readiness probes on the model server pods should use HTTP, not HTTPS:

```bash
oc get deployment -l app.kubernetes.io/part-of=llminferenceservice \
  -n my-namespace \
  -o jsonpath='{.items[0].spec.template.spec.containers[0].readinessProbe.httpGet.scheme}'
```

Expected output:

```text
HTTP
```

### 2. Check workload service port name

The workload service port should be named `http`, not `https`:

```bash
oc get svc -l app.kubernetes.io/component=workload \
  -n my-namespace \
  -o jsonpath='{.items[0].spec.ports[0].name}'
```

Expected output:

```text
http
```

### 3. Check that DestinationRules are absent (Istio environments only)

If your cluster uses Istio, verify that the controller removed the DestinationRules it previously created:

```bash
oc get destinationrules -n my-namespace \
  -l app.kubernetes.io/part-of=llminferenceservice
```

Expected output:

```text
No resources found in my-namespace namespace.
```

### 4. Check the status URL scheme

The LLMInferenceService status should report an `http://` URL:

```bash
oc get llminferenceservice sample-llm-inference-service \
  -n my-namespace \
  -o jsonpath='{.status.addresses[0].url}'
```

Expected output (the URL should start with `http://`, not `https://`):

```text
http://sample-llm-inference-service-kserve-workload-svc.my-namespace.svc.cluster.local
```

### 5. Check EPP scheduler flags

Verify that the EPP scheduler is running with `--secure-serving=false`:

```bash
oc get deployment -l app.kubernetes.io/component=scheduler \
  -n my-namespace \
  -o jsonpath='{.items[0].spec.template.spec.containers[0].args}' | tr ',' '\n' | grep secure
```

Expected output:

```text
--secure-serving=false
```

You should NOT see `--enable-cert-reload=true`, `--model-server-metrics-scheme=https`, or `--cert-path=/var/run/kserve/tls` in the args.

### 6. Send a test inference request

Verify that the model server responds to an HTTP request:

```bash
oc run curl-test --image=curlimages/curl:7.83.1 \
  -n my-namespace --restart=Never --rm -i -- \
  curl -s http://sample-llm-inference-service-kserve-workload-svc:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"RedHatAI/Qwen3-8B-FP8-dynamic","messages":[{"role":"user","content":"Hello"}],"max_tokens":50}'
```

You should receive a JSON response from the model server.

## Re-enabling TLS

To re-enable TLS, remove the `enableLLMInferenceServiceTLS` key from the ConfigMap (or set it to `true`), and restart the controller. The controller reconciles all LLMInferenceService resources and restores TLS behavior automatically.

For ConfigMap method:

```bash
oc patch configmap inferenceservice-config \
  -n redhat-ods-applications \
  --type merge \
  -p '{"data":{"ingress":"{}"}}'

oc rollout restart deployment llmisvc-controller-manager \
  -n redhat-ods-applications
```

For Kserve CR method: remove the `enableLLMInferenceServiceTLS` field from the Kserve CR, or set it to `true`. The operator handles the rest.

## Technical Reference

### ConfigMap key

The flag lives in the `ingress` section of the `inferenceservice-config` ConfigMap in the controller namespace (`redhat-ods-applications`).

| Key | Type | Default | Effect |
|-----|------|---------|--------|
| `enableLLMInferenceServiceTLS` | boolean | `true` (when key is absent, Go zero-value `false` is not used because the field has `omitempty` and the controller reads from `IngressConfig` which defaults to `true`) | When `false`, disables built-in TLS for all LLMInferenceService deployments |

### Kserve CR field

| Field | Type | Path | Effect |
|-------|------|------|--------|
| `enableLLMInferenceServiceTLS` | `*bool` (pointer to bool) | `spec.enableLLMInferenceServiceTLS` | When set to `false`, the kserve-module operator writes the value into the ConfigMap. When unset (nil), the KServe default (TLS enabled) is preserved. Uses `*bool` so that "not set" and "set to false" are distinguishable. |

### What changes when TLS is disabled

| Component | TLS enabled (default) | TLS disabled |
|-----------|----------------------|--------------|
| Probe scheme | HTTPS | HTTP |
| EPP `--secure-serving` | `--secure-serving=true` | `--secure-serving=false` |
| EPP `--enable-cert-reload` | `--enable-cert-reload=true` | Not rendered (absent) |
| EPP `--model-server-metrics-scheme` | `--model-server-metrics-scheme=https` | Not rendered (absent) |
| EPP `--cert-path` | `--cert-path=/var/run/kserve/tls` | Not rendered (absent) |
| P/D proxy `--secure-proxy` | `--secure-proxy=true` | `--secure-proxy=false` |
| P/D proxy `--decoder-use-tls` | `--decoder-use-tls=true` | Not rendered (absent) |
| P/D proxy `--prefiller-use-tls` | `--prefiller-use-tls=true` | Not rendered (absent) |
| vLLM `--enable-ssl-refresh` | Rendered | Not rendered (absent) |
| vLLM `--ssl-certfile` | `--ssl-certfile /var/run/kserve/tls/tls.crt` | Not rendered (absent) |
| vLLM `--ssl-keyfile` | `--ssl-keyfile /var/run/kserve/tls/tls.key` | Not rendered (absent) |
| Workload service port name | `https` | `http` |
| DestinationRules (Istio) | Created for TLS origination | Deleted |
| Status URL scheme | `https://` | `http://` |
| Self-signed certs | Generated | Still generated (no change) |

### Template files that gate on `.GlobalConfig.EnableTLS`

These are the LLMInferenceServiceConfig template files in `config/llmisvcconfig/` that render differently when TLS is disabled:

| Template file | What it gates |
|--------------|---------------|
| `config-llm-template.yaml` | vLLM SSL args, probe schemes |
| `config-llm-decode-template.yaml` | vLLM SSL args, probe schemes, proxy TLS flags |
| `config-llm-prefill-template.yaml` | vLLM SSL args, probe schemes |
| `config-llm-scheduler.yaml` | EPP `--secure-serving`, `--enable-cert-reload`, `--model-server-metrics-scheme`, `--cert-path` |
| `config-llm-decode-worker-data-parallel.yaml` | vLLM SSL args, probe schemes, proxy TLS flags |
| `config-llm-prefill-worker-data-parallel.yaml` | vLLM SSL args, probe schemes |
| `config-llm-worker-data-parallel.yaml` | vLLM SSL args, probe schemes |

### Controller code paths affected

| File | What it does |
|------|-------------|
| `pkg/apis/serving/v1beta1/configmap.go:128` | Defines `EnableLLMInferenceServiceTLS` field on `IngressConfig` struct |
| `pkg/controller/v1alpha2/llmisvc/config_loader.go:172` | Maps `IngressConfig.EnableLLMInferenceServiceTLS` to `Config.EnableTLS` |
| `pkg/controller/v1alpha2/llmisvc/workload.go:114` | Gates service port name (`https` vs `http`) |
| `pkg/controller/v1alpha2/llmisvc/router.go:323` | Gates status URL scheme (`HTTPS` vs `HTTP`) |
| `pkg/controller/v1alpha2/llmisvc/router_platform_networking_odh.go:122` | Decides whether to create or delete DestinationRules |
| `kserve-module/pkg/apis/v1alpha1/types.go:68` | Defines `*bool` field on Kserve CR spec |
| `kserve-module/pkg/kservemodule/configmap.go:71-72` | Propagates CR field value into ConfigMap |

### Source PRs

| PR | Repository | Description |
|----|------------|-------------|
| [#5525](https://github.com/kserve/kserve/pull/5525) | kserve/kserve | Upstream: wire enableLLMInferenceServiceTLS through reconcile pipeline |
| [#1595](https://github.com/opendatahub-io/kserve/pull/1595) | opendatahub-io/kserve | Midstream: gate TLS resources on enableLLMInferenceServiceTLS |
| [#1622](https://github.com/opendatahub-io/kserve/pull/1622) | opendatahub-io/kserve | kserve-module: add EnableLLMInferenceServiceTLS toggle to Kserve CR |

### Jira tickets

| Ticket | Description |
|--------|-------------|
| [RHAISTRAT-1726](https://redhat.atlassian.net/browse/RHAISTRAT-1726) | Feature: Provide a way to disable TLS within the LLMInferenceService deployment |
| [RHOAIENG-67441](https://redhat.atlassian.net/browse/RHOAIENG-67441) | Documentation tracking ticket |
| [RHOAIENG-62947](https://redhat.atlassian.net/browse/RHOAIENG-62947) | Downstream delivery epic |
