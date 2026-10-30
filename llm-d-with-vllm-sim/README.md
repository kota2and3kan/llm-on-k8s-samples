# llm-d with vLLM Simulator

Create a sample LLM infrastructure on Kubernetes with:

- vLLM Simulator
- Gateway API Inference Extension
- agentgateway as Gateway
- EndpointPicker and InferencePool

## Overview

```console
+--[Kubernetes]-------------------------------------------+
|                                                         |
|     +---------------+                                   |
|     | Client (curl) |                                   |
|     +-------+-------+                                   |
|             |                                           |
|     (Run OpenAI API)                                    |
|             |                                           |
|  +----------+----------------------------------------+  |
|  |          |                        [llm-d Router]  |  |
|  |          V                                        |  |
|  |  +-------+------+               +--------------+  |  |
|  |  | agentgateway |-------------->|  llm-d EPP   |  |  |
|  |  +-------+------+               +--------------+  |  |
|  |          |                                        |  |
|  +----------+----------------------------------------+  |
|             |                                           |
|             +--------------+--------------+             |
|             |              |              |             |
|  +----------+--------------+--------------+----------+  |
|  |          |              |              |          |  |
|  |          V              V              V          |  |
|  |    +-----------+  +-----------+  +-----------+    |  |
|  |    | vLLM      |  | vLLM      |  | vLLM      |    |  |
|  |    | Simulator |  | Simulator |  | Simulator |    |  |
|  |    +-----------+  +-----------+  +-----------+    |  |
|  |                                                   |  |
|  |                                 [Inference Pool]  |  |
|  +---------------------------------------------------+  |
|                                                         |
+---------------------------------------------------------+
```

## Deploy each component on Kubernetes

### Create a Kubernetes namespace

1. Create a Kubernetes namespace.

    ```console
    kubectl create ns llm-d
    ```

### Deploy vLLM Simulator

1. Deploy vLLM Simulator.

    ```console
    kubectl apply \
      -f ./vllm/vllm-sim.yaml \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]-------------------------------------------+
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |       +-----------+  +-----------+  +-----------+       |
    |       | vLLM      |  | vLLM      |  | vLLM      |       |
    |       | Simulator |  | Simulator |  | Simulator |       |
    |       +-----------+  +-----------+  +-----------+       |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    +---------------------------------------------------------+
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
    +--[Kubernetes]-------------------------------------------+
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |     +--------------+                                    |
    |     | agentgateway |                                    |
    |     +--------------+                                    |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |       +-----------+  +-----------+  +-----------+       |
    |       | vLLM      |  | vLLM      |  | vLLM      |       |
    |       | Simulator |  | Simulator |  | Simulator |       |
    |       +-----------+  +-----------+  +-----------+       |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    +---------------------------------------------------------+
    ```

### Deploy EPP (Endpoint Picker) and InferencePool

1. Deploy EPP (Endpoint Picker) and InferencePool.

    ```console
    helm install llm-d-router-gateway \
      oci://ghcr.io/llm-d/charts/llm-d-router-gateway \
      --namespace llm-d \
      --values ./llm-d-router/llm-d-router-gateway.yaml \
      --version v1.0.0
    ```

    ```console
    +--[Kubernetes]-------------------------------------------+
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    |  +---------------------------------------------------+  |
    |  |                                   [llm-d Router]  |  |
    |  |                                                   |  |
    |  |  +--------------+               +--------------+  |  |
    |  |  | agentgateway |               |  llm-d EPP   |  |  |
    |  |  +--------------+               +--------------+  |  |
    |  |                                                   |  |
    |  +---------------------------------------------------+  |
    |                                                         |
    |                                                         |
    |                                                         |
    |  +---------------------------------------------------+  |
    |  |                                                   |  |
    |  |                                                   |  |
    |  |    +-----------+  +-----------+  +-----------+    |  |
    |  |    | vLLM      |  | vLLM      |  | vLLM      |    |  |
    |  |    | Simulator |  | Simulator |  | Simulator |    |  |
    |  |    +-----------+  +-----------+  +-----------+    |  |
    |  |                                                   |  |
    |  |                                 [Inference Pool]  |  |
    |  +---------------------------------------------------+  |
    |                                                         |
    +---------------------------------------------------------+
    ```

### Deploy a client pod

1. Deploy a client pod.

    ```console
    kubectl apply \
      -f ./client/client.yaml \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]-------------------------------------------+
    |                                                         |
    |     +---------------+                                   |
    |     | Client (curl) |                                   |
    |     +---------------+                                   |
    |                                                         |
    |                                                         |
    |                                                         |
    |  +---------------------------------------------------+  |
    |  |                                   [llm-d Router]  |  |
    |  |                                                   |  |
    |  |  +--------------+               +--------------+  |  |
    |  |  | agentgateway |               |  llm-d EPP   |  |  |
    |  |  +--------------+               +--------------+  |  |
    |  |                                                   |  |
    |  +---------------------------------------------------+  |
    |                                                         |
    |                                                         |
    |                                                         |
    |  +---------------------------------------------------+  |
    |  |                                                   |  |
    |  |                                                   |  |
    |  |    +-----------+  +-----------+  +-----------+    |  |
    |  |    | vLLM      |  | vLLM      |  | vLLM      |    |  |
    |  |    | Simulator |  | Simulator |  | Simulator |    |  |
    |  |    +-----------+  +-----------+  +-----------+    |  |
    |  |                                                   |  |
    |  |                                 [Inference Pool]  |  |
    |  +---------------------------------------------------+  |
    |                                                         |
    +---------------------------------------------------------+
    ```

### Run sample requests

1. Run a shell in the client pod.

    ```console
    kubectl exec -it client -n llm-d -- bash
    ```

1. Check models.

    ```console
    curl -s http://inference-gateway.llm-d.svc.cluster.local/v1/models | jq .
    ```

1. Say "Hello!".

    ```console
    curl -s http://inference-gateway.llm-d.svc.cluster.local/v1/chat/completions \
      -H 'Content-Type: application/json' \
      -d '{
        "model": "Qwen/Qwen3-0.6B",
        "messages": [{"role": "user", "content": "Hello!"}],
        "max_tokens": 50
      }' | jq
    ```
    Note: The vLLM Simulator returns a random predefined sentence. The actual inference is not done in this environment.

    ```console
    +--[Kubernetes]-------------------------------------------+
    |                                                         |
    |     +---------------+                                   |
    |     | Client (curl) |                                   |
    |     +-------+-------+                                   |
    |             |                                           |
    |     (Run OpenAI API)                                    |
    |             |                                           |
    |  +----------+----------------------------------------+  |
    |  |          |                        [llm-d Router]  |  |
    |  |          V                                        |  |
    |  |  +-------+------+               +--------------+  |  |
    |  |  | agentgateway |-------------->|  llm-d EPP   |  |  |
    |  |  +-------+------+               +--------------+  |  |
    |  |          |                                        |  |
    |  +----------+----------------------------------------+  |
    |             |                                           |
    |             +--------------+--------------+             |
    |             |              |              |             |
    |  +----------+--------------+--------------+----------+  |
    |  |          |              |              |          |  |
    |  |          V              V              V          |  |
    |  |    +-----------+  +-----------+  +-----------+    |  |
    |  |    | vLLM      |  | vLLM      |  | vLLM      |    |  |
    |  |    | Simulator |  | Simulator |  | Simulator |    |  |
    |  |    +-----------+  +-----------+  +-----------+    |  |
    |  |                                                   |  |
    |  |                                 [Inference Pool]  |  |
    |  +---------------------------------------------------+  |
    |                                                         |
    +---------------------------------------------------------+
    ```

## Delete the sample deployment

```console
kubectl delete \
  -f ./client/client.yaml \
  -n llm-d
```
```console
helm uninstall \
  llm-d-router-gateway \
  -n llm-d
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
kubectl delete \
  -f ./vllm/vllm-sim.yaml \
  -n llm-d
```
```console
kubectl delete ns \
  agentgateway-system \
  llm-d
```
