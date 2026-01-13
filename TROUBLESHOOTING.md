# Troubleshooting Guide

Common issues encountered when setting up WireGuard and their solutions.

## Connection Issues

### No handshake / Data sent but nothing received

**Symptoms:**
- WireGuard shows "Data sent: XXX B" but no "Data received"
- No "Latest handshake" timestamp
- Connection appears active but nothing works

**Causes & Solutions:**

1. **Port forwarding is TCP instead of UDP**
   - WireGuard uses UDP exclusively
   - Check your router settings and change to UDP

2. **Port forward points to wrong IP**
   - Must point to the container IP (e.g., 192.168.X.210)
   - Not the host IP (e.g., 192.168.X.181)

3. **Firewall blocking UDP 51820**
   - Check `sudo ufw status`
   - Allow with `sudo ufw allow 51820/udp`

4. **ISP blocking port 51820**
   - Try a different external port (e.g., 443)
   - Update `WG_HOST` or use port mapping

### Handshake succeeds but can't reach services

**Symptoms:**
- Latest handshake shows recent time
- Data flowing both directions
- Can't ping or access local services

**Causes & Solutions:**

1. **Macvlan isolation** (can't reach the host server)
   - See "Macvlan Shim" section below

2. **Allowed IPs misconfigured**
   - Ensure `WG_ALLOWED_IPS` includes your LAN subnet
   - Example: `192.168.1.0/24,10.8.0.0/24`

3. **Client-side routing**
   - Check client config has correct `AllowedIPs`

## Macvlan Shim

### Can ping router but not the server

**Symptom:**
```bash
ping 192.168.X.1    # Works (router)
ping 192.168.X.181  # Fails (server)
```

**Cause:**
Docker macvlan containers cannot communicate with their host by default. This is a Linux kernel limitation, not a bug.

**Solution:**
Create a shim interface:

```bash
# Replace <INTERFACE> with your network interface (eth0, enp2s0f0, etc.)
# Replace X.210 with your wg-easy container IP
# Replace X.211 with an unused IP for the shim

sudo ip link add macvlan-shim link <INTERFACE> type macvlan mode bridge
sudo ip addr add 192.168.X.211/32 dev macvlan-shim
sudo ip link set macvlan-shim up
sudo ip route add 192.168.X.210/32 dev macvlan-shim
```

See `scripts/50-macvlan-shim` for a persistent version.

## MTU Issues

### Pages don't load / Blank pages

**Symptoms:**
- Small requests work (redirects, pings)
- Large content fails (web pages, file transfers)
- `curl` hangs after initial response
- Browser shows blank page

**Cause:**
MTU (Maximum Transmission Unit) is too high. Large packets get fragmented and dropped, especially over VPN tunnels.

**Solution:**
Lower MTU from default 1420 to 1280:

**Server side** (when creating container):
```bash
-e WG_MTU=1280
```

**Client side** (edit config):
```ini
[Interface]
...
MTU = 1280
```

### Why 1280?

- Standard MTU: 1500 bytes
- WireGuard overhead: ~60-80 bytes
- Safe WireGuard MTU: 1420
- Conservative (for nested tunnels, PPPoE): 1280
- 1280 is also the minimum IPv6 MTU, so it's widely compatible

## IPv6 / NAT64 Issues

### iOS shows IPv6 endpoint instead of IPv4

**Symptom:**
Even with IPv4 address in config, iPhone shows:
```
Endpoint: [2607:7700:0:2:0:2:xxxx:xxxx]:51820
```

**Cause:**
Your cellular carrier uses IPv6-only with NAT64. The carrier synthesizes an IPv6 address from your IPv4 endpoint. The last 32 bits of the IPv6 address are your IPv4 in hex.

**Solutions:**

1. **Use Tailscale for cellular**
   - Tailscale handles NAT64 properly
   - Install on server and phone, sign in with same account
   - "Just works" through difficult NAT

2. **Accept the limitation**
   - NAT64 translation of WireGuard UDP is unreliable
   - Use WireGuard for regular WiFi networks only

3. **Get native IPv6**
   - If your ISP supports IPv6, enable it on your router
   - Configure WireGuard to listen on IPv6
   - Cellular clients connect directly via IPv6

### Testing IPv6 availability

```bash
# Check if your ISP provides IPv6
curl -6 ifconfig.me

# Should return an IPv6 address starting with 2 or 3
# If it returns IPv4 or fails, your ISP doesn't support IPv6

# Check local IPv6
ifconfig | grep inet6

# Look for addresses starting with 2 or 3 (global)
# fe80:: addresses are link-local only
```

## IP Conflicts

### Container IP already in use

**Symptoms:**
- Intermittent connectivity
- Port forward won't save
- Multiple devices showing same IP in router

**Cause:**
Another device (often IoT like smart bulbs) has the same IP you assigned to wg-easy.

**Solution:**

1. Check router's device list for the IP
2. Choose an unused IP (preferably high in the range, e.g., .210, .220)
3. Recreate container with new IP:

```bash
docker stop wg-easy && docker rm wg-easy
# Run docker command with new --ip value
```

4. Update port forward to new IP

## DNS Issues

### Can't resolve hostnames through VPN

**Symptoms:**
- Can ping IPs but not hostnames
- `nslookup` fails

**Cause:**
DNS server unreachable or misconfigured.

**Solutions:**

1. **Check WG_DEFAULT_DNS setting**
   - Should be reachable from VPN subnet
   - Try your router IP or 1.1.1.1 as fallback

2. **Pi-hole specific**
   - Ensure Pi-hole allows queries from VPN subnet (10.8.0.0/24)
   - Check Pi-hole settings → Interface → "Permit all origins"

3. **Test DNS directly**
   ```bash
   # From connected client
   nslookup google.com 192.168.X.1  # Your DNS server IP
   ```

## Container Issues

### Container won't start

Check logs:
```bash
docker logs wg-easy
```

Common issues:
- **Permission denied**: Needs `--cap-add=NET_ADMIN` and `--cap-add=SYS_MODULE`
- **Address already in use**: Another container/process using the IP
- **Invalid password hash**: Ensure proper escaping of special characters

### Container loses config after restart

Ensure volume is mounted:
```bash
-v ~/wireguard:/etc/wireguard
```

Check the directory exists and has proper permissions:
```bash
ls -la ~/wireguard
```

## Useful Diagnostic Commands

```bash
# Check WireGuard status inside container
docker exec wg-easy wg show

# View container network settings
docker inspect wg-easy | grep -A 20 "Networks"

# Check if port is open (from external network)
nc -zvu your.domain.com 51820

# Test from server to container
ping 192.168.X.210  # Only works with macvlan shim

# Check routing table
ip route

# Monitor traffic (on server)
sudo tcpdump -i <INTERFACE> udp port 51820
```

## Getting Help

If you're still stuck:

1. Check [wg-easy GitHub Issues](https://github.com/wg-easy/wg-easy/issues)
2. Search [r/WireGuard](https://reddit.com/r/WireGuard)
3. Review [WireGuard documentation](https://www.wireguard.com/)
