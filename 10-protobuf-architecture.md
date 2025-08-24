# Documentation: Protobuf API Architecture

This document outlines the architecture for managing and consuming Protobuf API definitions across different services in the project.

## 1. Core Principle: Decentralized Schema with Buf.build

To ensure a single source of truth for all API contracts, this project uses the [Buf.build Schema Registry](https://buf.build/). This approach avoids the need for a dedicated Git repository to store `.proto` files or generated code.

The key principles are:

- **Decentralized:** The `.proto` files for a given service live within that service's own source code repository.
- **Centralized Registry:** All services push their Protobuf schemas to a single, central registry on Buf.build under the `spitikos` organization.
- **Automated SDK Generation:** Buf.build automatically generates client SDKs for multiple languages and publishes them to common package registries (like npm for TypeScript or Go modules).
- **Decoupling:** Consumer applications are completely decoupled from the `protoc` toolchain. They simply import the pre-generated, versioned SDK as a standard package dependency.

## 2. Consumption Strategy: Internal gRPC and Public SDKs

Both backend and frontend services consume the API definitions as standard, pre-generated packages from their respective registries.

- **Internal Communication**: Backend services communicate with each other via gRPC using the Kubernetes FQDN (e.g., `api.api.svc.cluster.local:50051`). This traffic does not leave the cluster.
- **External Communication**: Frontend applications consume a generated TypeScript client SDK from the npm registry. This SDK makes standard `fetch` requests to a public-facing API gateway, which then communicates with the internal gRPC services.

### Example Workflow

1.  **Schema Definition:** A developer defines or modifies a `.proto` file inside a backend service's repository (e.g., `spitikos/auth-service/proto/auth.proto`).
2.  **CI Push:** The service's CI/CD pipeline includes a step to run `buf push`, which pushes the schema to the `spitikos` organization on the Buf Schema Registry.
3.  **SDK Generation:** Buf automatically generates the necessary client code. For a frontend application, it would publish a new version of an npm package like `@spitikos/auth-service-sdk`.
4.  **Dependency Update:** A frontend developer can then update their `package.json` to the new version of the SDK to get access to the new, fully-typed client.

This modern, registry-based approach provides strong consistency and versioning guarantees while simplifying the development workflow significantly.
