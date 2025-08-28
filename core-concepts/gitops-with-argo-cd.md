# Core Concept: GitOps with Argo CD

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

### Installation

Argo CD is a core platform component. It is installed and managed via a dedicated **wrapper chart** located in the `spitikos/charts` repository at `charts/argocd`.

Unlike other applications, this root chart is not managed by Argo CD itself (to avoid a "chicken-and-egg" problem). Instead, it is installed and upgraded manually via a `helm` command, which has been added to the project's `taskfile.yaml` for convenience.

### "App of Apps" Pattern

We use the "App of Apps" pattern to manage our application landscape. The `charts/argocd` chart is the "root" application.

- **Single Source of Truth:** The `charts/argocd/values.yaml` file contains a list of all applications that should be deployed to the cluster.
- **Application Discovery:** The `charts/argocd/templates/applications.yaml` template ranges over this list and generates a corresponding Argo CD `Application` manifest for each entry.
- **Child Applications:** Each generated `Application` manifest points to its own Helm chart in the **`spitikos/charts`** repository.

This pattern ensures that adding a new application to the cluster is as simple as adding a new entry to the `applications` list in the root `values.yaml` file and committing it to Git.

### Handling Autoscaling

When an application is configured for autoscaling with KEDA, the number of pods is controlled by the autoscaler, not by the `replicaCount` value in Git. This creates a conflict that must be resolved.

We instruct Argo CD to ignore the replica count for autoscaled applications by adding an `ignoreDifferences` block to their definition in the `argocd/values.yaml` file.

```yaml
- name: homepage
  # ... other fields
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

This tells Argo CD to allow the KEDA autoscaler to manage the pod count freely without marking the application as `OutOfSync`.
