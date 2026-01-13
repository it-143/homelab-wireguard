# Self-Hosted WireGuard VPN for Home Lab

A comprehensive guide to setting up a self-hosted WireGuard VPN server using Docker, with solutions for common networking challenges like NAT64, macvlan routing, and MTU issues.

## Why WireGuard?

After evaluating various VPN solutions, I chose to build a self-hosted WireGuard VPN for the following reasons:

### Security
As an open-source protocol with robust security features, WireGuard provides strong protection for your data. The codebase is minimal (~4,000 lines of code) compared to OpenVPN (~100,000 lines), making it easier to audit and less prone to vulnerabilities.

### Flexibility
By customizing the configuration and adding plugins or modules as needed, you can tailor the VPN to fit specific use cases. Whether you're securing a home lab, enabling remote access to self-hosted services, or protecting IoT devices, WireGuard adapts to your needs.

### Cost-Effective
Creating a VPN from scratch using WireGuard is more cost-effective than licensing proprietary VPN solutions or paying for commercial VPN subscriptions. Once set up, there are no recurring fees—just your existing hardware and internet connection.

### Performance
WireGuard is designed for speed. It lives in the Linux kernel, uses state-of-the-art cryptography, and establishes connections almost instantaneously compared to traditional VPN protocols.

## What's Included

- **[SETUP.md](SETUP.md)** - Complete installation and configuration guide
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - Common issues and solutions
- **[scripts/](scripts/)** - Helper scripts for persistent configuration

## Quick Start

1. Install Docker on your server
2. Set up port forwarding (UDP 51820)
3. Follow the guide in [SETUP.md](SETUP.md)

## Use Cases

- **Remote Access**: Access your home network services (media servers, file shares, etc.) from anywhere
- **Ad Blocking on the Go**: Route traffic through Pi-hole even on mobile data
- **Secure Public WiFi**: Encrypt your traffic when on untrusted networks
- **Self-Hosted Alternative**: Take control of your VPN instead of relying on third-party services

## Lessons Learned

This project documents real-world troubleshooting of issues including:

- IPv6/NAT64 cellular network compatibility
- Docker macvlan networking limitations
- MTU optimization for reliable connections
- Port forwarding configuration pitfalls

## Complementary Tools

For networks where WireGuard alone isn't enough (like IPv6-only cellular with NAT64), consider running [Tailscale](https://tailscale.com/) alongside your self-hosted WireGuard. The two complement each other well:

| Solution | Best For |
|----------|----------|
| WireGuard (self-hosted) | Regular networks, full control |
| Tailscale | Difficult NAT situations, zero-config |

## Contributing

Found an issue or have an improvement? Pull requests are welcome!

## License

MIT License - See [LICENSE](LICENSE) for details.

---

By embracing the benefits of WireGuard and addressing potential challenges, you can create a secure, flexible, and high-performance VPN that meets your specific needs.
