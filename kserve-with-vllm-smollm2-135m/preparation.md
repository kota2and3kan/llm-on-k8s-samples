## Preparation for demo

1. Download Helm charts to the `./charts/` directory.

    ```console
    mkdir ./charts
    ```

    ```console
    helm pull \
      oci://quay.io/jetstack/charts/cert-manager \
      --version v1.21.3 \
      --destination ./charts
    ```

    ```console
    helm pull \
      oci://cr.agentgateway.dev/charts/agentgateway-crds \
      --version v1.7.0 \
      --destination ./charts
    ```

    ```console
    helm pull \
      oci://cr.agentgateway.dev/charts/agentgateway \
      --version v1.7.0 \
      --destination ./charts
    ```

    ```console
    helm pull \
      oci://ghcr.io/kserve/charts/kserve-llmisvc-crd \
      --version v0.21.0 \
      --destination ./charts
    ```

    ```console
    helm pull \
      oci://ghcr.io/kserve/charts/kserve-llmisvc-resources \
      --version v0.21.0 \
      --destination ./charts
    ```

    ```console
    helm pull \
      oci://ghcr.io/kserve/charts/kserve-runtime-configs \
      --version v0.21.0 \
      --destination ./charts
    ```

    ```console
    helm pull \
      oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
      --version 0.10.4 \
      --destination ./charts
    ```

    ```console
    helm pull \
      oci://ghcr.io/kagent-dev/kagent/helm/kagent \
      --version 0.10.4 \
      --destination ./charts
    ```

1. Verify that the Helm charts are downloaded.

    ```console
    ls ./charts/
    ```

1. Create a Kubernetes namespace.

    ```console
    kubectl create ns llm-d
    ```

1. Create a PVC to store the model `SmolLM2-135M`.

    ```console
    kubectl apply \
      -f ./preparation/pvc-model.yaml \
      -n llm-d
    ```

1. Download model.

    ```console
    kubectl apply \
      -f ./preparation/job-download-smollm2-135m.yaml \
      -n llm-d
    ```
