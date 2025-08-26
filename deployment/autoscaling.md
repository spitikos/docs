# Documentation: Autoscaling with KEDA

This document explains the event-driven autoscaling strategy used for applications in this project, which is powered by KEDA.

## 1. Architecture: KEDA and Prometheus

To enable scaling based on real-time traffic, we use a combination of KEDA and the existing Prometheus monitoring stack.

- **KEDA (Kubernetes Event-driven Autoscaling):** KEDA is a CNCF-graduated project that extends the standard Kubernetes Horizontal Pod Autoscaler (HPA) with more advanced capabilities. It acts as a metrics provider, gathering data from various sources (called "scalers").
- **Prometheus Scaler:** We use KEDA's built-in Prometheus scaler to query the cluster's Prometheus instance for application-specific metrics.
- **NGINX Metrics:** The NGINX Ingress Controller automatically exposes a Prometheus metric called `nginx_ingress_controller_requests`, which includes a `service` label identifying the backend application. This is the key metric we use to measure traffic.

The most important feature KEDA provides is the ability to **scale deployments down to zero replicas** when there is no traffic, and scale them back up to one when the first request comes in. This is ideal for the resource-constrained Raspberry Pi environment.

## 2. Implementation

The entire autoscaling configuration is managed declaratively via the GitOps workflow.

### 2.1. KEDA Installation

KEDA is installed and managed via a dedicated **wrapper chart** located in the `spitikos/charts` repository at `charts/keda`. This chart is deployed by the root Argo CD application, just like any other platform service.

The configuration in `charts/keda/values.yaml` sets resource limits for the KEDA components and enables a `ServiceMonitor` so that Prometheus can scrape KEDA's own metrics.

### 2.2. The Common `ScaledObject` Template

To make autoscaling easy to apply and consistent, a template for the KEDA `ScaledObject` custom resource has been added to the `_common` library chart.

This template is controlled by a top-level `scaledobject:` key in an application's `values.yaml`. If the key is present, a `ScaledObject` will be rendered for that application.

### 2.3. Enabling Autoscaling for an Application

To enable autoscaling for an application (e.g., `homepage`), two changes are required in its `values.yaml` file:

1.  **Add the `scaledobject` section:** This defines the Prometheus query and the request threshold that will trigger a scale-up event.

    ```yaml
    # homepage/values.yaml
    scaledobject:
      minReplicas: 0
      maxReplicas: 3
      prometheus:
        query: 'sum(rate(nginx_ingress_controller_requests{service="homepage"}[2m]))'
        threshold: "10" # Target 10 requests per second per replica
    ```

2.  **Configure Argo CD to Ignore Replicas:** The application's definition in the root `argocd/values.yaml` file must be updated to tell Argo CD to ignore the `replicas` field of the deployment. This prevents Argo CD and KEDA from fighting over control of the pod count.

    ```yaml
    # argocd/values.yaml
    - name: homepage
      # ... other fields
      ignoreDifferences:
        - group: apps
          kind: Deployment
          jsonPointers:
            - /spec/replicas
    ```

With these changes committed to Git, the application will be fully managed by KEDA, scaling up and down based on real traffic.