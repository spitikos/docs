# Documentation: Naming Conventions

This document outlines the strict naming conventions used across the project. Consistency is critical for the automation and GitOps workflows to function correctly.

## 1. Resource Naming

| Resource Type            | Convention   | Example                     |
| :----------------------- | :----------- | :-------------------------- |
| **Git Repository**       | `<app-name>` | `homepage`, `charts`        |
| **Docker Image**         | `<app-name>` | `ghcr.io/spitikos/homepage` |
| **Argo CD Application**  | `<app-name>` | `homepage`                  |
| **Kubernetes Namespace** | `<app-name>` | `homepage`                  |
| **Helm Chart Name**      | `<app-name>` | `homepage`                  |
| **Kubernetes Service**   | `<app-name>` | `homepage`                  |

### Summary of Rules:

- **Rule:** All resources, whether in-cluster (like **Namespaces** and **Services**) or external (like **Git Repositories** and **Docker Images**), should be named simply after the application's purpose, **without any prefixes** (e.g., `homepage`, `charts`, `traefik`).
- **Rule:** Docker images must be published under the `spitikos` organization on `ghcr.io`.
- **Exception:** Third-party applications that are not under our direct control (e.g., `kube-prometheus-stack`) do not follow these naming rules.
