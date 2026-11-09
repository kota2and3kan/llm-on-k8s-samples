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
|     +---------------+                              |
|     |    kagent     |                              |
|     +------+--------+                              |
|            |                                       |
|     (Run OpenAI API)                               |
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

1. Verify that the namespace `llm-d` is created.

    ```console
    kubectl get ns llm-d
    ```

### Deploy cert-manager

1. Deploy cert-manager as a dependency of KServe.

    ```console
    helm install cert-manager \
      ./charts/cert-manager-v1.21.3.tgz \
      --namespace cert-manager \
      --create-namespace \
      --set crds.enabled=true \
      --set crds.keep=false
    ```

1. Verify that cert-manager pods are deployed.

    ```console
    kubectl get pod -n cert-manager
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
      ./charts/agentgateway-crds-v1.7.0.tgz \
      --namespace agentgateway-system \
      --create-namespace
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
      ./charts/agentgateway-v1.7.0.tgz \
      --namespace agentgateway-system \
      --values ./agentgateway/agentgateway.yaml
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
      ./charts/kserve-llmisvc-crd-v0.21.0.tgz \
      --namespace kserve \
      --create-namespace
    ```

1. Verify that CRDs are created.

    ```console
    kubectl api-resources \
      --api-group=serving.kserve.io
    ```

1. Deploy a KServe llmisvc controller.

    ```console
    helm install kserve-llmisvc-resources \
      ./charts/kserve-llmisvc-resources-v0.21.0.tgz \
      --namespace kserve \
      --set kserve.llmisvc.createGIECRDs=false \
      --set kserve.llmisvc.controller.imagePullPolicy=IfNotPresent \
      --wait
    ```

1. Verify that the KServe llmisvc controller is deployed.

    ```console
    kubectl get pod \
      -l app.kubernetes.io/name=llmisvc-controller-manager \
      -n kserve
    ```

1. Deploy KServe configs.

    ```console
    helm install kserve-runtime-configs \
      ./charts/kserve-runtime-configs-v0.21.0.tgz \
      --namespace kserve \
      --set kserve.llmisvcConfigs.enabled=true
    ```

1. Verify that KServe configs are deployed.

    ```console
    kubectl get llminferenceserviceconfigs \
      -n kserve
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

1. Verify that the LLMInferenceService resource is deployed.

    ```console
    kubectl get llmisvc \
      -n llm-d
    ```

1. Verify that the vLLM pod is deployed.

    ```console
    kubectl get pod \
      -l app.kubernetes.io/component=llminferenceservice-workload \
      -n llm-d
    ```

1. Verify that EPP (Endpoint Picker), InferencePool, and HTTPRoute are deployed.

    ```console
    kubectl get pod \
      -l app.kubernetes.io/component=llminferenceservice-router-scheduler \
      -n llm-d
    ```

    ```console
    kubectl get inferencepool \
      smollm2-135m-inference-pool \
      -n llm-d
    ```

    ```console
    kubectl get httproute \
      smollm2-135m-kserve-route \
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

### Deploy kagent

1. Create a Kubernetes namespace.

    ```console
    kubectl create ns kagent
    ```

1. Verify that the namespace `kagent` is created.

    ```console
    kubectl get ns kagent
    ```

1. Create a dummy secret.

    ```console
    kubectl create secret generic kagent-openai \
      --from-literal=OPENAI_API_KEY='dummy-api-key' \
      -n kagent
    ```

1. Verify that the secret is created.

    ```console
    kubectl get secret \
      kagent-openai \
      -n kagent
    ```

1. Deploy CRDs for kagent.

    ```console
    helm install kagent-crds \
      ./charts/kagent-crds-0.10.4.tgz \
      --namespace kagent
    ```

1. Verify that CRDs are created.

    ```console
    kubectl api-resources \
      --categories=kagent
    ```

1. Deploy kagent.

    ```console
    helm install kagent \
      ./charts/kagent-0.10.4.tgz \
      --namespace kagent \
      --values ./kagent/kagent.yaml
    ```

1. Verify that kagent is deployed.

    ```console
    kubectl get pod \
      -n kagent
    ```

1. Deploy a sample agent.

    ```console
    kubectl apply \
      -f ./kagent/sample-agent.yaml \
      -n kagent
    ```

1. Verify that the sample agent is deployed.

    ```console
    kubectl get agent \
      sample-agent \
      -n kagent
    ```

    ```console
    kubectl get pod \
      -l kagent=sample-agent \
      -n kagent
    ```

    ```console
    +--[Kubernetes]--------------------------------------+
    |                                                    |
    |     +---------------+                              |
    |     |    kagent     |                              |
    |     +------+--------+                              |
    |            |                                       |
    |     (Run OpenAI API)                               |
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

## Appendix

1. Run the `kubectl port-forward` command.

    ```console
    kubectl port-forward \
      pod/$(kubectl get pod -l app.kubernetes.io/name=inference-gateway -o jsonpath='{.items[0].metadata.name}' -n llm-d) \
      15000:15000 \
      -n llm-d
    ```

1. Access http://localhost:15000/.

1. Sample request for CEL Playground.

    ```yaml
    request:
      method: POST
      path: /v1/chat/completions
      host: example.com
      headers:
        authorization: Bearer <token>
    jwt:
      iss: https://idp.example.com
      sub: alice
    ```

1. Sample expression.

    ```console
    jwt.iss == "https://idp.example.com" && jwt.sub == "alice"
    ```
