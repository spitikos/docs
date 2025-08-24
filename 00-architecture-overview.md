# Documentation: Architecture Overview

This document provides a high-level overview of the architecture, design patterns, and core concepts used in this project.

## 1. Core Philosophy: GitOps & Multi-Repo

The entire project is built around a **GitOps** philosophy using a **multi-repo** approach. The desired state of all applications, infrastructure, and configurations is defined declaratively across several key repositories under the `spitikos` GitHub organization. [Argo CD](https://argo-cd.readthedocs.io/) acts as the GitOps operator, continuously ensuring the live state of the cluster matches the state defined in Git.

The key principles are:

- **Declarative:** All configuration is declarative code, primarily Kubernetes and Helm manifests.
- **Versioned and Auditable:** All changes are Git commits, providing a clear audit trail and the ability to revert changes.
- **Automated:** The entire lifecycle, from code commit to live deployment, is automated.
- **Separation of Concerns:** Each part of the platform (core configuration, charts, applications) has its own dedicated repository.
- **Don't Repeat Yourself (DRY):** Common patterns are abstracted into reusable components (the `common` Helm chart, reusable GitHub workflows).

## 2. System Components

The platform consists of several key layers:

| Layer                       | Technology        | Purpose                                                                                            |
| :-------------------------- | :---------------- | :------------------------------------------------------------------------------------------------- |
| **Hardware**                | Raspberry Pi 5    | The physical server running the platform.                                                          |
| **Operating System**        | Ubuntu Server     | The base OS for the Raspberry Pi.                                                                  |
| **Container Orchestration** | k3s               | A lightweight, certified Kubernetes distribution.                                                  |
| **Public Ingress**          | Cloudflare Tunnel | Provides a secure, outbound-only connection to the Cloudflare network, handling TLS termination.   |
| **Internal Ingress**        | NGINX             | Routes traffic within the cluster from the tunnel to the correct service based on hostname.        |
| **Application Packaging**   | Helm              | Manages Kubernetes deployments using a reusable "wrapper chart" pattern.                           |
| **Continuous Integration**  | GitHub Actions    | Builds and pushes container images; automatically updates Helm values in the `charts` repository.  |
| **Continuous Deployment**   | Argo CD           | Detects changes in the `charts` repository and automatically syncs them to the Kubernetes cluster. |

## 3. Project Structure

The platform is organized across multiple repositories in the `spitikos` organization:

- **`spitikos/spitikos`**: The central repository. It contains:
  - `argocd/`: Argo CD application manifests that define what should be deployed.
  - `.github/workflows/`: Reusable GitHub Actions workflows for CI/CD.
  - `docs/`: All project documentation.
- **`spitikos/charts`**: Contains all the Helm charts for deploying applications and platform services.
- **`spitikos/<app-name>`** (e.g., `spitikos/homepage`): Each application has its own repository containing its source code and a CI workflow file.
