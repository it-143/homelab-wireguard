# Tailscale Setup Guide

Tailscale is a mesh VPN built on WireGuard that handles NAT traversal automatically. It's perfect for networks where traditional WireGuard struggles, like cellular networks with NAT64.

## Why Tailscale?

Traditional WireGuard requires:
- Port forwarding on your router
- A public IP or dynamic DNS
- Manual configuration of each client

Tailscale eliminates all of this by using NAT traversal techniques and relay servers when direct connections aren't possible.

### When to Use Tailscale vs WireGuard

| Situation | Recommendation |
|-----------|----------------|
| Cellular data (especially NAT64) | Tailscale |
| IPv6-only networks | Tailscale |
| Regular WiFi networks | Either works |
| Fully self-hosted requirement | WireGuard |
| Quick setup needed | Tailscale |

## Installation

### Server Setup
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### Client Setup

Install Tailscale on your devices:
- **iOS/Android**: Download from App Store / Play Store
- **macOS/Windows/Linux**: Download from [tailscale.com/download](https://tailscale.com/download)

Sign in with the same account on all devices.

## Subnet Router (Access Your LAN)

By default, Tailscale only lets devices talk to each other. To access your entire home network:

### Step 1: Enable IP Forwarding
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### Step 2: Advertise Your Subnet
```bash
sudo tailscale set --advertise-routes=192.168.X.0/24
```

### Step 3: Approve the Route

1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/machines)
2. Find your server → "..." menu → "Edit route settings"
3. Check the box next to your subnet
4. Click Save

## Exit Node (Route All Traffic)

To route ALL internet traffic through your home network (for Pi-hole ad blocking everywhere):

### Step 1: Advertise as Exit Node
```bash
sudo tailscale set --advertise-exit-node --advertise-routes=192.168.X.0/24
```

### Step 2: Approve in Admin Console

Edit route settings → Check "Use as exit node" → Save

### Step 3: Enable on Client

**iOS/Android**: Tailscale app → Exit Node → Select your server

**Desktop**: `tailscale set --exit-node=<server-name>`

## Disable Key Expiry

For servers, disable key expiry so it doesn't stop working:

1. [Tailscale Admin Console](https://login.tailscale.com/admin/machines)
2. Find your server → "..." menu → "Disable key expiry"

## Useful Commands
```bash
tailscale status          # Check connection status
tailscale ip              # View your Tailscale IP
tailscale ping <device>   # Test connectivity
sudo systemctl restart tailscaled  # Restart
```

## Resources

- [Tailscale Documentation](https://tailscale.com/kb/)
- [Subnet Routers Guide](https://tailscale.com/kb/1019/subnets/)
- [Exit Nodes Guide](https://tailscale.com/kb/1103/exit-nodes/)
