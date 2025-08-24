# Documentation: Initial Raspberry Pi Setup

This document covers the initial network configuration for the Raspberry Pi to ensure it has a stable presence on the local network.

## 1. Static IP Configuration

A static IP is crucial for providing a reliable endpoint for `kubectl` and SSH access. We use `netplan`, the standard network configuration tool for modern Ubuntu systems.

### Configuration Steps

1.  **Create the Netplan file:** On the Raspberry Pi, create or edit the file `/etc/netplan/50-cloud-init.yaml`.

2.  **Add the following content:** This configuration assigns a static IP address, defines the network gateway (your router), and sets the DNS servers.

    ```yaml
    network:
      version: 2
      renderer: networkd
      wifis:
        wlan0:
          dhcp4: no
          addresses:
            - 10.0.0.200/24
          routes:
            - to: default
              via: 10.0.0.1
          nameservers:
            # Using Cloudflare's DNS Resolvers for speed and privacy
            addresses: [1.1.1.1, 1.0.0.1]
          access-points:
            "YOUR_WIFI_SSID":
              auth:
                key-management: "psk"
                password: "YOUR_WIFI_PASSWORD"
    ```

    - **Note:** Replace `YOUR_WIFI_SSID` and `YOUR_WIFI_PASSWORD` with your actual Wi-Fi credentials.

3.  **Apply the configuration:** Run the following command on the Pi. This will cause the network interface to restart. If you are connected via SSH, the session will disconnect, and you will need to reconnect to the new static IP address (`10.0.0.200`).

    ```bash
    sudo netplan apply
    ```

With this configuration, the Raspberry Pi is now ready for the k3s installation.
