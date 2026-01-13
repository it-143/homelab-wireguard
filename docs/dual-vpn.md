# Dual VPN Strategy

Why and how to run both WireGuard and Tailscale for maximum reliability.

## The Problem

Self-hosted WireGuard is great, but it has limitations:

- **NAT64 cellular networks** - Many carriers run IPv6-only networks that don't reliably translate WireGuard's UDP traffic
- **Strict firewalls** - Some networks block UDP entirely or filter non-standard ports
- **Double NAT** - Complex network setups can prevent direct connections
- **Dynamic IP** - If your home IP changes and DNS hasn't updated, you're locked out

## The Solution

Run both VPN solutions, each optimized for different scenarios:

| VPN | Best For | Trade-off |
|-----|----------|-----------|
| **WireGuard** | Regular WiFi, full self-hosted | Requires port forwarding, manual config |
| **Tailscale** | Cellular, difficult networks | Uses Tailscale's infrastructure |

## Why Not Just Tailscale?

Tailscale is excellent, but:

- Relies on Tailscale's coordination servers
- Not fully self-hosted (though you can use Headscale)
- Free tier has device limits
- May not satisfy "self-host everything" philosophy

## Why Not Just WireGuard?

WireGuard alone struggles with:

- IPv6-only/NAT64 cellular networks
- Networks that block UDP port 51820
- Situations where you need to "just connect"

## Setup

### WireGuard (Already Configured)

Your existing wg-easy setup handles:
- Home WiFi access
- Coffee shop/hotel WiFi
- Office networks
- Most standard network configurations

### Tailscale Setup

Install on your server:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Authenticate via the URL provided.

Install on clients (phone, laptop):
- Download Tailscale app
- Sign in with same account
- Enable connection

### Usage Pattern

```
┌─────────────────────────────────────────┐
│           Trying to connect?            │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│      On cellular / 5G / LTE?            │
└─────────────────┬───────────────────────┘
                  │
         ┌───────┴───────┐
         │               │
        Yes              No
         │               │
         ▼               ▼
┌─────────────┐  ┌─────────────────────┐
│  Tailscale  │  │  Try WireGuard      │
└─────────────┘  └──────────┬──────────┘
                           │
                  ┌────────┴────────┐
                  │                 │
              Works?            Fails?
                  │                 │
                  ▼                 ▼
            ┌──────────┐    ┌─────────────┐
            │   Done   │    │  Tailscale  │
            └──────────┘    └─────────────┘
```

## Accessing Services

Both VPNs give you access to your home network, just via different IPs:

| Service | Via WireGuard | Via Tailscale |
|---------|---------------|---------------|
| Jellyfin | `http://192.168.X.181:8920` | `http://100.X.X.X:8920` |
| Pi-hole | `http://192.168.X.2/admin` | N/A (unless Pi-hole has Tailscale) |
| SSH | `ssh user@192.168.X.181` | `ssh user@100.X.X.X` |

### Adding Pi-hole to Tailscale

To get Pi-hole ad-blocking on cellular via Tailscale:

1. Install Tailscale on your Raspberry Pi (or Pi-hole host)
2. In Tailscale admin console, set the Pi as your DNS server
3. Enable MagicDNS if desired

## Client Configuration Tips

### iPhone/iOS

Keep both apps installed:
- **Tailscale**: Enable when on cellular
- **WireGuard**: Enable when on WiFi

iOS allows only one VPN active at a time, so they won't conflict.

### MacBook/Desktop

Both can be installed. Tailscale runs as a system service; WireGuard activates on-demand.

Typical workflow:
1. Try WireGuard first
2. If no handshake after 10 seconds, switch to Tailscale

### Automation Ideas

On macOS, you could create a script that:
1. Checks network type
2. Automatically activates appropriate VPN

```bash
#!/bin/bash
# Example concept - customize for your setup

NETWORK_TYPE=$(networksetup -getairportnetwork en0 | grep -i "cellular\|lte\|5g")

if [ -n "$NETWORK_TYPE" ]; then
    # On cellular - use Tailscale
    sudo tailscale up
else
    # On WiFi - try WireGuard
    # (manual activation via WireGuard app)
    echo "Use WireGuard"
fi
```

## Comparison Table

| Feature | WireGuard (self-hosted) | Tailscale |
|---------|------------------------|-----------|
| Self-hosted | ✅ Fully | ⚠️ Coordination servers external |
| NAT traversal | ⚠️ Requires port forward | ✅ Automatic |
| NAT64/IPv6-only | ❌ Problematic | ✅ Works |
| Speed | ✅ Direct connection | ✅ Direct when possible, relayed when not |
| Setup complexity | ⚠️ Moderate | ✅ Easy |
| Cost | ✅ Free | ✅ Free tier (100 devices) |
| Offline capable | ✅ Yes | ⚠️ Needs internet for coordination |

## Best of Both Worlds

By running both:

- ✅ Always have a working connection option
- ✅ Maintain self-hosted principles where possible
- ✅ Reliable access from any network type
- ✅ Fallback when one solution fails
- ✅ Learn both technologies

The goal isn't VPN purity—it's reliable access to your home lab from anywhere.
