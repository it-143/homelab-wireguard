# WireGuard Setup Guide

Complete instructions for setting up WireGuard VPN on a home server using wg-easy.

## Overview

This setup uses [wg-easy](https://github.com/wg-easy/wg-easy) - a Docker container that provides WireGuard with a web UI for easy client management.

## Prerequisites

- Linux server (Ubuntu/Debian recommended)
- Docker installed
- A domain pointing to your home IP (dynamic DNS works)
- Router access for port forwarding

## Installation

### Step 1: Create a macvlan network

Using macvlan gives the container its own IP on your LAN, which allows proper routing for VPN clients to reach other devices on your network.

First, find your network interface:

```bash
ip link show
```

Look for your primary interface (e.g., `eth0`, `enp2s0f0`, `ens18`).

Create the network:

```bash
docker network create -d macvlan \
  --subnet=192.168.X.0/24 \
  --gateway=192.168.X.1 \
  -o parent=<YOUR_INTERFACE> \
  macvlan_net
```

Replace:
- `192.168.X.0/24` with your network subnet
- `192.168.X.1` with your router IP
- `<YOUR_INTERFACE>` with your interface name

### Step 2: Generate a password hash

For the web UI password, generate a bcrypt hash:

```bash
# Using htpasswd
htpasswd -nbBC 12 "" "your-password" | tr -d ':\n'

# Or use https://bcrypt-generator.com/
```

### Step 3: Run wg-easy

Choose an unused IP on your network for the container (check your router's device list first).

```bash
docker run -d \
  --name wg-easy \
  --restart unless-stopped \
  --network macvlan_net \
  --ip 192.168.X.210 \
  -e WG_HOST=your.domain.com \
  -e PASSWORD_HASH='<YOUR_BCRYPT_HASH>' \
  -e WG_DEFAULT_DNS=192.168.X.1 \
  -e WG_ALLOWED_IPS=192.168.X.0/24,10.8.0.0/24 \
  -e WG_DEFAULT_ADDRESS=10.8.0.x \
  -e WG_MTU=1280 \
  -v ~/wireguard:/etc/wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
  --sysctl="net.ipv4.ip_forward=1" \
  ghcr.io/wg-easy/wg-easy
```

**Configuration options:**
- `WG_HOST`: Your domain or public IP
- `WG_DEFAULT_DNS`: DNS server for clients (your Pi-hole, router, or 1.1.1.1)
- `WG_ALLOWED_IPS`: Networks accessible through VPN
- `WG_MTU`: 1280 recommended (see troubleshooting for why)

### Step 4: Configure port forwarding

In your router's admin panel, create a port forward:

| Protocol | External Port | Internal IP | Internal Port |
|----------|---------------|-------------|---------------|
| **UDP** | 51820 | 192.168.X.210 | 51820 |

> ⚠️ **Important**: WireGuard uses **UDP**, not TCP. This is a common mistake.

### Step 5: Fix macvlan host communication

Containers on macvlan cannot communicate with their host by default. This means VPN clients can't reach services running directly on your server (not in containers).

Create a shim interface:

```bash
sudo ip link add macvlan-shim link <YOUR_INTERFACE> type macvlan mode bridge
sudo ip addr add 192.168.X.211/32 dev macvlan-shim
sudo ip link set macvlan-shim up
sudo ip route add 192.168.X.210/32 dev macvlan-shim
```

Make it persistent by copying the script from `scripts/50-macvlan-shim` to `/etc/networkd-dispatcher/routable.d/` and making it executable.

### Step 6: Verify installation

```bash
# Check container is running
docker ps | grep wg-easy

# Check container has correct IP
docker inspect wg-easy | grep -i ipaddress

# Test web UI (from local network)
curl http://192.168.X.210:51821
```

## Client Setup

### Web UI

Access the management interface at `http://192.168.X.210:51821` (local network only).

### Adding clients

1. Log in to the web UI
2. Click "+ New Client"
3. Enter a name (e.g., "iPhone", "MacBook", "Work-Laptop")
4. Download config or scan QR code

### Mobile (iOS/Android)

1. Install WireGuard app from App Store / Play Store
2. In web UI, click QR code icon next to your client
3. In app: Add Tunnel → Scan QR Code
4. Toggle VPN on

### Desktop (macOS/Windows/Linux)

1. Install WireGuard from [wireguard.com/install](https://www.wireguard.com/install/)
2. In web UI, download the .conf file
3. Import into WireGuard app
4. Activate tunnel

## Testing

With VPN connected from outside your network:

```bash
# Test connectivity to router
ping 192.168.X.1

# Test connectivity to server
ping 192.168.X.Y

# Test DNS (if using Pi-hole)
nslookup google.com 192.168.X.Z

# Test service access
curl http://192.168.X.Y:8096  # Jellyfin, etc.
```

## Updating

```bash
docker pull ghcr.io/wg-easy/wg-easy
docker stop wg-easy
docker rm wg-easy
# Re-run the docker run command from Step 3
```

Your configuration persists in `~/wireguard`.

## Backup

Critical files to backup:

```
~/wireguard/
├── wg0.conf          # Server config and client list
├── wg0.json          # Client metadata
└── *.conf            # Individual client configs
```

**If you lose these, all clients need to be reconfigured.**

## Next Steps

- See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues
- Consider setting up [Tailscale](https://tailscale.com/) for difficult networks
- Add monitoring with `docker logs -f wg-easy`
