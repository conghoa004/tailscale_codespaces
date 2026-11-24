# Tailscale in GitHub Codespaces & Dev Containers

[![Tailscale](https://img.shields.io/badge/Tailscale-Connected-blue?logo=tailscale&logoColor=white)](https://tailscale.com)
[![GitHub Codespaces](https://img.shields.io/badge/GitHub-Codespaces-181717?logo=github&logoColor=white)](https://github.com/features/codespaces)
[![Dev Containers](https://img.shields.io/badge/Dev_Containers-Ready-blueviolet?logo=visualstudiocode&logoColor=white)](https://containers.dev)

A ready-to-use development container configuration that connects your **GitHub Codespaces** (or local **VS Code Dev Containers**) directly into your private **Tailscale Tailnet** via interactive login.

---

## Table of Contents

- [Overview](#overview)
- [Key Features & Benefits](#key-features--benefits)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick Start (Interactive Login)](#quick-start-interactive-login)
  - [Step 1: Launch the Codespace](#step-1-launch-the-codespace)
  - [Step 2: Authenticate with Tailscale](#step-2-authenticate-with-tailscale)
  - [Step 3: Verify Connection](#step-3-verify-connection)
- [Running Locally with Dev Containers](#running-locally-with-dev-containers)
- [Configuration Breakdown](#configuration-breakdown)
- [Common Use Cases](#common-use-cases)
  - [1. Accessing Private Databases & Internal APIs](#1-accessing-private-databases--internal-apis)
  - [2. Accessing Codespace Services from Local Devices](#2-accessing-codespace-services-from-local-devices)
  - [3. Enabling Tailscale SSH](#3-enabling-tailscale-ssh)
  - [4. Exposing Services via Tailscale Serve / Funnel](#4-exposing-services-via-tailscale-serve--funnel)
- [Useful Tailscale Commands](#useful-tailscale-commands)
- [Troubleshooting & FAQ](#troubleshooting--faq)
- [License](#license)

---

## Overview

By default, GitHub Codespaces run inside isolated cloud virtual machines. While GitHub provides built-in port forwarding, connecting to private networks, internal databases, staging environments, or homelab hardware usually requires complicated tunnels or exposing ports to the public internet.

This repository uses the official [Tailscale Dev Container Feature](https://github.com/tailscale/codespace) (`ghcr.io/tailscale/codespace/tailscale`) to make your Codespace an authorized node inside your encrypted Tailscale network (Tailnet) without needing any pre-configured GitHub secrets.

---

## Key Features & Benefits

- **Zero-trust Private Networking**: Secure WireGuard-based mesh networking directly inside the Codespace container.
- **Bi-directional Access**:
  - Outbound: Access your homelab, internal VPCs, private databases, license servers, and internal Git servers directly from Codespaces.
  - Inbound: Connect to servers or processes running inside Codespaces from your laptop, mobile phone, or desktop using private MagicDNS or 100.x.y.z Tailscale IPs.
- **No Secret Setup Required**: Easily authenticate on-demand via the browser link generated directly in your terminal.
- **Zero Configuration Port Tunnels**: No need to rely on public URLs or GitHub port forward permissions.
- **Tailscale SSH Compatible**: SSH into your Codespace terminal securely from any authorized device.

---

## Project Structure

```text
.
├── .devcontainer/
│   └── devcontainer.json   # Dev Container definition with Tailscale feature & TUN device
└── README.md               # Documentation and setup guide
```

---

## Prerequisites

1. A [GitHub](https://github.com) account with access to GitHub Codespaces.
2. A [Tailscale](https://tailscale.com) account (free for personal use).
3. (Optional) [Docker Desktop](https://www.docker.com/) and [VS Code](https://code.visualstudio.com/) with the **Dev Containers** extension if running locally.

---

## Quick Start (Interactive Login)

You can connect your Codespace to your Tailnet interactively in a few simple steps without creating or managing API keys.

### Step 1: Launch the Codespace
1. In your GitHub repository, click the green **Code** button.
2. Select the **Codespaces** tab.
3. Click **Create codespace on main**.
4. Wait for the container to build and the VS Code editor to open.

### Step 2: Authenticate with Tailscale
1. Open the integrated terminal in VS Code (`Ctrl+` `~` or via the application menu).
2. Run the following command:
   ```bash
   sudo tailscale up --accept-routes
   ```
   > **Tip:** If you also want to be able to SSH into this Codespace from your other Tailscale devices, add `--ssh`:
   > ```bash
   > sudo tailscale up --accept-routes --ssh
   > ```
3. Tailscale will output an authentication link:
   ```text
   To authenticate, visit:

       https://login.tailscale.com/a/xxxxxx
   ```
4. Click the link (or copy-paste it into your web browser) and click **Connect** / log in to approve the device.
5. In the terminal, you will see confirmation:
   ```text
   Success.
   ```

### Step 3: Verify Connection
Check that your Codespace is connected and see your assigned Tailscale IP:
```bash
# Check status and see connected Tailnet peers
tailscale status

# Check the IPv4 address of this Codespace
tailscale ip -4
```

---

## Running Locally with Dev Containers

You can also run this setup locally on your computer with VS Code:

1. Install [Docker Desktop](https://www.docker.com/) and [Visual Studio Code](https://code.visualstudio.com/).
2. Install the **Dev Containers** extension in VS Code.
3. Clone this repository locally:
   ```bash
   git clone https://github.com/<your-username>/tailscale_codespaces.git
   cd tailscale_codespaces
   ```
4. Open the folder in VS Code:
   ```bash
   code .
   ```
5. Press `F1` (or `Ctrl+Shift+P` / `Cmd+Shift+P`) and choose:
   ```text
   Dev Containers: Reopen in Container
   ```
6. Inside the container terminal, log in interactively:
   ```bash
   sudo tailscale up --accept-routes
   ```

---

## Configuration Breakdown

The configuration resides in [`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json):

```json
{
  "name": "Tailscale Codespaces",
  "runArgs": [
    "--device=/dev/net/tun"
  ],
  "features": {
    "ghcr.io/tailscale/codespace/tailscale": {
      "version": "latest"
    }
  }
}
```

### Key Components:
- **`runArgs: ["--device=/dev/net/tun"]`**: Tailscale creates a virtual TUN network device interface. Exposing `/dev/net/tun` to the container allows kernel-level Tailscale networking.
- **`features.ghcr.io/tailscale/codespace/tailscale`**: The official Tailscale Dev Container feature. It installs the `tailscale` CLI and `tailscaled` daemon, mounts state volumes (`/var/lib/tailscale`), and sets up background network daemons.
- **State Persistence**: The feature automatically creates a dedicated Docker volume (`tailscale-${devcontainerId}`) mapped to `/var/lib/tailscale`, so your login session persists across Codespace restarts and rebuilds.

---

## Common Use Cases

### 1. Accessing Private Databases & Internal APIs
Once connected, any private resource in your Tailnet is accessible using its MagicDNS name or 100.x.y.z IP:

```bash
# Connect to a PostgreSQL database hosted in your homelab or private cloud
psql -h homelab-postgres -U postgres -d mydb

# Query an internal staging API
curl http://staging-api.internal-domain.ts.net:8080/health
```

### 2. Accessing Codespace Services from Local Devices
Get your Codespace's Tailscale IP:
```bash
tailscale ip -4
```
Start any dev server inside Codespaces:
```bash
python3 -m http.server 3000
```
From your home laptop, open your browser and navigate directly to:
```text
http://<codespace-tailscale-ip>:3000
# or using its MagicDNS name:
http://codespaces-xxxxx:3000
```
No port forwarding configuration or public exposures required.

### 3. Enabling Tailscale SSH
Manage your Codespace using native SSH without configuring authorized keys:
```bash
sudo tailscale up --ssh --accept-routes
```
From your laptop (or another machine on your Tailnet):
```bash
ssh <codespace-name>
# or
ssh <codespace-tailscale-ip>
```

### 4. Exposing Services via Tailscale Serve / Funnel
- **Tailscale Serve**: Share a web server privately across your Tailnet with automated HTTPS:
  ```bash
  tailscale serve --bg 3000
  ```
- **Tailscale Funnel**: Securely route public internet traffic to a port in your Codespace (if enabled in your Tailnet policy):
  ```bash
  tailscale funnel 3000
  ```

---

## Useful Tailscale Commands

| Command | Description |
| :--- | :--- |
| `tailscale status` | Check connection status and view peers on the Tailnet. |
| `tailscale ip -4` | Show the IPv4 address assigned to this Codespace. |
| `tailscale ping <peer>` | Ping another node on your Tailnet over WireGuard. |
| `sudo tailscale up` | Connect / update Tailscale flags (e.g. `--ssh`, `--accept-routes`). |
| `sudo tailscale down` | Disconnect the Codespace from your Tailnet. |
| `tailscale logout` | Log out and disconnect the node from your Tailscale account. |
| `tailscale netcheck` | Perform a diagnostic report on network and NAT traversal. |
| `tailscale serve status` | Show active Tailscale Serve proxy configurations. |

---

## Troubleshooting & FAQ

### 1. "Cannot open /dev/net/tun: No such file or directory"
Ensure that `runArgs: ["--device=/dev/net/tun"]` is present in your `devcontainer.json`. If running on custom Docker or local Linux environments, you might also need capabilities:
```json
"capAdd": [
  "CAP_NET_ADMIN",
  "CAP_MKNOD",
  "CAP_NET_RAW"
]
```

### 2. "tailscaled is not running" or permission errors
Ensure you run `tailscale up` with `sudo`:
```bash
sudo tailscale up --accept-routes
```
You can also check daemon status:
```bash
sudo tailscale status
```

### 3. Cleaning up old Codespaces from Tailnet
When you delete or stop using a Codespace, you can disconnect it before deleting:
```bash
sudo tailscale logout
```
Alternatively, remove the machine anytime from the [Tailscale Admin Console](https://login.tailscale.com/admin/machines).

---

## License

This project is licensed under the [MIT License](LICENSE) (or your preferred repository license).
