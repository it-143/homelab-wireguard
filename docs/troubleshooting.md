# Troubleshooting Guide

Common issues encountered when setting up WireGuard and their solutions.

## Connection Issues

### No Handshake / Data Sent But Not Received

**Symptoms:**
- WireGuard app shows "Data sent: XXX B" but no "Data received"
- No "Latest handshake" timestamp
- Connection appears active but nothing works

**Causes & Solutions:**

1. **Wrong port forward protocol**
   - WireGuard uses UDP, not TCP
   - Check your router's port forwarding rules
   - Delete any TCP rules for port 51820 and create UDP rule

2. **Port forward pointing to wrong IP**
   - Should point to container IP (e.g., 192.168.X.210), not host IP
   - Verify with `docker inspect wg-easy | grep -i ipaddress`

3. **Firewall blocking UDP**
   - Check if UFW or iptables is blocking port 51820
   - `sudo ufw allow 51820/udp`

### Handshake Works But Can't Reach Services

**Symptoms:**
- Handshake timestamp shows recent time
- Data sent and received
- Can't ping or access services on home network

**Causes & Solutions:**

1. **Macvlan host isolation**
   - Containers on macvlan can't communicate with their host
   - Create macvlan shim interface (see [setup guide](setup.md#step-7-fix-macvlan-host-communication))

2. **Routing not configured**
   - Check `WG_ALLOWED_IPS` includes your LAN subnet
   - Should be: `192.168.X.0/24,10.8.0.0/24`

3. **Client AllowedIPs mismatch**
   - Client config must include networks you want to access
   - Check the `[Peer]` section in client config

### Can Ping Some IPs But Not Others

**Symptoms:**
- Can ping router (192.168.X.1) ✓
- Can ping container (192.168.X.210) ✓
- Cannot ping server (192.168.X.181) ✗

**Cause:** Macvlan isolation - the WireGuard container can't reach its host.

**Solution:** Create the macvlan shim interface:

```bash
sudo ip link add macvlan-shim link <INTERFACE> type macvlan mode bridge
sudo ip addr add 192.168.X.211/32 dev macvlan-shim
sudo ip link set macvlan-shim up
sudo ip route add 192.168.X.210/32 dev macvlan-shim
```

## Page Loading Issues

### Small Requests Work, Pages Don't Load

**Symptoms:**
- `curl` gets redirect responses (302) quickly
- Actual page content hangs forever
- Browser shows blank page
- Works fine when accessing locally on server

**Cause:** MTU too high. Large packets are getting fragmented and dropped.

**Solution:** Lower MTU from 1420 to 1280:

1. Edit client config
2. Change `MTU = 1420` to `MTU = 1280`
3. Save and reconnect

Or set `WG_MTU=1280` when creating the container.

### DNS Not Resolving

**Symptoms:**
- Can ping IP addresses
- Can't resolve hostnames
- `.local` or `.home` domains don't work

**Causes & Solutions:**

1. **Wrong DNS in config**
   - Check `DNS` line in client config
   - Should point to your Pi-hole or router

2. **Pi-hole not accessible**
   - Verify Pi-hole is running and reachable from VPN subnet
   - May need to allow 10.8.0.0/24 in Pi-hole settings

## iOS / Mobile Specific Issues

### IPv4 Endpoint Converts to IPv6

**Symptoms:**
- Set endpoint to IPv4 address (e.g., `208.59.63.195:51820`)
- After saving, shows IPv6 (e.g., `[2607:7700:0:2:0:2:xxxx:xxxx]:51820`)
- Works on WiFi, fails on cellular

**Cause:** Your cellular carrier uses IPv6-only with NAT64. iOS synthesizes an IPv6 address from the IPv4.

**Solutions:**

1. **Use Tailscale for cellular** (recommended)
   - Tailscale handles NAT64 properly
   - Keep WireGuard for regular WiFi networks

2. **Hardcode IPv4 in config file**
   - Download config from wg-easy web UI
   - Edit on computer: change Endpoint to your public IPv4
   - Transfer edited config to phone
   - Import as new tunnel
   - Note: Won't survive IP changes if using dynamic DNS

3. **Enable IPv6 on home network**
   - If your ISP provides IPv6, enable it on your router
   - Configure WireGuard to listen on IPv6
   - Allows direct IPv6 connection from cellular

### VPN Connects But Nothing Loads on Cellular

**Cause:** NAT64 doesn't reliably translate WireGuard's UDP traffic.

**Solution:** Use Tailscale for cellular access. It's designed to handle these situations.

## Container Issues

### Container Won't Start

**Check logs:**
```bash
docker logs wg-easy
```

**Common causes:**

1. **Capabilities not granted**
   - Need `--cap-add=NET_ADMIN --cap-add=SYS_MODULE`

2. **Sysctl not set**
   - Need `--sysctl="net.ipv4.conf.all.src_valid_mark=1"`
   - Need `--sysctl="net.ipv4.ip_forward=1"`

3. **Invalid password hash**
   - Regenerate bcrypt hash
   - Ensure single quotes around hash in docker command

### Can't Access Web UI

**Symptoms:**
- Container running
- Can't reach `http://192.168.X.210:51821`

**Causes & Solutions:**

1. **Accessing from host machine**
   - Macvlan isolation prevents host from reaching container
   - Access from a different device on the network

2. **Wrong IP**
   - Verify container IP: `docker inspect wg-easy | grep -i ipaddress`

3. **Container not on macvlan**
   - Check network: `docker inspect wg-easy | grep -i networkmode`

### IP Conflict with IoT Device

**Symptoms:**
- Port forward won't save
- Intermittent connectivity
- Another device has same IP

**Solution:**
1. Check router device list for conflicts
2. Choose unused IP
3. Recreate container:

```bash
docker stop wg-easy && docker rm wg-easy
# Run docker command with new --ip value
```

4. Update port forward to new IP

## Testing Commands

```bash
# Check if port is open (from external network)
nc -zvu your.domain.com 51820

# Check container is healthy
docker ps | grep wg-easy

# View WireGuard interface status
docker exec wg-easy wg show

# Check routes inside container
docker exec wg-easy ip route

# Ping test with specific interface
ping -c 4 192.168.X.181

# Verbose curl for debugging
curl -v http://192.168.X.181:8920
```

## Still Stuck?

If none of these solutions work:

1. Check the [wg-easy GitHub issues](https://github.com/wg-easy/wg-easy/issues)
2. Verify each step in the [setup guide](setup.md)
3. Consider using Tailscale as a reliable fallback
