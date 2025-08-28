# Core Concept: Helm Chart Architecture

This document explains the reusable and maintainable Helm chart architecture used in this project. The design is based on the idiomatic **Wrapper Chart** pattern, with all charts being centrally managed in the dedicated `spitikos/charts` repository.

## 1. Core Concept: The Common Library Chart

The architecture is built around a single library chart that provides common, reusable templates.

- **`charts/_common`**: This is a `library` chart that provides templates for a `Deployment`, `Service`, `Ingress`, `ConfigMap`, and KEDA `ScaledObject`. It cannot be deployed by itself. Its purpose is to enforce consistency and reduce duplication.

### The "If It Exists, Render It" Pattern

The common templates use a simple but powerful pattern: if a top-level key exists in an application's `values.yaml` (e.g., `ingress:`), the corresponding resource will be rendered. If the key is omitted, the resource is skipped.

This makes an application's `values.yaml` a clear, declarative manifest of which components it needs, without requiring redundant `enabled: true` flags.

## 2. Application Charts (e.g., `charts/homepage`)

Our own applications use a simple wrapper chart pattern:

1.  **`Chart.yaml`**: Defines the application's metadata and declares a dependency on our local `_common` library chart.

    ```yaml
    dependencies:
      - name: common
        version: "0.1.0"
        repository: "file://../_common"
    ```

2.  **`values.yaml`**: Provides the specific values that the `_common` chart's templates will use. This is where we define the application's unique properties. The presence of a top-level key like `ingress:` or `scaledobject:` is enough to enable that feature.

3.  **`templates/common.yaml`**: Contains a simple, single file that explicitly calls all the desired templates from the `_common` chart.

    **Example: `charts/homepage/templates/common.yaml`**

    ```yaml
    {{- include "common.deployment" . | nindent 0 }}
    ---
    {{- include "common.service" . | nindent 0 }}
    ---
    {{- include "common.ingress" . | nindent 0 }}
    ---
    {{- include "common.scaledobject" . | nindent 0 }}
    ```

    This tells Helm: "For the `homepage` chart, render these common components using the values from `homepage/values.yaml`."

## 3. Platform Service Charts (e.g., `charts/ingress-nginx`)

Third-party services are also managed using the wrapper chart pattern. This allows us to manage their configuration declaratively and deploy them with Argo CD.

1.  **`Chart.yaml`**: Declares a dependency on the official third-party chart (e.g., `ingress-nginx/ingress-nginx`).
2.  **`values.yaml`**: Contains a top-level key matching the dependency's name (e.g., `ingress-nginx:`) that passes a block of configuration directly to the subchart.
3.  **`templates/`**: Can contain any additional custom resources we want to layer on top, though this is often not needed.

This pattern gives us the power of official, community-maintained charts while retaining full control over configuration in a way that integrates perfectly with our GitOps workflow.
