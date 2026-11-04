# llm-d with vLLM and SmolLM2-135M-Instruct

Create a sample LLM infrastructure on Kubernetes with:

- vLLM with SmolLM2-135M-Instruct
- Gateway API Inference Extension
- agentgateway as Gateway
- EndpointPicker and InferencePool

## Overview

```console
+--[Kubernetes]--------------------------------+
|                                              |
|     +---------------+                        |
|     | Client (curl) |                        |
|     +------+--------+                        |
|            |                                 |
|     (Run OpenAI API)                         |
|            |                                 |
|  +---------+------------------------------+  |
|  |         |              [llm-d Router]  |  |
|  |         V                              |  |
|  |  +------+-------+    +--------------+  |  |
|  |  | agentgateway |--->|  llm-d EPP   |  |  |
|  |  +------+-------+    +--------------+  |  |
|  |         |                              |  |
|  +---------+------------------------------+  |
|            |                                 |
|            +---------+                       |
|                      |                       |
|  +-------------------+--------------------+  |
|  |                   |                    |  |
|  |                   V                    |  |
|  |            +--------------+            |  |
|  |            | vLLM         |            |  |
|  |            | SmolLM2-135M |            |  |
|  |            +--------------+            |  |
|  |                                        |  |
|  |                      [Inference Pool]  |  |
|  +----------------------------------------+  |
|                                              |
+----------------------------------------------+
```

## Deploy each component on Kubernetes

### Create a Kubernetes namespace

1. Create a Kubernetes namespace.

    ```console
    kubectl create ns llm-d
    ```

1. Verify that the namespace `llm-d` is created.

    ```console
    kubectl get ns llm-d
    ```

### Deploy vLLM with SmolLM2-135M-Instruct

1. Deploy PVC to store model data.

    ```console
    kubectl apply \
      -f ./vllm/pvc-vllm-hf-cache.yaml \
      -n llm-d
    ```

1. Verify that the PVC is deployed.

    ```console
    kubectl get pvc \
      vllm-hf-cache \
      -n llm-d
    ```

1. Deploy vLLM with SmolLM2-135M-Instruct.

    ```console
    kubectl apply \
      -f ./vllm/vllm-smollm2-135m.yaml \
      -n llm-d
    ```

1. Verify that the vLLM pod is deployed.

    ```console
    kubectl get pod \
      -l app=vllm-smollm2-135m \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]--------------------------------+
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |               +--------------+               |
    |               | vLLM         |               |
    |               | SmolLM2-135M |               |
    |               +--------------+               |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    +----------------------------------------------+
    ```

### Deploy CRDs for Gateway API and Gateway API Inference Extension

1. Deploy CRDs for Gateway API and Gateway API Inference Extension.

    ```console
    kubectl apply \
      -f ./gateway-api/gateway-api.yaml \
      -f ./gateway-api/gateway-api-inference-extension.yaml
    ```

1. Verify that CRDs are created.

    ```console
    kubectl api-resources \
      --api-group=gateway.networking.k8s.io
    ```

    ```console
    kubectl api-resources \
      --api-group=inference.networking.k8s.io
    ```

    ```console
    kubectl api-resources \
      --api-group=inference.networking.x-k8s.io
    ```

### Deploy agentgateway as implementation of Gateway (GatewayClass)

1. Deploy CRDs for agentgateway.

    ```console
    helm install agentgateway-crds \
      oci://cr.agentgateway.dev/charts/agentgateway-crds \
      --namespace agentgateway-system \
      --create-namespace \
      --version v1.7.0
    ```

1. Verify that CRDs are created.

    ```console
    kubectl api-resources \
      --categories=agentgateway
    ```

1. Deploy a default configuration for agentgateway.

    ```console
    kubectl apply \
      -f ./agentgateway/agentgateway-parameters.yaml \
      -n agentgateway-system
    ```

1. Verify that agentgatewayparameters is created.

    ```console
    kubectl get agentgatewayparameters \
      clusterip-by-default \
      -n agentgateway-system
    ```

1. Deploy an agentgateway controller.

    ```console
    helm install agentgateway \
      oci://cr.agentgateway.dev/charts/agentgateway \
      --namespace agentgateway-system \
      --values ./agentgateway/agentgateway.yaml \
      --version v1.7.0
    ```

1. Verify that the agentgateway controller is deployed.

    ```console
    kubectl get pod \
      -n agentgateway-system
    ```

1. Verify that the GatewayClass resource is created.

    ```console
    kubectl get gatewayclass
    ```

### Deploy Gateway

1. Deploy Gateway.

    ```console
    kubectl apply \
      -f ./gateway-api/gateway.yaml \
      -n llm-d
    ```

1. Verify that gateway and agentgateway pod are deployed.

    ```console
    kubectl get gateway \
      inference-gateway \
      -n llm-d
    ```

    ```console
    kubectl get pod \
      -l app.kubernetes.io/name=inference-gateway \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]--------------------------------+
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |     +--------------+                         |
    |     | agentgateway |                         |
    |     +--------------+                         |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |               +--------------+               |
    |               | vLLM         |               |
    |               | SmolLM2-135M |               |
    |               +--------------+               |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    +----------------------------------------------+
    ```

### Deploy EPP (Endpoint Picker), InferencePool, and HTTPRoute

1. Deploy EPP (Endpoint Picker), InferencePool, and HTTPRoute.

    ```console
    helm install llm-d-router-gateway \
      oci://ghcr.io/llm-d/charts/llm-d-router-gateway \
      --namespace llm-d \
      --values ./llm-d-router/llm-d-router-gateway.yaml \
      --version v1.0.0
    ```

1. Verify that EPP (Endpoint Picker), InferencePool, and HTTPRoute are deployed.

    ```console
    kubectl get pod \
      -l llm-d-router-gateway=llm-d-router-gateway-epp \
      -n llm-d
    ```

    ```console
    kubectl get inferencepool \
      llm-d-router-gateway \
      -n llm-d
    ```

    ```console
    kubectl get httproute \
      llm-d-router-gateway \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]--------------------------------+
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |                                              |
    |  +----------------------------------------+  |
    |  |                        [llm-d Router]  |  |
    |  |                                        |  |
    |  |  +--------------+    +--------------+  |  |
    |  |  | agentgateway |    |  llm-d EPP   |  |  |
    |  |  +--------------+    +--------------+  |  |
    |  |                                        |  |
    |  +----------------------------------------+  |
    |                                              |
    |                                              |
    |                                              |
    |  +----------------------------------------+  |
    |  |                                        |  |
    |  |                                        |  |
    |  |            +--------------+            |  |
    |  |            | vLLM         |            |  |
    |  |            | SmolLM2-135M |            |  |
    |  |            +--------------+            |  |
    |  |                                        |  |
    |  |                      [Inference Pool]  |  |
    |  +----------------------------------------+  |
    |                                              |
    +----------------------------------------------+
    ```

### Deploy a client pod

1. Deploy a client pod.

    ```console
    kubectl apply \
      -f ./client/client.yaml \
      -n llm-d
    ```

1. Verify that the client pod is deployed.

    ```console
    kubectl get pod \
      client \
      -n llm-d
    ```

    ```console
    +--[Kubernetes]--------------------------------+
    |                                              |
    |     +---------------+                        |
    |     | Client (curl) |                        |
    |     +---------------+                        |
    |                                              |
    |                                              |
    |                                              |
    |  +----------------------------------------+  |
    |  |                        [llm-d Router]  |  |
    |  |                                        |  |
    |  |  +--------------+    +--------------+  |  |
    |  |  | agentgateway |    |  llm-d EPP   |  |  |
    |  |  +--------------+    +--------------+  |  |
    |  |                                        |  |
    |  +----------------------------------------+  |
    |                                              |
    |                                              |
    |                                              |
    |  +----------------------------------------+  |
    |  |                                        |  |
    |  |                                        |  |
    |  |            +--------------+            |  |
    |  |            | vLLM         |            |  |
    |  |            | SmolLM2-135M |            |  |
    |  |            +--------------+            |  |
    |  |                                        |  |
    |  |                      [Inference Pool]  |  |
    |  +----------------------------------------+  |
    |                                              |
    +----------------------------------------------+
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
        "model": "HuggingFaceTB/SmolLM2-135M-Instruct",
        "messages": [{"role": "user", "content": "Hello!"}],
        "max_tokens": 50
      }' | jq
    ```
    Note: It may take a bit of time to return a response since this sample uses vLLM with CPU for inference.

    ```console
    +--[Kubernetes]--------------------------------+
    |                                              |
    |     +---------------+                        |
    |     | Client (curl) |                        |
    |     +------+--------+                        |
    |            |                                 |
    |     (Run OpenAI API)                         |
    |            |                                 |
    |  +---------+------------------------------+  |
    |  |         |              [llm-d Router]  |  |
    |  |         V                              |  |
    |  |  +------+-------+    +--------------+  |  |
    |  |  | agentgateway |--->|  llm-d EPP   |  |  |
    |  |  +------+-------+    +--------------+  |  |
    |  |         |                              |  |
    |  +---------+------------------------------+  |
    |            |                                 |
    |            +---------+                       |
    |                      |                       |
    |  +-------------------+--------------------+  |
    |  |                   |                    |  |
    |  |                   V                    |  |
    |  |            +--------------+            |  |
    |  |            | vLLM         |            |  |
    |  |            | SmolLM2-135M |            |  |
    |  |            +--------------+            |  |
    |  |                                        |  |
    |  |                      [Inference Pool]  |  |
    |  +----------------------------------------+  |
    |                                              |
    +----------------------------------------------+
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
  -f ./vllm/vllm-smollm2-135m.yaml \
  -n llm-d
```
```console
kubectl delete \
  -f ./vllm/pvc-vllm-hf-cache.yaml \
  -n llm-d
```
```console
kubectl delete ns \
  agentgateway-system \
  llm-d
```
