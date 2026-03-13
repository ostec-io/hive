# Linux VM Setup Guide (Mac/Windows)

HIVE requires a **Linux Docker Engine** for Swarm/overlay networking. Docker Desktop is not supported for Swarm.
This guide shows how to run a Linux VM that works with Swarm.

---

## Why a Linux VM?

Docker Desktop runs inside a hidden VM with NAT and non-standard networking.
In Swarm mode this commonly causes:
- Overlay DNS failures (services cannot resolve other services).
- Swarm advertise-addr issues (nodes report the wrong IP).
- Unreliable metrics/agent behavior.

Using a Linux VM with a real network interface avoids these issues.

---

## VM Requirements

- **OS**: Ubuntu 22.04/24.04 (server)
- **CPU**: 2+ cores (4 recommended)
- **RAM**: 8 GB minimum (16 recommended)
- **Disk**: 30 GB+
- **Networking**: **Bridged** (recommended) or Host-only if you only test locally

If you plan multi-node Swarm tests, use **Bridged** networking so other hosts can reach the VM.

---

## Create a VM (Pick One)

### Mac
- UTM (Apple Silicon) or VirtualBox (Intel)
- Create Ubuntu Server VM
- Network: **Bridged**

### Windows
- Hyper-V or VirtualBox
- Create Ubuntu Server VM
- Network: **External/Bridged**

---

## Install Docker Engine (Ubuntu)

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```

Log out and back in, then verify:
```bash
docker version
```

---

## Enable Swarm (Single Node)

```bash
docker swarm init --advertise-addr <VM_IP>
```

Verify:
```bash
docker node ls
```

---

## Firewall (If UFW is enabled)

Allow Swarm ports **only** on your private subnet or VPN:

```bash
sudo ufw allow from <PRIVATE_SUBNET> to any port 2377 proto tcp
sudo ufw allow from <PRIVATE_SUBNET> to any port 7946 proto tcp
sudo ufw allow from <PRIVATE_SUBNET> to any port 7946 proto udp
sudo ufw allow from <PRIVATE_SUBNET> to any port 4789 proto udp
```

---

## Next Steps

- [Getting Started](getting-started.md)
- [Development Environment](../environments/development.md)
