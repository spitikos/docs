# Secret Management with HashiCorp Vault

This document outlines the platform's strategy for managing secrets, which leverages HashiCorp Vault to dynamically inject secrets into application pods. This approach avoids the need to store sensitive information in static Kubernetes `Secret` objects, which are only base64 encoded and not truly secure.

## Mechanism

The core of our secret management is the Vault Kubernetes Injector, which is deployed cluster-wide. The injector identifies pods that are annotated for secret injection and modifies them to include a Vault Agent `init` container.

The process is as follows:

1.  An application's Helm chart is configured with specific `podAnnotations` to request secrets from Vault.
2.  When the pod starts, the `vault-agent-init` container runs first. It authenticates to Vault using the pod's Kubernetes Service Account.
3.  Upon successful authentication, the agent fetches the secrets from the path specified in the annotations.
4.  It then renders these secrets into a shell script (e.g., `config.env`) inside a shared `emptyDir` volume mounted at `/vault/secrets`. The script contains a series of `export KEY=VALUE` commands.
5.  After the `init` container completes, the main application container starts. Its entrypoint is overridden to use a shell (`/bin/sh`).
6.  The shell is commanded to `source` the `config.env` script first, which exports the secrets as environment variables. It then executes the application's main binary.

This ensures that the application starts with all its required secrets available as environment variables, without them ever being stored in the application's container image or in a Kubernetes `Secret`.

## Example Implementation

The `api` service is a reference implementation for this pattern. The following snippet from its `values.yaml` enables secret injection:

```yaml
deployment:
  # ...
  podAnnotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "api"
    vault.hashicorp.com/agent-init-first: "true"
    vault.hashicorp.com/agent-inject-secret-config.env: "secret/data/spitikos/api"
    vault.hashicorp.com/agent-inject-template-config.env: |
      {{- with secret "secret/data/spitikos/api" -}}
      {{- range $key, $value := .Data.data }}
      export {{ $key }}={{ $value }}
      {{- end }}
      {{- end }}
  
  # The command is overridden to start a shell.
  command: ["/bin/sh"]
  
  # The shell is instructed to source the secrets file before running the app.
  args: ["-c", "source /vault/secrets/config.env && /server"]
```

## Prerequisites for New Applications

For a new application to use this secret management system, two conditions must be met:

1.  **A Vault Role Must Exist:** A Vault Kubernetes Auth Role must be created that binds the application's Kubernetes Service Account (e.g., `serviceAccountName: my-app` in the `my-app` namespace) to a Vault Policy that grants read access to the desired secret path.
2.  **Container Must Include a Shell:** The application's container image must contain a shell (e.g., `/bin/sh`). Distroless or scratch images are incompatible with this injection method.

