# Documentation: CI/CD and GitOps Workflow

This document explains the project's fully automated CI/CD pipeline, which is designed for a multi-repo structure and enables a seamless GitOps workflow with Argo CD.

## 1. Git Strategy: Multi-Repo

The project is split into multiple repositories to enforce separation of concerns:

- **Application Repositories** (e.g., `spitikos/homepage`): Contain the source code for a single application.
- **`spitikos/charts`**: A dedicated repository containing all Helm charts.
- **`spitikos/spitikos`**: The central repository containing Argo CD manifests, documentation, and reusable CI/CD workflows.

## 2. The CI/CD Pipeline: From Code to Live Deployment

The pipeline is a chain reaction that starts with a `git push` to an application repository and ends with a live deployment, with no manual intervention required.

### Step 1: The Trigger (Application Repository)

- **Location:** Each application repository (e.g., `spitikos/homepage`) contains its own CI workflow file (e.g., `.github/workflows/ci.yaml`).
- **Trigger:** The workflow is configured to run on pushes to the `main` branch.

### Step 2: Reusable Release Workflow (`spitikos/spitikos`)

- **Location:** The application's workflow uses the `uses:` clause to call a **reusable workflow** named `release.yaml` located in the central `spitikos/spitikos` repository.
- **Purpose:** This reusable workflow is responsible for the entire release process.

#### Job 1: `docker-publish`

- Builds a single-platform (`linux/arm64`) Docker image for the application.
- Tags the image with the short commit SHA of the triggering commit.
- Pushes the new image to the GitHub Container Registry (ghcr.io).

#### Job 2: `set-package-public`

- This job runs after `docker-publish` succeeds.
- It uses the GitHub CLI to programmatically set the visibility of the newly published package to **public**. This ensures all application images are accessible to the cluster without requiring image pull secrets.

#### Job 3: `update-chart`

- This job runs only after `docker-publish` succeeds.
- It checks out the **`spitikos/charts`** repository.
- It finds the `values.yaml` file for the specific application (e.g., `charts/homepage/values.yaml`).
- It replaces the `image.tag` value with the new commit SHA tag from the `docker-publish` job.
- It commits this change directly back to the `spitikos/charts` repository with a message like `ci: deploy homepage version <sha>`.
- It pushes the commit to `main`.

### Step 3: Argo CD (The Deployer)

- The `git push` to the `spitikos/charts` repository is the trigger for the final step.
- Argo CD, which is constantly monitoring this repository, detects the new commit.
- Because its sync policy is set to `automated`, it immediately begins a sync process.
- It pulls the new Helm chart configuration, renders the templates with the updated image tag, and applies the changes to the cluster.

The new version of the application is now live. This entire process, from code to deployment, is fully automated.
