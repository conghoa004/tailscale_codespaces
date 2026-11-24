# Tailscale in GitHub Codespaces & Dev Containers

[![Tailscale](https://img.shields.io/badge/Tailscale-Connected-blue?logo=tailscale&logoColor=white)](https://tailscale.com)
[![GitHub Codespaces](https://img.shields.io/badge/GitHub-Codespaces-181717?logo=github&logoColor=white)](https://github.com/features/codespaces)
[![Dev Containers](https://img.shields.io/badge/Dev_Containers-Ready-blueviolet?logo=visualstudiocode&logoColor=white)](https://containers.dev)

A ready-to-use development container configuration that connects your **GitHub Codespaces** (or local **VS Code Dev Containers**) directly into your private **Tailscale Tailnet**.

---

## Table of Contents

- [Overview](#overview)
- [Key Features & Benefits](#key-features--benefits)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
  - [Method 1: Automatic Authentication (Recommended)](#method-1-automatic-authentication-recommended)
  - [Method 2: Manual Interactive Authentication](#method-2-manual-interactive-authentication)
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

This repository uses the official [Tailscale Dev Container Feature](https://github.com/tailscale/codespace) (`ghcr.io/tailscale/codespace/tailscale`) to make your Codespace an authorized node inside your encrypted Tailscale network (Tailnet).

---

## Key Features & Benefits

- **Zero-trust Private Networking**: Secure WireGuard-based mesh networking directly inside the Codespace container.
- **Bi-directional Access**:
  - Outbound: Access your home lab, internal VPCs, private databases, license servers, and internal Git servers directly from Codespaces.
  - Inbound: Connect to servers or processes running inside Codespaces from your laptop, mobile phone, or desktop using private MagicDNS or 100.x.y.z Tailscale IPs.
- **Zero Configuration Port Tunnels**: No need to rely on public URLs or GitHub port forward permissions.
- **Ephemeral Node Support**: Automatically clean up inactive Codespaces from your Tailnet when deleted.
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

## Quick Start

### Method 1: Automatic Authentication (Recommended)

Automatic authentication allows your Codespace to automatically join your Tailnet immediately upon boot without any manual login prompt.

#### Step 1: Generate an Auth Key in Tailscale
1. Log in to the [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys).
2. Click **Generate auth key...**.
3. Configure the key settings:
   - **Reusable**: Optional (Enable if you plan to launch multiple Codespaces with the same key).
   - **Ephemeral**: **Recommended (Yes)** — Codespaces are temporary environments. Ephemeral keys ensure the node is automatically removed from your Tailnet when the Codespace is shut down or deleted.
   - **Tags**: (Optional) Assign ACL tags (e.g., `tag:codespace`, `tag:dev`) to apply specific access rules.
4. Click **Generate key** and copy the generated key (`tskey-auth-...`).

#### Step 2: Add Secret to GitHub
You can add the secret globally for all your Codespaces or specifically for this repository:

1. In GitHub, go to your repository **Settings** (or user **Settings** > **Codespaces** for global use).
2. Navigate to **Secrets** > **Codespace user secrets**.
3. Click or **New secret**.
4. Name: `TS_AUTH_KEY`
5. Value: Paste your Tailscale auth key (`tskey-auth-...`).
6. Save the secret.

#### Step 3: Launch Codespace
1. Click the green **Code** button on GitHub.
2. Select the **Codespaces** tab.
3. Click **Create codespace on main**.
4. Once the terminal opens, your Codespace will automatically authenticate and join your Tailnet!

---

### Method 2: Manual Interactive Authentication

If you do not want to configure a GitHub secret, you can log in manually:

1. Create and open your Codespace normally.
2. Open the integrated terminal inside VS Code.
3. Run the following command:
   ```bash
   sudo tailscale up --accept-routes
   ```
4. A login URL will appear in the terminal:
   ```text
   To authenticate, visit:
   https://login.tailscale.com/a/xxxxxx
   ```
5. Click the URL or copy-paste it into your browser and approve the device.
6. The terminal will confirm success: `Success.`

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
6. Inside the container terminal, authenticate via `sudo tailscale up` or provide `TS_AUTH_KEY` in your local environment.

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
- **State Persistence**: The feature automatically creates a dedicated Docker volume (`tailscale-${devcontainerId}`) mapped to `/var/lib/tailscale`, so your login state persists across Codespace restarts and rebuilds.

---

## Common Use Cases

### 1. Accessing Private Databases & Internal APIs
Once connected, any private resource in your Tailnet is accessible using its MagicDNS name or 100.x.y.z IP:

```bash
# Connect to a PostgreSQL database hosted in your home lab or private cloud
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

### 2. Old/Offline Codespaces cluttering the Tailscale Machines list
Generate your auth key with the **Ephemeral** option enabled. With ephemeral keys, Tailscale will automatically prune the node from your machine list shortly after the Codespace terminates.

### 3. Codespace did not connect automatically
- Verify the secret name in GitHub is exactly `TS_AUTH_KEY` (all caps, with underscores).
- Check if the secret was granted access to this repository under *Settings* > *Secrets and variables* > *Codespaces*.
- Check the startup logs in Codespace:
  ```bash
  sudo tailscale status
  ```

---

## License

This project is licensed under the [MIT License](LICENSE) (or your preferred repository license).
