# Documentation: Helm Chart Architecture

This document explains the reusable and maintainable Helm chart architecture used in this project. The design is based on the idiomatic **Wrapper Chart** pattern, with all charts being centrally managed in the dedicated `spitikos/charts` repository. This pattern is used for both our own applications and for third-party platform services like Traefik.

## 1. Core Concept: Wrapper and Common Charts

The architecture consists of two types of charts:

- **Common Chart:** The `charts/_common` chart is a library chart that provides reusable, generic templates for a `Deployment`, `Service`, and `Ingress`. It cannot be deployed itself.
- **Wrapper Charts:** These are deployable charts that "wrap" the `_common` chart as a dependency. They provide a layer of configuration (`values.yaml`) that the common templates use.

### NGINX Ingress Template

The `_common/templates/_ingress.tpl` is the heart of the ingress configuration. It generates a standard Kubernetes `Ingress` resource. It is designed to be flexible, allowing `values.yaml` to pass in any necessary annotations. This is how we enable gRPC:

**`api/values.yaml`:**
```yaml
ingress:
  enabled: true
  host: api.spitikos.dev
  annotations:
    # This annotation is passed through to the Ingress template
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
```

This pattern keeps the application charts simple while allowing for powerful, per-app ingress configuration.

## 2. Application Charts (e.g., `charts/homepage`)

Our own applications use a simple wrapper chart pattern:

1.  **`Chart.yaml`**: Defines the application's metadata and declares a dependency on our local `_common` library chart.

    ```yaml
    dependencies:
      - name: common
        version: "0.1.0"
        repository: "file://../_common"
    ```

2.  **`values.yaml`**: Provides the specific values that the `_common` chart's templates will use. This is where we define the application's unique properties, such as its container image, ingress hostname, and service port.

3.  **`templates/` directory**: Contains simple, one-line "pass-through" template files. Each file explicitly calls a corresponding template from the `_common` chart.

    **Example: `charts/homepage/templates/manifests.yaml`**

    ```yaml
    { { - include "common.deployment" . | nindent 0 } }
    ---
    { { - include "common.service" . | nindent 0 } }
    ---
    { { - include "common.ingress-route" . | nindent 0 } }
    ```

    This tells Helm: "For the `homepage` chart, render a Deployment, Service, and IngressRoute using the templates from `_common` and the values from `homepage/values.yaml`."

    > **Note:** For details on the ongoing issues with routing gRPC traffic via `IngressRoute`, see the `12-grpc-routing-issue.md` document.

## 3. Platform Service Charts (e.g., `charts/traefik`)

Third-party services are also managed using the wrapper chart pattern. This allows us to manage their configuration declaratively and deploy them with Argo CD.

1.  **`Chart.yaml`**: Declares a dependency on the official third-party chart (e.g., `traefik/traefik`).
2.  **`values.yaml`**: Contains two sections:
    - A top-level key (e.g., `traefik:`) that passes a block of configuration directly to the subchart.
    - A section for values used by custom templates in our wrapper chart (e.g., the `ingress:` block for the dashboard).
3.  **`templates/`**: Contains any additional custom resources we want to layer on top, such as the `IngressRoute` for the Traefik dashboard.

This pattern gives us the power of official, community-maintained charts while retaining full control over configuration in a way that integrates perfectly with our GitOps workflow.
