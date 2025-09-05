# Secret Management with HashiCorp Vault

This document outlines the platform's strategy for managing secrets, which leverages HashiCorp Vault to dynamically inject secrets directly into application pods at runtime. This approach is designed to work securely with **distroless** container images that do not have a shell.

## Mechanism: Agent Injector with Exec Mode

The core of our secret management is the Vault Kubernetes Injector, which is deployed cluster-wide. The injector identifies pods that are annotated for secret injection and modifies their specification to change how the application container is started.

The process is as follows:

1.  An application's Helm chart is configured with specific `podAnnotations` to request secrets from Vault and to enable the injector's "exec" mode.
2.  When the pod starts, the Vault injector mutates its definition. It changes the entrypoint of the main application container to be the `vault-agent` binary.
3.  The `vault-agent` process starts first inside the container. It authenticates to Vault using the pod's Kubernetes Service Account.
4.  Upon successful authentication, the agent fetches the secrets from the path specified in the annotations and renders them in a `KEY=VALUE` format.
5.  The `vault-agent` then uses the `exec` syscall to replace itself with the application's original binary (e.g., `/server`), passing the fetched secrets in as environment variables.

This ensures that the application starts with all its required secrets available as environment variables, without them ever being stored in the container image or in a Kubernetes `Secret`. The `exec` method is critical as it maintains a single process (PID 1) in the container, which is a containerization best practice.

## Example Implementation

The `api` service is a reference implementation for this pattern. The following snippet from its `values.yaml` enables secret injection for a distroless container:

```yaml
deployment:
  # ...
  podAnnotations:
    # Enable the injector
    vault.hashicorp.com/agent-inject: "true"
    
    # Tell the injector to use "exec" mode, not the default "sidecar" mode
    vault.hashicorp.com/agent-run-as-sidecar: "false"
    
    # Specify the Vault role to authenticate against
    vault.hashicorp.com/role: "api"
    
    # The command for the agent to execute after injecting secrets
    vault.hashicorp.com/agent-exec-command: "/server"
    
    # Define which secret to fetch
    vault.hashicorp.com/agent-inject-secret-config.env: "secret/data/spitikos/api"
    
    # Define the template for rendering the secret into KEY=VALUE format
    vault.hashicorp.com/agent-inject-template-config.env: |
      {{- with secret "secret/data/spitikos/api" -}}
      {{- range $key, $value := .Data.data }}
      {{ $key }}={{ $value }}
      {{- end }}
      {{- end }}
```

## Prerequisites for New Applications

For a new application to use this secret management system, the following condition must be met:

1.  **A Vault Role Must Exist:** A Vault Kubernetes Auth Role must be created that binds the application's Kubernetes Service Account (e.g., `serviceAccountName: my-app` in the `my-app` namespace) to a Vault Policy that grants read access to the desired secret path.

Unlike other methods, this approach **does not require the container to have a shell**, making it ideal for secure, minimal, distroless images.