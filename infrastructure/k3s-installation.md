# Documentation: k3s Installation

This document details the installation of the k3s Kubernetes distribution and the configuration of `kubectl` for remote management.

## 1. k3s Installation on the Raspberry Pi

We install k3s using the official installation script, with a key modification to disable the default bundled Traefik ingress controller. This allows us to install and manage Traefik ourselves via Helm for greater control.

### Installation Command

Run the following command on the Raspberry Pi:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -s - --write-kubeconfig-mode 644
```

- `INSTALL_K3S_EXEC="--disable=traefik"`: Prevents the default Traefik from being installed.
- `--write-kubeconfig-mode 644`: Makes the generated kubeconfig file readable by any user, simplifying the process of copying it to your local machine.

### Verification

After the installation, verify that the k3s service is running correctly:

```bash
sudo systemctl status k3s # Or k3s-server.service
```

The output should show the service as `active (running)`.

### Critical: Verify Cluster Networking

Before deploying any applications, it is critical to verify that the cluster's internal networking and DNS are healthy. Run the following command:

```bash
kubectl run --rm -it --image=busybox test-dns -- sh -c "nslookup kubernetes.default"
```

This command must succeed. If it fails with a timeout or a `serviceaccount not found` error, the K3s installation is not healthy. You must perform a "deep clean" (uninstall, delete `/etc/rancher` and `/var/lib/rancher`, and reboot) before re-installing.

## 2. Configuring Remote `kubectl` Access

To manage the cluster from your local development machine, you need to copy the cluster's configuration file.

### Steps

1.  **Retrieve the kubeconfig:** On the Raspberry Pi, display the contents of the configuration file:

    ```bash
    sudo cat /etc/rancher/k3s/k3s.yaml
    ```

2.  **Configure your local machine:**
    - Copy the entire YAML output from the command above.
    - Open the `~/.kube/config` file on your local machine.
    - Paste the copied configuration into the file. If you have other cluster configurations, merge the new configuration into the `clusters`, `contexts`, and `users` lists.
    - **Crucially, find the `server` line and change the IP address from `127.0.0.1` to the Pi's static IP:**
      ```diff
      - server: https://127.0.0.1:6443
      + server: https://10.0.0.200:6443
      ```

3.  **Verify the connection:** From your local machine, run:
    ```bash
    kubectl get nodes
    ```
    You should see your Raspberry Pi node listed with a `Ready` status. Your cluster is now ready for remote management.
