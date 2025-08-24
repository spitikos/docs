# Documentation: NGINX Ingress Controller

This document covers the installation and configuration of the NGINX Ingress Controller, which is responsible for routing external traffic to the correct services within the Kubernetes cluster.

## 1. Architecture: Declarative Deployment via GitOps

NGINX is a critical piece of the platform infrastructure. As such, it is managed declaratively using the same GitOps principles as our own applications.

- **Wrapper Chart:** The configuration is defined in a dedicated **wrapper chart** located in the `spitikos/charts` repository at `charts/nginx`. This chart includes the official `ingress-nginx` chart as a dependency.
- **Declarative Configuration:** All configuration is defined in the `charts/nginx/values.yaml` file in the `spitikos/charts` repository.
- **GitOps Management:** The entire NGINX deployment is managed by an Argo CD `Application` manifest located in the `spitikos/spitikos` repository at `argocd/apps/nginx.yaml`. Argo CD ensures that the NGINX deployment in the cluster always matches the configuration defined in the `charts` repository.

## 2. Key Configuration Choices

The configuration in `charts/nginx/values.yaml` is minimal, relying on the official chart's sensible defaults.

- `service.type: LoadBalancer`: This is the default in the official chart. It tells the K3s Service Load Balancer (Klipper) to expose NGINX on the Raspberry Pi's local IP address on standard ports (80 and 443).
- `config.use-forwarded-headers: "true"`: This is a crucial setting in the K3s environment. It tells NGINX to trust the `X-Forwarded-*` headers set by the Klipper Load Balancer, ensuring the application receives the real client IP address.

## 3. Exposing Services

Services are exposed using the standard Kubernetes `Ingress` resource. To maintain consistency, all application charts use a common template. For details on this, and for the specific annotations required for gRPC, see the main `12-ingress-architecture.md` document.
