# WireGuard Setup Guide

Complete instructions for setting up WireGuard VPN on a home server using wg-easy.

## Prerequisites

- Linux server (Ubuntu 22.04+ recommended)
- Docker installed
- A domain pointing to your home IP (dynamic DNS services like freemyip.com work well)
- Router access for port forwarding

## Step 1: Find Your Network Interface

```bash
ip link show
```

Look for your physical interface (e.g., `eth0`, `enp2s0f0`). Note this for the next step.

## Step 2: Create a Macvlan Network

Macvlan gives the container its own IP on your LAN, enabling proper routing for VPN clients.

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

## Step 3: Choose an Available IP

Check your router's device list or scan your network to find an unused IP:

```bash
nmap -sn 192.168.X.0/24
```

Choose an IP that's not in use (e.g., `192.168.X.210`).

## Step 4: Generate a Password Hash

For the web UI password, generate a bcrypt hash:

```bash
docker run --rm -it ghcr.io/wg-easy/wg-easy wgpw 'YOUR_PASSWORD'
```

Or use an online bcrypt generator.

## Step 5: Run wg-easy

```bash
docker run -d \
  --name wg-easy \
  --restart unless-stopped \
  --network macvlan_net \
  --ip 192.168.X.210 \
  -e WG_HOST=your.domain.com \
  -e PASSWORD_HASH='<YOUR_BCRYPT_HASH>' \
  -e WG_DEFAULT_DNS=192.168.X.2 \
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

### Environment Variables Explained

| Variable | Description |
|----------|-------------|
| `WG_HOST` | Your domain or public IP |
| `PASSWORD_HASH` | Bcrypt hash for web UI login |
| `WG_DEFAULT_DNS` | DNS server for VPN clients (use Pi-hole IP for ad blocking) |
| `WG_ALLOWED_IPS` | Networks accessible through VPN |
| `WG_DEFAULT_ADDRESS` | VPN client address range |
| `WG_MTU` | Maximum transmission unit (1280 recommended for compatibility) |

## Step 6: Configure Router Port Forward

Create a port forwarding rule:

| Field | Value |
|-------|-------|
| Name | WireGuard |
| Protocol | **UDP** (not TCP!) |
| External Port | 51820 |
| Internal IP | 192.168.X.210 |
| Internal Port | 51820 |

**Important**: WireGuard uses UDP. A TCP forward will not work.

## Step 7: Fix Macvlan Host Communication

Containers on macvlan cannot communicate with their host by default. Create a shim interface:

```bash
sudo ip link add macvlan-shim link <YOUR_INTERFACE> type macvlan mode bridge
sudo ip addr add 192.168.X.211/32 dev macvlan-shim
sudo ip link set macvlan-shim up
sudo ip route add 192.168.X.210/32 dev macvlan-shim
```

### Make It Persistent

Create the file `/etc/networkd-dispatcher/routable.d/50-macvlan-shim`:

```bash
sudo tee /etc/networkd-dispatcher/routable.d/50-macvlan-shim << 'EOF'
#!/bin/bash
ip link add macvlan-shim link <YOUR_INTERFACE> type macvlan mode bridge
ip addr add 192.168.X.211/32 dev macvlan-shim
ip link set macvlan-shim up
ip route add 192.168.X.210/32 dev macvlan-shim
EOF

sudo chmod +x /etc/networkd-dispatcher/routable.d/50-macvlan-shim
```

## Step 8: Create Clients

1. Access the web UI: `http://192.168.X.210:51821`
2. Log in with your password
3. Click "+ New Client"
4. Name it (e.g., "iPhone", "MacBook")
5. Download config or scan QR code

### Client Configuration Tips

For maximum compatibility, ensure the client config has:

```ini
[Interface]
PrivateKey = <generated>
Address = 10.8.0.X/24
DNS = 192.168.X.2
MTU = 1280

[Peer]
PublicKey = <generated>
PresharedKey = <generated>
AllowedIPs = 192.168.X.0/24, 10.8.0.0/24
Endpoint = your.domain.com:51820
```

**Note**: If connecting from IPv6-only networks (like cellular), you may need to hardcode the IPv4 address in the Endpoint field.

## Step 9: Test the Connection

1. Connect to VPN from a different network (not your home WiFi)
2. Check for handshake in client app
3. Try to ping your server: `ping 192.168.X.181`
4. Access a service: `http://192.168.X.181:8920`

## Verification Commands

```bash
# Check container is running
docker ps | grep wg-easy

# View container IP
docker inspect wg-easy | grep -i ipaddress

# Check WireGuard status
docker exec wg-easy wg show

# View logs
docker logs wg-easy
```

## Next Steps

- See [Troubleshooting](troubleshooting.md) if you encounter issues
- See [Dual VPN Strategy](dual-vpn.md) for using WireGuard alongside Tailscale
