# Documentation: Ingress Architecture

This document details the architecture for exposing services to the public internet. It covers the components involved, the final working configuration, and the rationale behind the design choices, informed by significant troubleshooting.

## 1. Core Philosophy

The goal is to expose both standard web traffic (HTTPS) and gRPC traffic securely and reliably without opening any ports on the home network's router. The entire platform is decoupled from the physical location's network limitations.

The chosen architecture uses three main components:
1.  **K3s Service Load Balancer (Klipper):** Exposes services to the node's local network.
2.  **NGINX Ingress Controller:** Acts as the central, smart router inside the Kubernetes cluster.
3.  **Cloudflare Tunnel (`cloudflared`):** Provides a secure, outbound-only connection from the cluster to the Cloudflare network, acting as the public entry point.

---

## 2. The Troubleshooting Journey: Challenges & Solutions

Exposing the gRPC `api` server alongside standard web traffic presented several significant challenges. The final architecture is a direct result of solving these issues.

### Challenge 1: Port Forwarding Failure

The initial, most traditional approach for a home lab is to use port forwarding on the router. This failed for two reasons:
*   **ISP Firewalls:** The Xfinity router's "Advanced Security" feature automatically blocked incoming traffic on port 443, treating it as a potential threat. This is a common issue with residential ISPs.
*   **Lack of Portability:** This method ties the cluster to the physical router's configuration and public IP address.

**Solution:** This approach was abandoned in favor of a more robust solution that does not require any inbound ports: Cloudflare Tunnel.

### Challenge 2: Cloudflare Tunnel gRPC Limitations

While the Cloudflare Tunnel worked immediately for standard HTTPS traffic, it failed for gRPC traffic, resulting in `522 Connection Timed Out` errors from Cloudflare. The official documentation was not up-to-date with the client's features.

**Solution:** After extensive research, a GitHub issue revealed the undocumented, correct way to proxy gRPC traffic through the tunnel:
1.  The `cloudflared` daemon on the node must be started with the `--protocol http2` flag.
2.  The `config.yml` must point to the ingress controller's `https` endpoint and use the `http2Origin: true` flag.

This combination enables the correct end-to-end HTTP/2 communication that gRPC requires, solving the `522` errors.

---

## 3. Final Architecture & Configuration

This is the definitive, working configuration for the entire platform.

### Component 1: K3s Service Load Balancer

*   **Role:** When a service is created with `type: LoadBalancer` (like the NGINX Ingress Controller's service), the built-in K3s load balancer (Klipper) automatically binds to the Raspberry Pi's local IP address (`10.0.0.200`) on the service's specified ports (e.g., 80 and 443).
*   **Configuration:** This is a built-in K3s feature. Our only responsibility is to ensure the NGINX service is set to `type: LoadBalancer`, which is the default in the official Helm chart.

### Component 2: NGINX Ingress Controller

*   **Role:** The single, central router for all traffic entering the cluster. It inspects the hostname of incoming requests and routes them to the correct internal service based on `Ingress` resource rules.
*   **Configuration:** Managed via the `charts/charts/nginx/` wrapper chart.
*   **Key `Ingress` Annotations:**
    *   `kubernetes.io/ingress.class: "nginx"`: Tells NGINX to manage this Ingress.
    *   `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"`: **Crucial for gRPC.** Tells NGINX to speak gRPC to the backend service.
*   **TLS:** Every `Ingress` resource must have a `tls` section. This tells NGINX to terminate TLS for that host using its default self-signed certificate. This is required for the `cloudflared` agent to connect via `https`.

### Component 3: Cloudflare Tunnel

*   **Role:** The secure bridge from the Cloudflare network to the NGINX Ingress Controller. It makes an outbound-only connection, so no ports need to be opened on the router.
*   **Configuration is in two parts on the Raspberry Pi:**

    **1. `systemd` Service Override:** This forces the daemon to run in the correct protocol mode.
    *   **File:** `/etc/systemd/system/cloudflared.service.d/override.conf`
    ```ini
    [Service]
    ExecStart=
    ExecStart=/usr/local/bin/cloudflared --protocol http2 tunnel run
    ```
    *(Note: The first `ExecStart=` is blank to clear the original command before replacing it.)*

    **2. `config.yml` Ingress Rules:** This tells the agent how to route incoming requests.
    *   **File:** `/etc/cloudflared/config.yml`
    ```yaml
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

      # Other rules for SSH, etc.
      - hostname: "k8s.spitikos.dev"
        service: "tcp://127.0.0.1:6443"
      - hostname: "ssh.spitikos.dev"
        service: "ssh://127.0.0.1:22"

      # Final catch-all
      - service: http_status:404
    ```

### Component 4: Cloudflare DNS

*   **Role:** To direct all traffic for `spitikos.dev` and its subdomains to the Cloudflare network, where the Tunnel can pick it up.
*   **Configuration:** All records are `CNAME`s pointing to the tunnel's unique ID.
    *   **Type:** `CNAME`
    *   **Name:** `api` (or `homepage`, `@`, etc.)
    *   **Target:** `<YOUR_TUNNEL_ID>.cfargotunnel.com`
    *   **Proxy status:** **Proxied (orange cloud)**

---

## 4. Final Traffic Flow

The final, working data path for a gRPC request is as follows:

1.  **Client** sends a gRPC request to `api.spitikos.dev`.
2.  **Cloudflare DNS** resolves to a Cloudflare IP.
3.  **Cloudflare Edge** receives the request, terminates the public TLS, and sees that it's for a Tunnel.
4.  The request is sent through the secure **Cloudflare Tunnel** using the HTTP/2 protocol.
5.  The **`cloudflared` agent** on the Pi receives the request.
6.  `cloudflared` makes a new `https` request to the **K3s Load Balancer** at `10.0.0.200:443`.
7.  The K3s Load Balancer forwards the traffic to the **NGINX Ingress pod**.
8.  **NGINX** terminates the internal TLS, sees the `Host` is `api.spitikos.dev`, and reads the `backend-protocol: "GRPC"` annotation.
9.  NGINX forwards the request as gRPC to the **`api` service**.
10. The `api` service sends the request to the **`api` pod**, which processes it.
