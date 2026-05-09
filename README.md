# vIXP-VPN-WireGuard-

Core Tasks

1. WireGuard Server Setup on Both IXP VMs

What to do:

· Install WireGuard on Ubuntu Server 22.04 on both VM1 (IXP-EAST) and VM2 (IXP-WEST)
· Configure each WireGuard interface as its own "IXP peering LAN"
· Allocate IP addresses from two separate RFC 1918 private ranges:
  · VM1: 10.0.1.0/24
  · VM2: 10.0.2.0/24
· Enable IP forwarding on both VMs so traffic can reach the FRR route server

Key config decisions:

· Each WireGuard interface becomes a separate BGP peering subnet
· VM1 route server IP: 10.0.1.1, VM2 route server IP: 10.0.2.1
· Each student gets a unique IP on their assigned IXP's subnet
· Each FRR daemon binds to its own WireGuard IP only

2. Automated Peer Configuration Generation

· Portal (VM3) triggers WireGuard config generation on the correct IXP VM when admin approves a student
· Generate unique keypair per student on their assigned IXP
· Create the student's .conf file for download
· Add the peer entry using wg syncconf (avoids disrupting existing tunnels)

Important: Private key included in download but not stored in the database.

3. Security Hardening (Both VMs)

What to do:

· Configure UFW on both VM1 and VM2:
  · Allow UDP port 51820 (WireGuard)
  · Allow SSH from portal VM (VM3) and admin IPs only
  · Block everything else
· Ensure FRR's BGP daemon binds only to the WireGuard interface IP, never the public interface
· Verify no route leakage, all traffic stays within RFC 1918 space

---

Phase 1: VM Preparation (Both VMs)

Step 1.1: Access Each VM

VM1:

```bash
ssh username@151.158.219.194
```

VM2:

```bash
ssh username@<151.159.219.195
```

Step 1.2: Update System and Enable IP Forwarding (Both VMs)

```bash
sudo apt update && sudo apt upgrade -y
```

Enable IP forwarding:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment or add:

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

```bash
sudo sysctl -p
```

Verify:

```bash
sysctl net.ipv4.ip_forward
# Should output: net.ipv4.ip_forward = 1
```

---

Phase 2: WireGuard Installation (Both VMs)

Step 2.1: Install WireGuard (Both VMs)

```bash
sudo apt install wireguard wireguard-tools -y
```

Step 2.2: Generate Server Keypair (Both VMs)

On VM1:

```bash
sudo mkdir -p /etc/wireguard/keys
sudo chmod 700 /etc/wireguard/keys

# Generate private key
wg genkey | sudo tee /etc/wireguard/keys/server-private.key

# Generate public key from private key
sudo cat /etc/wireguard/keys/server-private.key | wg pubkey | sudo tee /etc/wireguard/keys/server-public.key

# Restrict permissions
sudo chmod 600 /etc/wireguard/keys/server-private.key
sudo chmod 644 /etc/wireguard/keys/server-public.key
```

Repeat the exact same steps on VM2 (each VM gets its own unique keypair).

Step 2.3: Create WireGuard Configuration

On VM1 (/etc/wireguard/wg0.conf):

```ini
[Interface]
Address = 10.0.1.1/24
ListenPort = 51820
PrivateKey = <vm1-server-private-key>

# SaveConfig must be false : we manage peers via syncconf
SaveConfig = false

# Participants added below by automation script
```

On VM2 (/etc/wireguard/wg0.conf):

```ini
[Interface]
Address = 10.0.2.1/24
ListenPort = 51820
PrivateKey = <vm2-server-private-key>

# SaveConfig must be false — we manage peers via syncconf
SaveConfig = false

# Participants added below by automation script
```

Replace placeholders with actual keys:

```bash
# On VM1
sudo sed -i "s|<vm1-server-private-key>|$(sudo cat /etc/wireguard/keys/server-private.key)|" /etc/wireguard/wg0.conf

# On VM2
sudo sed -i "s|<vm2-server-private-key>|$(sudo cat /etc/wireguard/keys/server-private.key)|" /etc/wireguard/wg0.conf
```

Step 2.4: Set Permissions (Both VMs)

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

Step 2.5: Start WireGuard (Both VMs)

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

Verify:

```bash
sudo wg show
# Should show interface wg0 with listening port 51820
```

---

Phase 3: Firewall Configuration (Both VMs)

```bash
# Allow WireGuard
sudo ufw allow 51820/udp

# Allow SSH (restrict to portal VM and admin IPs)
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

---

Phase 4: Student Peer Management Scripts

Install on both VM1 and VM2.

Step 4.1: Create Directories

```bash
sudo mkdir -p /opt/vixp/scripts
sudo mkdir -p /opt/vixp/configs
```

Step 4.2: Create Peer Addition Script (add-peer.sh)

```bash
sudo nano /opt/vixp/scripts/add-peer.sh
```

```bash
#!/bin/bash
# vIXP WireGuard Peer Addition Script
# Usage: add-peer.sh <student-id> <wg-ip>

set -e

STUDENT_ID=$1
WG_IP=$2

if [ -z "$STUDENT_ID" ] || [ -z "$WG_IP" ]; then
    echo "Usage: $0 <student-id> <wg-ip>"
    exit 1
fi

KEY_DIR="/etc/wireguard/keys/students"
CONFIG_DIR="/opt/vixp/configs"
PEER_FILE="/etc/wireguard/peers.conf"

mkdir -p "$KEY_DIR" "$CONFIG_DIR"

# Generate student keypair
STUDENT_PRIVATE_KEY=$(wg genkey)
STUDENT_PUBLIC_KEY=$(echo "$STUDENT_PRIVATE_KEY" | wg pubkey)

# Save keys
echo "$STUDENT_PRIVATE_KEY" > "$KEY_DIR/${STUDENT_ID}-private.key"
echo "$STUDENT_PUBLIC_KEY" > "$KEY_DIR/${STUDENT_ID}-public.key"
chmod 600 "$KEY_DIR/${STUDENT_ID}-private.key"

# Append peer to peers.conf
cat >> "$PEER_FILE" <<EOF
[Peer]
# Student: $STUDENT_ID
PublicKey = $STUDENT_PUBLIC_KEY
AllowedIPs = ${WG_IP}/32

EOF

# Generate student config file for download
SERVER_PUBLIC_KEY=$(cat /etc/wireguard/keys/server-public.key)
SERVER_ENDPOINT=$(curl -s ifconfig.me):51820

cat > "$CONFIG_DIR/${STUDENT_ID}.conf" <<EOF
[Interface]
Address = ${WG_IP}/24
PrivateKey = ${STUDENT_PRIVATE_KEY}
DNS = 1.1.1.1

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = ${WG_IP%.*}.0/24
PersistentKeepalive = 25
EOF

# Apply new peer configuration
if [ -f "$PEER_FILE" ]; then
    wg syncconf wg0 <(cat /etc/wireguard/wg0.conf "$PEER_FILE")
fi

echo "CONFIG_PATH=$CONFIG_DIR/${STUDENT_ID}.conf"
echo "PUBLIC_KEY=$STUDENT_PUBLIC_KEY"
echo "SUCCESS"
```

```bash
sudo chmod +x /opt/vixp/scripts/add-peer.sh
```

Step 4.3: Create Peer Removal Script (remove-peer.sh)
```bash
#!/bin/bash
# vIXP WireGuard Peer Removal Script
# Usage: remove-peer.sh <student-id>

STUDENT_ID=$1

if [ -z "$STUDENT_ID" ]; then
    echo "Usage: $0 <student-id>"
    exit 1
fi

echo "Removing peer: $STUDENT_ID"

PEER_FILE="/etc/wireguard/peers.conf"
KEY_DIR="/etc/wireguard/keys/students"
CONFIG_DIR="/opt/vixp/configs"

# Remove peer from WireGuard runtime first
PUB_KEY_FILE="$KEY_DIR/${STUDENT_ID}-public.key"

if [ -f "$PUB_KEY_FILE" ]; then
    PUB_KEY=$(sudo cat "$PUB_KEY_FILE")
    sudo wg set wg0 peer "$PUB_KEY" remove
    echo "Peer removed from WireGuard runtime"
else
    echo "Warning: Public key file not found at $PUB_KEY_FILE"
fi

# Remove peer block from peers.conf
if [ -f "$PEER_FILE" ]; then
    sudo sed -i "/# Student: $STUDENT_ID/,/^$/d" "$PEER_FILE"
    if [ -s "$PEER_FILE" ]; then
        sudo bash -c 'wg syncconf wg0 <(cat /etc/wireguard/wg0.conf /etc/wireguard/peers.conf)' 2>/dev/null
    else
        sudo bash -c 'wg syncconf wg0 <(cat /etc/wireguard/wg0.conf)' 2>/dev/null
    fi
fi

# Clean up keys and config
sudo rm -f "$KEY_DIR/${STUDENT_ID}-private.key"
sudo rm -f "$KEY_DIR/${STUDENT_ID}-public.key"
sudo rm -f "$CONFIG_DIR/${STUDENT_ID}.conf"

echo "SUCCESS: Peer $STUDENT_ID removed"
sudo wg show
```

Step 4.4: Create Empty Peers File (Both VMs)

```bash
sudo touch /etc/wireguard/peers.conf
```

Step 4.5: Create Full Reset Script (reset-all.sh)
```bash
sudo nano /opt/vixp/scripts/reset-all.sh
Content:
bash
#!/bin/bash
# vIXP Full VPN Reset
# Removes all student peers and regenerations keys

echo "WARNING: This will remove ALL student WireGuard peers."
read -p "Are you sure? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Aborted."
    exit 0
fi

# Clear all peers
sudo truncate -s 0 /etc/wireguard/peers.conf
sudo wg syncconf wg0 <(cat /etc/wireguard/wg0.conf)

# Remove all student keys and configs
sudo rm -rf /etc/wireguard/keys/students/*
sudo rm -rf /opt/vixp/configs/*

echo "All peers removed. System reset complete."
```

```bash
sudo chmod +x /opt/vixp/scripts/reset-all.sh
```

---

Phase 5: Set Up SSH Access from Portal (VM3) to Both IXP VMs

Step 5.1: On Portal VM (VM3), Generate SSH Keys

```bash
# Key for VM1
ssh-keygen -t ed25519 -f ~/.ssh/vixp-vm1 -N "" -C "vixp-portal@vm3"

# Key for VM2
ssh-keygen -t ed25519 -f ~/.ssh/vixp-vm2 -N "" -C "vixp-portal@vm3"
```

Step 5.2: Copy Public Keys to Each VM

```bash
# Copy to VM1
ssh-copy-id -i ~/.ssh/vixp-vm1.pub ubuntu@<vm1-private-ip>

# Copy to VM2
ssh-copy-id -i ~/.ssh/vixp-vm2.pub ubuntu@<vm2-private-ip>
```

Step 5.3: Create Wrapper Script (Both VM1 and VM2)

Same wrapper script as original — install on both VMs.

Step 5.4: Test from Portal VM

```bash
# Test VM1
ssh -i ~/.ssh/vixp-vm1 ubuntu@<vm1-private-ip> status

# Test VM2
ssh -i ~/.ssh/vixp-vm2 ubuntu@<vm2-private-ip> status
```

---

Phase 6: Manual Testing (Both VMs)

Test VM1:

```bash
sudo /opt/vixp/scripts/add-peer.sh test-student-east 10.0.1.2
sudo cat /opt/vixp/configs/test-student-east.conf
ping 10.0.1.1  # From client
sudo /opt/vixp/scripts/remove-peer.sh test-student-east
```

Test VM2:

```bash
sudo /opt/vixp/scripts/add-peer.sh test-student-west 10.0.2.2
sudo cat /opt/vixp/configs/test-student-west.conf
ping 10.0.2.1  # From client
sudo /opt/vixp/scripts/remove-peer.sh test-student-west
```

---

Phase 7: Integration with Portal Team (To be CONFIRMED)

Updated wrapper commands:

```
# For students assigned to IXP-East (VM1)
ssh -i ~/.ssh/vixp-vm1 ubuntu@<vm1-ip> add <student-id> <10.0.1.x>
ssh -i ~/.ssh/vixp-vm1 ubuntu@<vm1-ip> remove <student-id>

# For students assigned to IXP-West (VM2)
ssh -i ~/.ssh/vixp-vm2 ubuntu@<vm2-ip> add <student-id> <10.0.2.x>
ssh -i ~/.ssh/vixp-vm2 ubuntu@<vm2-ip> remove <student-id>
```

IP allocation:

· VM1: Start from 10.0.1.2 and increment
· VM2: Start from 10.0.2.2 and increment

---

Quick Reference: Two IXP Subnets

 VM1 (IXP-East) VM2 (IXP-West)
WireGuard subnet 10.0.1.0/24 10.0.2.0/24
Route server IP 10.0.1.1 10.0.2.1
Student IPs 10.0.1.2 to 10.0.1.254 10.0.2.2 to 10.0.2.254
Configs stored in /opt/vixp/configs/ /opt/vixp/configs/
SSH key on portal ~/.ssh/vixp-vm1 ~/.ssh/vixp-vm2

---

import subprocess

def add_student_to_ixp(student_id, ixp):
    if ixp == "east":
        ip = get_next_available_ip("10.0.1.0/24")
        result = subprocess.run([
            "ssh", "-i", "/home/ubuntu/.ssh/vixp-vm1",
            "ubuntu@192.168.64.10",
            "/opt/vixp/scripts/wrapper.sh", "add", student_id, ip
        ], capture_output=True, text=True)
    else:  # west
        ip = get_next_available_ip("10.0.2.0/24")
        result = subprocess.run([
            "ssh", "-i", "/home/ubuntu/.ssh/vixp-vm2",
            "ubuntu@192.168.64.20",
            "/opt/vixp/scripts/wrapper.sh", "add", student_id, ip
        ], capture_output=True, text=True)

    # Parse CONFIG_PATH from result.stdout
    # SCP the .conf file back to VM3 for student download
