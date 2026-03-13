# WireGuard VPN Setup

## Overview

HIVE production deployments use a WireGuard VPN mesh for secure inter-node communication. This guide covers setting up a full mesh topology where all nodes can communicate directly over encrypted tunnels.

## Architecture

```
        Node 1 (10.0.0.1)
        [CoreDNS + Traefik]
           /        \
          /          \
    Node 2 -------- Node 3
  (10.0.0.2)      (10.0.0.3)

        ↑
   VPN Clients
  (10.0.0.10+)
  DNS → 10.0.0.1
```

**VPN Subnet**: `10.0.0.0/24`
**WireGuard Port**: `51820/udp`
**Internal DNS**: `10.0.0.1:53` (CoreDNS on Node 1)

## Prerequisites

- 3+ Linux machines (Ubuntu/Debian recommended)
- Root or sudo access on all machines
- At least one machine with a public IP
- UDP port 51820 open on firewalls

## Step 1: Install WireGuard

Run on **all nodes**:

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install wireguard wireguard-tools -y

# RHEL/CentOS/Rocky
sudo dnf install epel-release -y
sudo dnf install wireguard-tools -y

# Verify installation
wg --version
```

## Step 2: Generate Keys

Run on **each node separately**:

```bash
# Create config directory
sudo mkdir -p /etc/wireguard
cd /etc/wireguard

# Generate private and public key pair
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey

# Secure the private key
sudo chmod 600 privatekey

# View your keys
echo "Private Key: $(sudo cat privatekey)"
echo "Public Key: $(sudo cat publickey)"
```

**Record the keys** for each node:

| Node   | Private Key         | Public Key          | VPN IP     | Public IP          |
|--------|---------------------|---------------------|------------|--------------------|
| Node 1 | `<node1_private>`   | `<node1_public>`    | 10.0.0.1   | `<your_node1_ip>`  |
| Node 2 | `<node2_private>`   | `<node2_public>`    | 10.0.0.2   | `<your_node2_ip>`  |
| Node 3 | `<node3_private>`   | `<node3_public>`    | 10.0.0.3   | `<your_node3_ip>`  |

## Step 3: Configure Node 1 (Manager + DNS)

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
# Node 1 Configuration (Manager + DNS + Gateway)
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <node1_private_key>

# Enable IP forwarding for cluster traffic
PostUp = sysctl -w net.ipv4.ip_forward=1
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT

# Peer: Node 2
[Peer]
PublicKey = <node2_public_key>
AllowedIPs = 10.0.0.2/32
Endpoint = <node2_public_ip>:51820
PersistentKeepalive = 25

# Peer: Node 3
[Peer]
PublicKey = <node3_public_key>
AllowedIPs = 10.0.0.3/32
Endpoint = <node3_public_ip>:51820
PersistentKeepalive = 25
```

## Step 4: Configure Node 2

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.0.0.2/24
ListenPort = 51820
PrivateKey = <node2_private_key>

PostUp = sysctl -w net.ipv4.ip_forward=1
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT

# Peer: Node 1
[Peer]
PublicKey = <node1_public_key>
AllowedIPs = 10.0.0.1/32
Endpoint = <node1_public_ip>:51820
PersistentKeepalive = 25

# Peer: Node 3
[Peer]
PublicKey = <node3_public_key>
AllowedIPs = 10.0.0.3/32
Endpoint = <node3_public_ip>:51820
PersistentKeepalive = 25
```

## Step 5: Configure Node 3

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.0.0.3/24
ListenPort = 51820
PrivateKey = <node3_private_key>

PostUp = sysctl -w net.ipv4.ip_forward=1
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT

# Peer: Node 1
[Peer]
PublicKey = <node1_public_key>
AllowedIPs = 10.0.0.1/32
Endpoint = <node1_public_ip>:51820
PersistentKeepalive = 25

# Peer: Node 2
[Peer]
PublicKey = <node2_public_key>
AllowedIPs = 10.0.0.2/32
Endpoint = <node2_public_ip>:51820
PersistentKeepalive = 25
```

## Step 6: Secure and Start

Run on **all nodes**:

```bash
# Secure config files
sudo chmod 600 /etc/wireguard/wg0.conf

# Open firewall port
sudo ufw allow 51820/udp
sudo ufw reload

# Start WireGuard
sudo wg-quick up wg0

# Enable auto-start on boot
sudo systemctl enable wg-quick@wg0
```

## Step 7: Verify Connectivity

```bash
# Check WireGuard status
sudo wg show

# Expected output shows peers with recent handshakes:
# peer: <peer_public_key>
#   endpoint: x.x.x.x:51820
#   allowed ips: 10.0.0.x/32
#   latest handshake: 5 seconds ago
#   transfer: 1.24 KiB received, 1.56 KiB sent

# Test connectivity from each node
ping -c 3 10.0.0.1
ping -c 3 10.0.0.2
ping -c 3 10.0.0.3
```

## Adding Laptop/Client Devices

### IP Assignment Scheme

| Machine Type | VPN IP Range |
|--------------|--------------|
| Swarm Managers | 10.0.0.1-9 |
| Swarm Workers | 10.0.0.10-19 |
| Frontend Nodes | 10.0.0.20-29 |
| Laptops/Dev | 10.0.0.30-49 |

### Client Configuration

For laptops and development machines:

```ini
[Interface]
Address = 10.0.0.30/24
PrivateKey = <laptop_private_key>
# Use VPN DNS for internal hostname resolution
DNS = 10.0.0.1

[Peer]
PublicKey = <node1_public_key>
AllowedIPs = 10.0.0.0/24
Endpoint = <node1_public_ip>:51820
PersistentKeepalive = 25
```

### Add Peer to Existing Nodes

On each server node, add the new peer:

```bash
# Live update (no restart required)
sudo wg set wg0 peer <laptop_public_key> allowed-ips 10.0.0.30/32

# Or add to config for persistence
# Then reload:
sudo wg syncconf wg0 <(wg-quick strip wg0)
```

## Internal DNS Setup

When using internal services, configure CoreDNS on Node 1 to resolve internal hostnames.

### DNS Resolution Table

| Hostname | Resolves To | Service |
|----------|-------------|---------|
| `auth.vpn.example.com` | 10.0.0.1 | Keycloak |
| `traefik.vpn.example.com` | 10.0.0.1 | Traefik Dashboard |
| `rabbitmq.vpn.example.com` | 10.0.0.1 | RabbitMQ Management |
| `neo4j.vpn.example.com` | 10.0.0.1 | Neo4j Browser |

All hostnames point to Traefik on Node 1, which routes based on the `Host` header.

### Testing DNS

```bash
# Test DNS resolution
dig @10.0.0.1 auth.vpn.example.com

# Test HTTPS access
curl -I https://auth.vpn.example.com
```

## Docker Swarm Integration

Initialize Docker Swarm using VPN IPs:

```bash
# On manager node (Node 1)
docker swarm init --advertise-addr 10.0.0.1

# On worker nodes
docker swarm join --token <token> 10.0.0.1:2377
```

**Important**: Swarm ports (2377, 7946, 4789) should only be accessible via VPN, not on public IPs.

## Troubleshooting

### No Handshake Occurring

```bash
# Check if WireGuard is running
sudo wg show

# Check if port is open
sudo ss -ulnp | grep 51820

# Check firewall
sudo iptables -L -n | grep 51820
```

### Handshake But No Ping

```bash
# Check routing
ip route show

# Check if interface is up
ip addr show wg0

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward
# Should return 1
```

### Restart WireGuard

```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```

## Quick Reference

| Action | Command |
|--------|---------|
| Generate keys | `wg genkey \| tee privatekey \| wg pubkey > publickey` |
| Start VPN | `sudo wg-quick up wg0` |
| Stop VPN | `sudo wg-quick down wg0` |
| Check status | `sudo wg show` |
| Enable on boot | `sudo systemctl enable wg-quick@wg0` |
| View config | `sudo cat /etc/wireguard/wg0.conf` |
| Add peer live | `sudo wg set wg0 peer <pubkey> allowed-ips <ip>/32` |

## Security Best Practices

1. **Rotate keys periodically**: Generate new key pairs every 6-12 months
2. **Use strong endpoints**: Prefer public IPs or stable dynamic DNS
3. **Limit AllowedIPs**: Only allow necessary IP ranges
4. **Monitor connections**: Check `wg show` regularly for unexpected peers
5. **Backup configurations**: Store encrypted copies of `wg0.conf` files

## See Also

- [Security Architecture](security.md) - Zero-trust model
- [Swarm Setup](swarm-setup.md) - Docker Swarm deployment
- [Production Environment](../environments/production.md) - Dual-swarm architecture
