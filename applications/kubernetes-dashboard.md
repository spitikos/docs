# Documentation: Kubernetes Dashboard

This document details the deployment of the Kubernetes Dashboard, a general-purpose web UI for Kubernetes clusters.

## 1. Architecture: Declarative Deployment via GitOps

Like all other components in this project, the Kubernetes Dashboard is deployed declaratively via GitOps.

- **Wrapper Chart:** We use a wrapper chart located in the `spitikos/charts` repository at `charts/kube-dashboard`. This chart includes the official `kubernetes-dashboard/kubernetes-dashboard` chart as a dependency.
- **Declarative Configuration:** All configuration is defined in `charts/kube-dashboard/values.yaml` in the `spitikos/charts` repository.
- **GitOps Management:** The dashboard is deployed and managed by an Argo CD `Application` manifest in the `spitikos/spitikos` repository. To deploy, modify, or remove the dashboard, a change is made in the `spitikos/charts` repository and pushed to Git. Argo CD handles the rest.

## 2. Ingress and Networking

The dashboard is exposed at **https://kube.spitikos.dev**.

The wrapper chart contains a standard `Ingress` template that is managed by the NGINX Ingress Controller.

## 3. Authentication: Service Account and Token

The dashboard is configured to use token-based authentication. The Helm chart creates a `ServiceAccount` named `admin-user` with `cluster-admin` privileges.

### How to Log In

1.  **Generate a Login Token:** From your local machine, run the following command. The token is short-lived by default.

    ```bash
    kubectl create token admin-user -n kube-dashboard
    ```

2.  **Copy the token** that is printed to the console.

3.  **Access the Dashboard:**
    - Navigate to **https://kube.spitikos.dev**.
    - Select the **Token** authentication method.
    - Paste the token into the field and click **Sign in**.
