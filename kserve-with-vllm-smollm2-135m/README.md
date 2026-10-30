# KServe with vLLM and SmolLM2-135M-Instruct

Create a sample LLM infrastructure on Kubernetes with:

- KServe (LLMInferenceService)
- vLLM with SmolLM2-135M-Instruct
- Gateway API Inference Extension
- agentgateway as Gateway
- EndpointPicker and InferencePool
- curl and kagent as clients

## Overview

```console
+--[Kubernetes]--------------------------------------+
|                                                    |
|     +---------------+        +---------------+     |
|     |     curl      |        |     kagent    |     |
|     +------+--------+        +--------+------+     |
|            |                          |            |
|            +------------+-------------+            |
|                         |                          |
|                  (Run OpenAI API)                  |
|                         |                          |
|            +------------+                          |
|            |                                       |
|  +---------+------------------------------------+  |
|  |         |    [KServe (LLMInferenceService)]  |  |
|  |         |                                    |  |
|  |  +------+---------------------------------+  |  |
|  |  |      |                 [llm-d Router]  |  |  |
|  |  |      V                                 |  |  |
|  |  |  +---+----------+    +--------------+  |  |  |
|  |  |  | agentgateway |--->|  llm-d EPP   |  |  |  |
|  |  |  +---+----------+    +--------------+  |  |  |
|  |  |      |                                 |  |  |
|  |  +------+---------------------------------+  |  |
|  |         |                                    |  |
|  |         +------------+                       |  |
|  |                      |                       |  |
|  |  +-------------------+--------------------+  |  |
|  |  |                   |  [Inference Pool]  |  |  |
|  |  |                   V                    |  |  |
|  |  |            +--------------+            |  |  |
|  |  |            | vLLM         |            |  |  |
|  |  |            | SmolLM2-135M |            |  |  |
|  |  |            +--------------+            |  |  |
|  |  |                                        |  |  |
|  |  +----------------------------------------+  |  |
|  |                                              |  |
|  +----------------------------------------------+  |
|                                                    |
+----------------------------------------------------+
```

## Deploy each component on Kubernetes

### Create a Kubernetes namespace

1. Create a Kubernetes namespace.

    ```console
    kubectl create ns llm-d
    ```

### Deploy cert-manager

1. Deploy cert-manager as a dependency of KServe.

    ```console
    helm install cert-manager \
      oci://quay.io/jetstack/charts/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --set crds.enabled=true \
      --set crds.keep=false \
      --version v1.21.3
    ```

### Deploy CRDs for Gateway API and Gateway API Inference Extension

1. Deploy CRDs for Gateway API and Gateway API Inference Extension.

    ```console
    kubectl apply \
      -f ./gateway-api/gateway-api.yaml \
      -f ./gateway-api/gateway-api-inference-extension.yaml
    ```

### Deploy agentgateway as implementation of Gateway (GatewayClass)

1. Deploy CRDs for agentgateway.

    ```console
    helm install agentgateway-crds \
      oci://cr.agentgateway.dev/charts/agentgateway-crds \
      --namespace agentgateway-system \
      --create-namespace \
      --version v1.6.0
    ```

1. Deploy a default configuration for agentgateway.

    ```console
    kubectl apply \
      -f ./agentgateway/agentgateway-parameters.yaml \
      -n agentgateway-system
    ```

1. Deploy agentgateway.

    ```console
    helm install agentgateway \
      oci://cr.agentgateway.dev/charts/agentgateway \
      --namespace agentgateway-system \
      --values ./agentgateway/agentgateway.yaml \
      --version v1.6.0
    ```

### Deploy Gateway

1. Deploy Gateway.

    ```console
    kubectl apply \
      -f ./gateway-api/gateway.yaml \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]--------------------------------------+
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |        +--------------+                            |
    |        | agentgateway |                            |
    |        +--------------+                            |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    +----------------------------------------------------+
    ```

### Deploy KServe

1. Deploy CRDs for KServe.

    ```console
    helm install kserve-llmisvc-crd \
      oci://ghcr.io/kserve/charts/kserve-llmisvc-crd \
      --namespace kserve \
      --create-namespace \
      --version v0.21.0
    ```

1. Deploy KServe resources.

    ```console
    helm install kserve-llmisvc-resources \
      oci://ghcr.io/kserve/charts/kserve-llmisvc-resources \
      --namespace kserve \
      --set kserve.llmisvc.createGIECRDs=false \
      --wait \
      --version v0.21.0
    ```

1. Deploy KServe configs.

    ```console
    helm install kserve-runtime-configs \
      oci://ghcr.io/kserve/charts/kserve-runtime-configs \
      --namespace kserve \
      --set kserve.llmisvcConfigs.enabled=true \
      --version v0.21.0
    ```

    ```console
    +--[Kubernetes]--------------------------------------+
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |  +----------------------------------------------+  |
    |  |                                    [KServe]  |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |     +--------------+                         |  |
    |  |     | agentgateway |                         |  |
    |  |     +--------------+                         |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  +----------------------------------------------+  |
    |                                                    |
    +----------------------------------------------------+
    ```

### Deploy an LLMInferenceService resource

1. Deploy an LLMInferenceService resource.

    ```console
    kubectl apply \
      -f ./kserve/llmisvc-smollm2-135m.yaml \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]--------------------------------------+
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |  +----------------------------------------------+  |
    |  |              [KServe (LLMInferenceService)]  |  |
    |  |                                              |  |
    |  |  +----------------------------------------+  |  |
    |  |  |                        [llm-d Router]  |  |  |
    |  |  |                                        |  |  |
    |  |  |  +--------------+    +--------------+  |  |  |
    |  |  |  | agentgateway |    |  llm-d EPP   |  |  |  |
    |  |  |  +--------------+    +--------------+  |  |  |
    |  |  |                                        |  |  |
    |  |  +----------------------------------------+  |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |  +----------------------------------------+  |  |
    |  |  |                      [Inference Pool]  |  |  |
    |  |  |                                        |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |            | vLLM         |            |  |  |
    |  |  |            | SmolLM2-135M |            |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |                                        |  |  |
    |  |  +----------------------------------------+  |  |
    |  |                                              |  |
    |  +----------------------------------------------+  |
    |                                                    |
    +----------------------------------------------------+
    ```

### Deploy a client pod

1. Deploy a client pod.

    ```console
    kubectl apply \
      -f ./client/client.yaml \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]--------------------------------------+
    |                                                    |
    |     +---------------+                              |
    |     |     curl      |                              |
    |     +---------------+                              |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |                                                    |
    |  +----------------------------------------------+  |
    |  |              [KServe (LLMInferenceService)]  |  |
    |  |                                              |  |
    |  |  +----------------------------------------+  |  |
    |  |  |                        [llm-d Router]  |  |  |
    |  |  |                                        |  |  |
    |  |  |  +--------------+    +--------------+  |  |  |
    |  |  |  | agentgateway |    |  llm-d EPP   |  |  |  |
    |  |  |  +--------------+    +--------------+  |  |  |
    |  |  |                                        |  |  |
    |  |  +----------------------------------------+  |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |                                              |  |
    |  |  +----------------------------------------+  |  |
    |  |  |                      [Inference Pool]  |  |  |
    |  |  |                                        |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |            | vLLM         |            |  |  |
    |  |  |            | SmolLM2-135M |            |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |                                        |  |  |
    |  |  +----------------------------------------+  |  |
    |  |                                              |  |
    |  +----------------------------------------------+  |
    |                                                    |
    +----------------------------------------------------+
    ```

### Run sample requests

1. Run a shell in the client pod.

    ```console
    kubectl exec -it client -n llm-d -- bash
    ```

1. Check models.

    ```console
    curl -s http://inference-gateway.llm-d.svc.cluster.local/llm-d/smollm2-135m/v1/models | jq .
    ```

1. Say "Hello!".

    ```console
    curl -s http://inference-gateway.llm-d.svc.cluster.local/llm-d/smollm2-135m/v1/chat/completions \
      -H 'Content-Type: application/json' \
      -d '{
        "model": "HuggingFaceTB/SmolLM2-135M-Instruct",
        "messages": [{"role": "user", "content": "Hello!"}],
        "max_tokens": 50
      }' | jq
    ```
    Note: It may take a bit of time to return a response since this sample uses vLLM with CPU for inference.

    ```console
    +--[Kubernetes]--------------------------------------+
    |                                                    |
    |     +---------------+                              |
    |     |     curl      |                              |
    |     +------+--------+                              |
    |            |                                       |
    |            +------------+                          |
    |                         |                          |
    |                  (Run OpenAI API)                  |
    |                         |                          |
    |            +------------+                          |
    |            |                                       |
    |  +---------+------------------------------------+  |
    |  |         |    [KServe (LLMInferenceService)]  |  |
    |  |         |                                    |  |
    |  |  +------+---------------------------------+  |  |
    |  |  |      |                 [llm-d Router]  |  |  |
    |  |  |      V                                 |  |  |
    |  |  |  +---+----------+    +--------------+  |  |  |
    |  |  |  | agentgateway |--->|  llm-d EPP   |  |  |  |
    |  |  |  +---+----------+    +--------------+  |  |  |
    |  |  |      |                                 |  |  |
    |  |  +------+---------------------------------+  |  |
    |  |         |                                    |  |
    |  |         +------------+                       |  |
    |  |                      |                       |  |
    |  |  +-------------------+--------------------+  |  |
    |  |  |                   |  [Inference Pool]  |  |  |
    |  |  |                   V                    |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |            | vLLM         |            |  |  |
    |  |  |            | SmolLM2-135M |            |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |                                        |  |  |
    |  |  +----------------------------------------+  |  |
    |  |                                              |  |
    |  +----------------------------------------------+  |
    |                                                    |
    +----------------------------------------------------+
    ```

### Deploy kagent

1. Create a Kubernetes namespace.

    ```console
    kubectl create ns kagent
    ```

1. Create a dummy secret.

    ```console
    kubectl create secret generic kagent-openai \
      --from-literal=OPENAI_API_KEY='dummy-api-key' \
      -n kagent
    ```

1. Deploy CRDs for kagent.

    ```console
    helm install kagent-crds \
      oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
      --namespace kagent \
      --version 0.10.3
    ```

1. Deploy kagent.

    ```console
    helm install kagent \
      oci://ghcr.io/kagent-dev/kagent/helm/kagent \
      --namespace kagent \
      --values ./kagent/kagent.yaml \
      --version 0.10.3
    ```

1. Deploy a sample agent.

    ```console
    kubectl apply \
      -f ./kagent/sample-agent.yaml \
      -n kagent
    ```

    ```console
    +--[Kubernetes]--------------------------------------+
    |                                                    |
    |     +---------------+        +---------------+     |
    |     |     curl      |        |     kagent    |     |
    |     +------+--------+        +--------+------+     |
    |            |                          |            |
    |            +------------+-------------+            |
    |                         |                          |
    |                  (Run OpenAI API)                  |
    |                         |                          |
    |            +------------+                          |
    |            |                                       |
    |  +---------+------------------------------------+  |
    |  |         |    [KServe (LLMInferenceService)]  |  |
    |  |         |                                    |  |
    |  |  +------+---------------------------------+  |  |
    |  |  |      |                 [llm-d Router]  |  |  |
    |  |  |      V                                 |  |  |
    |  |  |  +---+----------+    +--------------+  |  |  |
    |  |  |  | agentgateway |--->|  llm-d EPP   |  |  |  |
    |  |  |  +---+----------+    +--------------+  |  |  |
    |  |  |      |                                 |  |  |
    |  |  +------+---------------------------------+  |  |
    |  |         |                                    |  |
    |  |         +------------+                       |  |
    |  |                      |                       |  |
    |  |  +-------------------+--------------------+  |  |
    |  |  |                   |  [Inference Pool]  |  |  |
    |  |  |                   V                    |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |            | vLLM         |            |  |  |
    |  |  |            | SmolLM2-135M |            |  |  |
    |  |  |            +--------------+            |  |  |
    |  |  |                                        |  |  |
    |  |  +----------------------------------------+  |  |
    |  |                                              |  |
    |  +----------------------------------------------+  |
    |                                                    |
    +----------------------------------------------------+
    ```

### Access kagent UI

1. Run the `kubectl port-forward` command.

    ```console
    kubectl port-forward \
      svc/kagent-ui 8080:8080 \
      -n kagent
    ```

1. Access http://localhost:8080/.

## Delete the sample deployment

```console
kubectl delete \
  -f ./kagent/sample-agent.yaml \
  -n kagent
```
```console
helm uninstall \
  kagent \
  kagent-crds \
  -n kagent
```
```console
kubectl delete secrets \
  kagent-openai \
  -n kagent
```
```console
kubectl delete \
  -f ./client/client.yaml \
  -n llm-d
```
```console
kubectl delete \
  -f ./kserve/llmisvc-smollm2-135m.yaml \
  -n llm-d
```
```console
kubectl delete \
  validatingwebhookconfiguration \
  llminferenceservice.serving.kserve.io \
  llminferenceserviceconfig.serving.kserve.io
```
```console
helm uninstall \
  kserve-runtime-configs \
  -n kserve
```
```console
helm uninstall \
  kserve-llmisvc-resources \
  -n kserve
```
```console
helm uninstall \
  kserve-llmisvc-crd \
  -n kserve
```
```console
kubectl delete \
  -f ./gateway-api/gateway.yaml \
  -n llm-d
```
```console
kubectl delete \
  -f ./agentgateway/agentgateway-parameters.yaml \
  -n agentgateway-system
```
```console
helm uninstall \
  agentgateway \
  agentgateway-crds \
  -n agentgateway-system
```
```console
kubectl delete \
  -f ./gateway-api/gateway-api.yaml \
  -f ./gateway-api/gateway-api-inference-extension.yaml
```
```console
helm uninstall \
  cert-manager \
  -n cert-manager
```
```console
kubectl delete ns \
  kagent \
  kserve \
  agentgateway-system \
  cert-manager \
  llm-d
```
