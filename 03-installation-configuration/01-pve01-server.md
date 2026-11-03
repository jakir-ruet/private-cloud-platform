### Back up the current configuration

```bash
mkdir -p /root/network-backup

cp -a /etc/network/interfaces \
  /root/network-backup/interfaces.$(date +%F-%H%M%S)

cp -a /etc/hosts \
  /root/network-backup/hosts.$(date +%F-%H%M%S)
```

```bash
ls -lh /root/network-backup/
```

### Verify the existing cluster and Ceph networks

```bash
echo "===== COROSYNC CONFIGURATION ====="
cat /etc/pve/corosync.conf

echo "===== CLUSTER STATUS ====="
pvecm status

echo "===== COROSYNC LINKS ====="
corosync-cfgtool -s

echo "===== CEPH STATUS ====="
ceph -s

echo "===== CEPH NETWORK CONFIGURATION ====="
ceph config dump | grep -E 'public_network|cluster_network'
```

```bash
# Run host machine
ip -br addr show vboxnet0
```

```bash
vi /etc/network/interfaces
```

```bash
auto lo
iface lo inet loopback

# NAT - Internet connectivity
iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.2.15/24
    gateway 10.0.2.2
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

# Host-only - Management
iface nic1 inet manual

auto vmbr1
iface vmbr1 inet static
    address 192.168.56.11/24
    bridge-ports nic1
    bridge-stp off
    bridge-fd 0

# Cluster - Corosync/KNET
auto nic2
iface nic2 inet static
    address 172.20.100.11/24

# Ceph - Public and replication traffic
auto nic3
iface nic3 inet static
    address 172.20.101.11/24

source /etc/network/interfaces.d/*
```

```bash
ifquery --list
ifreload -a
ip -br addr
ip route
```

```bash
ping -c 3 192.168.56.1
ping -c 3 1.1.1.1
getent hosts download.proxmox.com
```

### Back up the current configuration

```bash
mkdir -p /root/apt-repository-backup

cp -a /etc/apt/sources.list.d/pve-enterprise.sources \
    /root/apt-repository-backup/

cp -a /etc/apt/sources.list.d/ceph.sources \
    /root/apt-repository-backup/

echo "Repository backups:"
ls -l /root/apt-repository-backup/
```

### Disable the Enterprise repositories

```bash
mv /etc/apt/sources.list.d/pve-enterprise.sources \
   /etc/apt/sources.list.d/pve-enterprise.sources.disabled

mv /etc/apt/sources.list.d/ceph.sources \
   /etc/apt/sources.list.d/ceph.sources.disabled
```

### Configure the no-subscription repositories

Create the Proxmox VE repository definition

```bash
cat > /etc/apt/sources.list.d/pve-no-subscription.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

### Create the Ceph Squid repository definition

```bash
cat > /etc/apt/sources.list.d/ceph-no-subscription.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/ceph-squid
Suites: trixie
Components: no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

### Step 4 — Update the package indexes

```bash
apt update
apt upgrade
```

### Validate the configuration

```bash
echo "=== Enabled Proxmox repository definitions ==="
grep -RniE 'URIs:|Suites:|Components:' \
    /etc/apt/sources.list.d/*.sources

echo
echo "=== Proxmox version ==="
pveversion -v

echo
echo "=== Candidate package versions ==="
apt-cache policy proxmox-ve
```

### validate the current network configuration

```bash
echo "===== NETWORK INTERFACES ====="
cat /etc/network/interfaces

echo
echo "===== INTERFACE ADDRESSES ====="
ip -br addr

echo
echo "===== ROUTING TABLE ====="
ip route

echo
echo "===== NETWORK CONFIGURATION ====="
ifquery --list

echo
echo "===== HOST CONNECTIVITY ====="
ping -c 3 192.168.56.1

echo
echo "===== INTERNET CONNECTIVITY ====="
ping -c 3 1.1.1.1

echo
echo "===== DNS RESOLUTION ====="
getent hosts download.proxmox.com
```

### validate Proxmox services

```bash
echo "===== PROXMOX VERSION ====="
pveversion -v

echo
echo "===== PMXCFS ====="
systemctl status pve-cluster --no-pager

echo
echo "===== PROXMOX SERVICES ====="
systemctl status pvedaemon pveproxy pvestatd --no-pager

echo
echo "===== CLUSTER CONFIGURATION ====="
ls -lah /etc/pve/

echo
echo "===== COROSYNC CONFIGURATION ====="
ls -l /etc/pve/corosync.conf
```

### validate the APT repositories

```bash
echo "===== REPOSITORY FILES ====="
ls -lah /etc/apt/sources.list.d/

echo
echo "===== ENABLED REPOSITORIES ====="
grep -RniE '^(Types:|URIs:|Suites:|Components:)' \
    /etc/apt/sources.list.d/*.sources

echo
echo "===== PACKAGE INDEX UPDATE ====="
apt update

echo
echo "===== PROXMOX PACKAGE POLICY ====="
apt-cache policy proxmox-ve
```

### Configure `/etc/hosts` on all nodes

Run on **each Proxmox node**:

```bash
hostnamectl
hostname -s
cat /etc/hosts
```

Back up the current file before editing:

```bash
cp -a /etc/hosts "/etc/hosts.bak.$(date +%F-%H%M%S)"
vi /etc/hosts
```

Ensure the file contains these consistent mappings. Preserve any required IPv6 entries and other valid local entries, but remove or correct conflicting mappings for `pve01`.

```text
127.0.0.1 localhost.localdomain localhost

192.168.56.11 pve01.example.local pve01
192.168.56.12 pve02.example.local pve02
192.168.56.13 pve03.example.local pve03

# Optional VirtualBox-related aliases; not the Proxmox guest hostnames:
192.168.56.11 pve01-server
192.168.56.12 pve02-server
192.168.56.13 pve03-server
```

**Important:** Do not map `pve01` or `pve01.example.local` to the NAT address `10.0.2.15`. The canonical hostname must resolve consistently to the management IP. Do not change `/etc/hostname` if the node already has the correct short hostname.

Validate on each node:

```bash
hostname -s
hostname -f
getent hosts "$(hostname -s)"
getent hosts pve01.example.local
getent hosts pve02.example.local
getent hosts pve03.example.local
```

Expected FQDNs are `pve01.example.local`, `pve02.example.local`, and `pve03.example.local` respectively. Each name should resolve to its corresponding management IP (`192.168.56.11`, `.12`, or `.13`). Do not continue until hostname lookup succeeds on all three nodes.

## Step 2 — Verify time and network connectivity

Run on all three nodes:

```bash
timedatectl
ip -br address
```

Confirm the clock is synchronized and the Corosync/Ceph interfaces have their planned addresses. From each node, test reachability to the other nodes on the management, Corosync, and Ceph networks. For example, from `pve01`:

```bash
ping -c 3 192.168.56.12
ping -c 3 192.168.56.13
ping -c 3 -I vmbr2 172.20.100.12
ping -c 3 -I vmbr2 172.20.100.13
ping -c 3 -I vmbr3 172.20.101.12
ping -c 3 -I vmbr3 172.20.101.13
```

Adjust interface names only if the actual configuration uses different names. Do not create the cluster until hostname resolution and network tests pass.

### Initialize the Proxmox Cluster

Prepare PVE01 - On pve01-server:

- Configure hostname: pve01.
- Configure management IP: 192.168.56.11.
- Configure cluster network: 172.20.100.11.
- Configure Ceph network: 172.20.101.11.
- Verify hostname resolution.
- Verify network connectivity.
- Confirm Proxmox VE services are healthy.

```bash
pvecm create homelab-cluster
pvecm status
pvecm nodes
```

### Investigate Corosync

```bash
echo "===== COROSYNC SERVICE ====="
systemctl status corosync --no-pager -l

echo "===== COROSYNC ENABLEMENT ====="
systemctl is-enabled corosync

echo "===== COROSYNC CONFIGURATION ====="
cat /etc/pve/corosync.conf

echo "===== COROSYNC CONFIGURATION FILE ====="
cat /etc/corosync/corosync.conf

echo "===== COROSYNC JOURNAL ====="
journalctl -u corosync -b --no-pager -n 100

echo "===== COROSYNC NETWORK STATUS ====="
corosync-cfgtool -s

echo "===== CLUSTER FILESYSTEM ====="
systemctl status pve-cluster --no-pager -l
```

### Ceph Installation and Configuration

```bash
echo "===== PROXMOX VERSION ====="
pveversion -v

echo "===== CLUSTER STATUS ====="
pvecm status
pvecm nodes

echo "===== NETWORK INTERFACES ====="
ip -br addr

echo "===== ROUTING ====="
ip route

echo "===== STORAGE ====="
pvesm status

echo "===== PHYSICAL/VIRTUAL DISKS ====="
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL

echo "===== MEMORY ====="
free -h

echo "===== CEPH PACKAGE VERSION ====="
ceph --version 2>&1 || true

echo "===== CEPH REPOSITORY ====="
cat /etc/apt/sources.list.d/ceph-no-subscription.sources
```

```bash
echo "===== CURRENT CEPH SOURCES ====="
grep -R -n -E 'ceph-|Components:|URIs:' \
    /etc/apt/sources.list.d/*ceph* 2>/dev/null

echo "===== INSTALLED CEPH PACKAGES ====="
dpkg-query -W -f='${Package} ${Version}\n' \
    'ceph*' 'librados*' 2>/dev/null

echo "===== AVAILABLE CEPH VERSIONS ====="
apt-cache policy ceph
```
