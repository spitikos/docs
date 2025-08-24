# Documentation: GitOps with Argo CD

This document describes the GitOps workflow used in this project, which is powered by Argo CD. GitOps is a paradigm where the entire desired state of the system is declaratively defined in a Git repository.

## 1. GitOps Workflow

The automated deployment workflow is as follows:

1.  A developer pushes code to an application repository (e.g., `spitikos/homepage`).
2.  The application's CI pipeline builds a new Docker image and pushes it to `ghcr.io`.
3.  The CI pipeline then checks out the `spitikos/charts` repository and updates the image `tag` in the corresponding Helm chart's `values.yaml` file.
4.  **Argo CD**, which constantly monitors the `spitikos/charts` repository, detects the change to `values.yaml`.
5.  Argo CD automatically "syncs" the application. It renders the Helm chart with the new values and applies the resulting manifests to the cluster. The new version is now live.

The `spitikos/charts` repository is the single source of truth for what is running in the cluster.

## 2. Argo CD Architecture

### Installation and Configuration

Argo CD is a core platform component, but it is also managed via GitOps just like any other application.

- **Wrapper Chart:** The configuration is defined in a dedicated **wrapper chart** located in the `spitikos/charts` repository at `charts/argocd`. This chart includes the official `argo-cd` chart as a dependency.
- **Declarative Configuration:** All configuration, including ingress, is defined in the `charts/argocd/values.yaml` file.
- **GitOps Management:** The entire Argo CD deployment is managed by an Argo CD `Application` manifest located in the `spitikos/spitikos` repository at `argocd/apps/argocd.yaml`.

This self-management pattern ensures that the configuration for Argo CD itself is version-controlled and reproducible.

### Ingress and Access

Argo CD is accessible at **https://argocd.spitikos.dev**. The official Helm chart has built-in capabilities to create a correctly configured `Ingress` resource for NGINX, which is enabled in our `values.yaml`.

### "App of Apps" Pattern

We use the "App of Apps" pattern to manage our application landscape.

- **Root Application:** A single `Application` manifest, `argocd/root-app.yaml` in the `spitikos/spitikos` repo, is manually applied to the cluster after the initial Argo CD installation. This is the only imperative step.
- **App Discovery:** The `root` application is configured to monitor the `argocd/apps/` directory in the `spitikos/spitikos` repository.
- **Child Applications:** For every `.yaml` file in the `argocd/apps/` directory, Argo CD automatically creates a corresponding `Application`. Each of these files defines a service (e.g., `homepage`, `nginx`), pointing to its Helm chart in the **`spitikos/charts`** repository.

This pattern ensures that adding a new application to the cluster is as simple as adding a new `Application` manifest to the `argocd/apps/` directory and committing it to Git.