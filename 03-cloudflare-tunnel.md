# Documentation: Cloudflare Tunnel

This document explains the final, working configuration for `cloudflared`, which creates a secure, outbound-only connection from the cluster to the Cloudflare network. This exposes services to the internet without opening any firewall ports.

This configuration is the result of extensive troubleshooting and is the single source of truth for proxying both standard HTTPS and gRPC traffic.

## 1. DNS Configuration

All public-facing services are defined as `CNAME` records in Cloudflare DNS, pointing to the tunnel's unique ID. This is the most robust method.

- **Type:** `CNAME`
- **Name:** `api` (or `homepage`, `@`, etc.)
- **Target:** `<YOUR_TUNNEL_ID>.cfargotunnel.com`
- **Proxy status:** **Proxied (orange cloud)**

## 2. `cloudflared` Service Configuration

The `cloudflared` daemon on the Raspberry Pi requires two pieces of configuration to handle gRPC correctly.

### 2.1. `systemd` Service Override

The daemon must be started with the `--protocol http2` flag. This is a global setting that enables the correct transport protocol for the entire tunnel.

1.  **Create an override directory:**
    ```bash
    sudo mkdir -p /etc/systemd/system/cloudflared.service.d/
    ```
2.  **Create the override file:**
    *   **File:** `/etc/systemd/system/cloudflared.service.d/override.conf`
    ```ini
    [Service]
    ExecStart=
    ExecStart=/usr/local/bin/cloudflared --protocol http2 tunnel run
    ```
    *(Note: The first `ExecStart=` is blank to clear the original command before replacing it.)*
3.  **Reload the systemd daemon** after creating or changing this file:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart cloudflared
    ```

### 2.2. `config.yml` Ingress Rules

This file tells the `cloudflared` agent how to route traffic. The key is to use a consistent `https` service type with the `http2Origin` flag for all web and gRPC traffic.

*   **File:** `/etc/cloudflared/config.yml`
```yaml
tunnel: <YOUR_TUNNEL_UUID>
credentials-file: <YOUR_CREDENTIALS_FILE_PATH>

ingress:
  # This rule handles gRPC traffic for the API server.
  - hostname: api.spitikos.dev
    service: https://10.0.0.200:443
    originRequest:
      noTLSVerify: true
      http2Origin: true

  # This is a consistent catch-all for all other HTTPS traffic.
  - hostname: "*.spitikos.dev"
    service: https://10.0.0.200:443
    originRequest:
      noTLSVerify: true
      http2Origin: true

  # Other rules for non-HTTP services.
  - hostname: "k8s.spitikos.dev"
    service: "tcp://127.0.0.1:6443"
  - hostname: "ssh.spitikos.dev"
    service: "ssh://127.0.0.1:22"

  # Final catch-all
  - service: http_status:404
```
- **`service: https://10.0.0.200:443`**: All traffic is pointed to the NGINX LoadBalancer service on the Pi's local IP at the standard HTTPS port.
- **`noTLSVerify: true`**: This is required because the NGINX Ingress Controller will present its default, self-signed "Fake Certificate," which is not publicly trusted.
- **`http2Origin: true`**: This flag enables the HTTP/2 protocol for the connection from `cloudflared` to NGINX, which is necessary for gRPC to work.