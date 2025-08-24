# Documentation: Remote & Local Development

This document explains the different methods for interacting with the cluster from your local machine, both for direct management and for application development.

## 1. Direct Remote Management (SSH & `kubectl`)

Thanks to the Cloudflare Tunnel, you can securely manage the Raspberry Pi and the Kubernetes cluster from anywhere without a VPN.

### 1.1. Remote SSH Access

The tunnel is configured to proxy SSH traffic. To connect:

1.  **Ensure `cloudflared` is installed** on your local machine (`brew install cloudflared`).
2.  **Add the following to your `~/.ssh/config` file:**
    ```
    Host pi.spitikos.dev
      HostName ssh.spitikos.dev
      ProxyCommand /opt/homebrew/bin/cloudflared access ssh --hostname %h
    ```
    _(Note: Verify the path to your `cloudflared` executable by running `which cloudflared`)_
3.  **Connect:** You can now simply run `ssh pi.spitikos.dev`.

### 1.2. Remote `kubectl` Access via TCP Tunnel

To securely connect `kubectl` to the cluster, we use `cloudflared` to create a local TCP proxy that tunnels traffic to the Kubernetes API server.

#### Method A: Persistent Background Service (Recommended)

This is the best practice. It creates a background service that starts automatically when you log in.

1.  **Create a `launchd` agent file.** Run the following command to create the service definition:

    ```bash
    cat << EOF > ~/Library/LaunchAgents/com.cloudflare.cloudflared.k8s-proxy.plist
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.cloudflare.cloudflared.k8s-proxy</string>
        <key>ProgramArguments</key>
        <array>
            <string>/opt/homebrew/bin/cloudflared</string>
            <string>access</string>
            <string>tcp</string>
            <string>--hostname</string>
            <string>k8s.spitikos.dev</string>
            <string>--url</string>
            <string>localhost:6443</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <true/>
    </dict>
    </plist>
    EOF
    ```

2.  **Load and start the service.** This command enables the service to start on login.
    ```bash
    launchctl load -w ~/Library/LaunchAgents/com.cloudflare.cloudflared.k8s-proxy.plist
    ```

#### Method B: Manual Terminal Session (Temporary)

Use this method if you only need temporary access.

1.  **Run the proxy command** in a dedicated terminal window. You must leave this window open.
    ```bash
    cloudflared access tcp --hostname k8s.spitikos.dev --url localhost:6443
    ```

#### `kubeconfig` Setup (For Both Methods)

Your `~/.kube/config` file should use the original certificate data from the Pi, but point to your **local proxy**.

1.  **Get the original `kubeconfig`** from the Pi: `sudo cat /etc/rancher/k3s/k3s.yaml`
2.  **Copy it** to your local `~/.kube/config`.
3.  **Change only the `server` line** to point to the local proxy:
    `diff
    - server: https://127.0.0.1:6443
    + server: https://localhost:6443
    `
    Your `kubectl` commands will now work seamlessly from any network.

## 2. Local Application Development with Telepresence

When you are actively developing a service on your local machine and need it to communicate with _other services running inside the cluster_, Telepresence is the best tool.

- **Remote `kubectl`** is for managing the cluster.
- **Telepresence** is for developing applications that run against the cluster.

### Setup and Usage

1.  **Install the CLI (macOS):**

    ```bash
    brew install datawire/telepresence/telepresence
    ```

2.  **Install the Traffic Manager:**

    ```bash
    telepresence helm install
    ```

3.  **Connect to the Cluster:**

    ```bash
    sudo telepresence connect
    ```

4.  **Run your local service** (e.g., `pnpm dev`). Any code that connects to an in-cluster service (e.g., `my-api.my-namespace.svc.cluster.local`) will now succeed.

5.  **Disconnect:**
    ```bash
    telepresence quit
    ```
