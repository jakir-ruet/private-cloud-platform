# Private Cloud Platform Lab

## Objective

Build a nested private cloud on Ubuntu 24.04.5 LTS using VirtualBox,
three Proxmox VE nodes, Ceph distributed storage, and enterprise-style
application, security, observability, automation, and disaster recovery
workflows.

## Target architecture

- Physical host: Ubuntu 24.04.5 LTS
- Hypervisor: VirtualBox
- Virtualization platform: Three-node Proxmox VE cluster
- Distributed storage: Ceph
- Database: Oracle Database 26ai
- Backend: Java 25 and Spring Boot
- Frontend: Next.js
- Reverse proxy: Nginx
- Security: Network segmentation, TLS, access control, hardening
- Observability: Metrics, logs, dashboards, alerts
- Automation: Terraform and Ansible
- Resilience: Backups, restore testing, migration, HA, disaster recovery

## Implementation policy

- Implement one phase at a time.
- Validate every phase before continuing.
- Preserve existing host data.
- Never initialize a system disk as a Ceph OSD.
- Test backups by restoring them.
- This nested lab shares one physical host and is not physical-host HA.

### Project's master plan

| Phase | Scope                                    | Completion criteria                                                                   |
| ----- | ---------------------------------------- | ------------------------------------------------------------------------------------- |
| 01    | Host baseline and project initialization | Host, disk, RAM, virtualization, and existing VMs inventoried                         |
| 02    | VirtualBox network and lab design        | Management, cluster, Ceph, workload, and internet connectivity designed and validated |
| 03    | Three Proxmox VE nodes                   | Three nodes installed and reachable                                                   |
| 04    | Proxmox cluster                          | All three nodes joined and quorum validated                                           |
| 05    | Ceph distributed storage                 | MON/MGR services, OSDs, pools, and health checks validated                            |
| 06    | Proxmox HA and migration                 | VM migration, HA behavior, and recovery tests completed                               |
| 07    | Oracle Database 26ai                     | Database VM deployed and database connectivity validated                              |
| 08    | Java 25, Spring Boot, and Next.js        | Backend and frontend deployed with end-to-end connectivity                            |
| 09    | Security and observability               | Network controls, TLS, access controls, metrics, logs, and alerts configured          |
| 10    | Automation and disaster recovery         | Ansible/Terraform workflows, backup, restore, and recovery procedures tested          |

### Our initial architecture

```bash
Ubuntu 24.04.5 LTS — Physical Host
│
├── VirtualBox
│   │
│   ├── Proxmox Node 01
│   │   ├── Proxmox VE
│   │   ├── Ceph MON / MGR
│   │   └── Dedicated Ceph OSD disk
│   │
│   ├── Proxmox Node 02
│   │   ├── Proxmox VE
│   │   ├── Ceph MON / MGR
│   │   └── Dedicated Ceph OSD disk
│   │
│   └── Proxmox Node 03
│       ├── Proxmox VE
│       ├── Ceph MON / MGR
│       └── Dedicated Ceph OSD disk
│
├── Proxmox Cluster
│   ├── VM migration
│   ├── High availability
│   └── Recovery testing
│
├── Application Workloads
│   ├── Oracle Database 26ai
│   ├── Java 25 / Spring Boot
│   ├── Next.js
│   └── Reverse proxy
│
├── Security and Observability
│   ├── Network segmentation
│   ├── TLS and access control
│   ├── Metrics and dashboards
│   └── Logs and alerts
│
└── Automation and Disaster Recovery
    ├── Ansible
    ├── Terraform
    ├── Backups
    └── Restore and recovery tests
```

### Complete private-cloud project

| Phase | Component             | Learning objective                                           |
| ----- | --------------------- | ------------------------------------------------------------ |
| 1     | Ubuntu + VirtualBox   | Host preparation, resource allocation, nested virtualization |
| 2     | Proxmox VE            | Three-node cluster, networking, quorum                       |
| 3     | Ceph                  | OSDs, MONs, replicated pools, storage health                 |
| 4     | Proxmox HA            | VM migration, failure testing, recovery                      |
| 5     | Oracle Database 26ai  | Database deployment, storage, backup, and recovery           |
| 6     | Java 25 + Spring Boot | Backend services and API deployment                          |
| 7     | Next.js               | Frontend deployment and reverse-proxy integration            |
| 8     | Security              | Network segmentation, TLS, IAM, secrets, and hardening       |
| 9     | Observability         | Metrics, logs, dashboards, and alerts                        |
| 10    | Automation and DR     | Terraform/Ansible, backups, and disaster-recovery exercises  |

### Proxmox node addressing

| Node / VM Name | Management IP      | Cluster IP         | Ceph IP            |
| -------------- | ------------------ | ------------------ | ------------------ |
| `pve01-server` | `192.168.56.11/24` | `172.20.100.11/24` | `172.20.101.11/24` |
| `pve02-server` | `192.168.56.12/24` | `172.20.100.12/24` | `172.20.101.12/24` |
| `pve03-server` | `192.168.56.13/24` | `172.20.100.13/24` | `172.20.101.13/24` |

### Prepare VM (Confirmed VM specification)

| VM name               |                `pve01-server` |                `pve02-server` |                `pve03-server` |  Total |
| --------------------- | ----------------------------: | ----------------------------: | ----------------------------: | -----: |
| vCPU                  |                             2 |                             2 |                             2 | 6 vCPU |
| RAM                   |                         3 GiB |                         3 GiB |                         3 GiB |  9 GiB |
| System disk           |                         32 GB |                         32 GB |                         32 GB |  96 GB |
| Initial Ceph OSD disk |                         10 GB |                         10 GB |                         10 GB |  30 GB |
| Network adapters (4)  | NAT, host-only, cluster, Ceph | NAT, host-only, cluster, Ceph | NAT, host-only, cluster, Ceph |     12 |
| Hypervisor            |                    Proxmox VE |                    Proxmox VE |                    Proxmox VE |      — |
| Installation ISO      |   `/opt/proxmox-ve_9.2-1.iso` |   `/opt/proxmox-ve_9.2-1.iso` |   `/opt/proxmox-ve_9.2-1.iso` |      — |

### Proposed network design

| Network               | Subnet / Addressing      | VirtualBox Attachment     | Purpose                                    |
| --------------------- | ------------------------ | ------------------------- | ------------------------------------------ |
| Management            | `192.168.56.0/24`        | Host-only: `vboxnet0`     | Host-to-Proxmox management                 |
| Proxmox cluster       | `172.20.100.0/24`        | Internal: `pve-cluster`   | Cluster and Corosync communication         |
| Ceph storage          | `172.20.101.0/24`        | Internal: `pve-ceph`      | Ceph storage traffic                       |
| Application workloads | `192.168.57.0/24`        | Internal: `pve-workloads` | Application VM networking; configure later |
| Internet access       | VirtualBox NAT with DHCP | NAT adapter               | Internet connectivity                      |

| Adapter   | VirtualBox network               | Purpose                       | Example configuration |
| --------- | -------------------------------- | ----------------------------- | --------------------- |
| Adapter 1 | NAT                              | Internet access               | VirtualBox DHCP       |
| Adapter 2 | Host-only (`vboxnet0`)           | Host-to-Proxmox management    | `192.168.56.11–13/24` |
| Adapter 3 | Internal Network (`pve-cluster`) | Proxmox cluster communication | `172.20.100.11–13/24` |
| Adapter 4 | Internal Network (`pve-ceph`)    | Ceph storage traffic          | `172.20.101.11–13/24` |

#### VirtualBox graphical interface

**Configure Adapter 1:** NAT

**Configure Adapter 1:** Host-only (`vboxnet0`)

Repeat these steps for each VM.

1. Open VirtualBox Manager.
2. Select pve01-server.
3. Ensure the VM is Powered Off.
4. Click Settings → Network.

**Configure Adapter 3:**

- Enable Network Adapter: ✓
- Attached to: Internal Network
- Name: pve-cluster
- Cable Connected: ✓

Click the Adapter 4 tab.

**Configure Adapter 4:**

- Enable Network Adapter: ✓
- Attached to: Internal Network
- Name: pve-ceph
- Cable Connected: ✓

Click OK.

Repeat for pve02-server and pve03-server, using the exact same network names.

| VM             | Adapter 3 name | Adapter 4 name |
| -------------- | -------------- | -------------- |
| `pve01-server` | `pve-cluster`  | `pve-ceph`     |
| `pve02-server` | `pve-cluster`  | `pve-ceph`     |
| `pve03-server` | `pve-cluster`  | `pve-ceph`     |

> `pve01-server`, `pve02-server` & `pve03-server` ready for use.

### Step 1: Back up the current configuration on `pve01-server`, `pve02-server` & `pve03-server`

```bash
mkdir -p /root/apt-repository-backup

cp -a /etc/apt/sources.list.d/pve-enterprise.sources \
    /root/apt-repository-backup/

cp -a /etc/apt/sources.list.d/ceph.sources \
    /root/apt-repository-backup/

echo "Repository backups:"
ls -l /root/apt-repository-backup/
```

### Step 2: Disable the Enterprise repositories

```bash
mv /etc/apt/sources.list.d/pve-enterprise.sources \
   /etc/apt/sources.list.d/pve-enterprise.sources.disabled

mv /etc/apt/sources.list.d/ceph.sources \
   /etc/apt/sources.list.d/ceph.sources.disabled
```

### Step 3 Configure the no-subscription repositories

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

Create the Ceph Squid repository definition

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

# Static IP Configuration — Proxmox VE Cluster

This guide configures static IP addresses and Linux bridges on three Proxmox VE nodes running as VirtualBox VMs on an Ubuntu host.

## 1. Network Architecture

### VirtualBox Network Adapters

| Adapter   | VirtualBox network      | Purpose                        |
| --------- | ----------------------- | ------------------------------ |
| Adapter 1 | NAT                     | Internet access                |
| Adapter 2 | Host-only: `vboxnet0`   | Proxmox management             |
| Adapter 3 | Internal: `pve-cluster` | Corosync cluster communication |
| Adapter 4 | Internal: `pve-ceph`    | Ceph storage traffic           |

### IP Address Plan

| Node           | Management (`vmbr1`) | Cluster (`vmbr2`)  | Ceph (`vmbr3`)     |
| -------------- | -------------------- | ------------------ | ------------------ |
| `pve01-server` | `192.168.56.11/24`   | `172.20.100.11/24` | `172.20.101.11/24` |
| `pve02-server` | `192.168.56.12/24`   | `172.20.100.12/24` | `172.20.101.12/24` |
| `pve03-server` | `192.168.56.13/24`   | `172.20.100.13/24` | `172.20.101.13/24` |

### Network Summary

| Network    | Subnet            | Gateway    | Purpose                   |
| ---------- | ----------------- | ---------- | ------------------------- |
| NAT        | `10.0.2.0/24`     | `10.0.2.2` | Internet access           |
| Management | `192.168.56.0/24` | None       | Administration and web UI |
| Cluster    | `172.20.100.0/24` | None       | Corosync                  |
| Ceph       | `172.20.101.0/24` | None       | Ceph storage              |

**Routing principle:** Only `vmbr0` has a default gateway. The management, cluster, and Ceph bridges have no default gateway.

> **Important:** The NAT address `10.0.2.15/24` is retained on each node's independently configured VirtualBox NAT network. Management, cluster, and Ceph IP addresses must be unique within their respective networks.

---

## 2. Verify the Ubuntu Host Network

Run these commands on the **Ubuntu host**, not inside a Proxmox VM.

### 2.1 Check the Host-Only Interface

```bash
ip -br addr show vboxnet0
```

Expected address:

```text
vboxnet0    UP    192.168.56.1/24
```

### 2.2 Verify VirtualBox Adapter Mapping

Run the command for each VM:

```bash
VBoxManage showvminfo "pve01-server" --machinereadable |
grep -E '^(nic[1-4]|macaddress[1-4]|hostonlyadapter[1-4]|intnet[1-4]|bridgeadapter[1-4])='
```

```bash
VBoxManage showvminfo "pve02-server" --machinereadable |
grep -E '^(nic[1-4]|macaddress[1-4]|hostonlyadapter[1-4]|intnet[1-4]|bridgeadapter[1-4])='
```

```bash
VBoxManage showvminfo "pve03-server" --machinereadable |
grep -E '^(nic[1-4]|macaddress[1-4]|hostonlyadapter[1-4]|intnet[1-4]|bridgeadapter[1-4])='
```

Verify that each VM uses:

- Adapter 1: NAT
- Adapter 2: Host-only `vboxnet0`
- Adapter 3: Internal Network `pve-cluster`
- Adapter 4: Internal Network `pve-ceph`

---

## 3. Static IP Configuration — `pve01-server`

### 3.1 Inspect the Existing Configuration

Run inside `pve01-server` as root:

```bash
cat /etc/network/interfaces
ip -br addr
ip -br link
ip route
```

### 3.2 Back Up the Configuration

```bash
cp -a /etc/network/interfaces \
  "/etc/network/interfaces.backup-$(date +%Y%m%d-%H%M%S)"

cp -a /etc/hosts \
  "/etc/hosts.backup-$(date +%Y%m%d-%H%M%S)"
```

### 3.3 Confirmed Interface Mapping

| Interface | VirtualBox network     | Purpose         | Bridge  |
| --------- | ---------------------- | --------------- | ------- |
| `nic0`    | NAT                    | Internet access | `vmbr0` |
| `nic1`    | Host-only `vboxnet0`   | Management      | `vmbr1` |
| `nic2`    | Internal `pve-cluster` | Corosync        | `vmbr2` |
| `nic3`    | Internal `pve-ceph`    | Ceph storage    | `vmbr3` |

### 3.4 IP Address Assignment

| Bridge  | Physical virtual NIC | IP address         | Gateway    |
| ------- | -------------------- | ------------------ | ---------- |
| `vmbr0` | `nic0`               | `10.0.2.15/24`     | `10.0.2.2` |
| `vmbr1` | `nic1`               | `192.168.56.11/24` | None       |
| `vmbr2` | `nic2`               | `172.20.100.11/24` | None       |
| `vmbr3` | `nic3`               | `172.20.101.11/24` | None       |

### 3.5 Configure the Interfaces

Edit the file:

```bash
vi /etc/network/interfaces
```

Use this configuration:

```text
auto lo
iface lo inet loopback

# VirtualBox Adapter 1: NAT / Internet
iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.2.15/24
    gateway 10.0.2.2
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 2: Management
iface nic1 inet manual

auto vmbr1
iface vmbr1 inet static
    address 192.168.56.11/24
    bridge-ports nic1
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 3: Cluster / Corosync
iface nic2 inet manual

auto vmbr2
iface vmbr2 inet static
    address 172.20.100.11/24
    bridge-ports nic2
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 4: Ceph storage
iface nic3 inet manual

auto vmbr3
iface vmbr3 inet static
    address 172.20.101.11/24
    bridge-ports nic3
    bridge-stp off
    bridge-fd 0

source /etc/network/interfaces.d/*
```

### 3.6 Validate and Apply

```bash
ifquery --list
```

Expected configured interfaces include:

```text
lo
vmbr0
vmbr1
vmbr2
vmbr3
```

Apply the configuration from the VirtualBox console:

```bash
ifreload -a
```

Check the resulting addresses and routes:

```bash
ip -br addr
ip route
```

Test connectivity:

```bash
ping -c 3 192.168.56.1
ping -c 3 1.1.1.1
getent hosts download.proxmox.com
```

From the Ubuntu host, test management connectivity:

```bash
ping -c 3 192.168.56.11
```

Management URL:

`https://192.168.56.11:8006`

---

## 4. Static IP Configuration — `pve02-server`

### 4.1 Inspect and Back Up

Run inside `pve02-server`:

```bash
cat /etc/network/interfaces
ip -br addr
ip -br link
ip route
```

Back up the existing files:

```bash
cp -a /etc/network/interfaces \
  "/etc/network/interfaces.backup-$(date +%Y%m%d-%H%M%S)"

cp -a /etc/hosts \
  "/etc/hosts.backup-$(date +%Y%m%d-%H%M%S)"
```

### 4.2 Interface and IP Mapping

| Interface | VirtualBox network     | Bridge  | IP address         |
| --------- | ---------------------- | ------- | ------------------ |
| `nic0`    | NAT                    | `vmbr0` | `10.0.2.15/24`     |
| `nic1`    | Host-only `vboxnet0`   | `vmbr1` | `192.168.56.12/24` |
| `nic2`    | Internal `pve-cluster` | `vmbr2` | `172.20.100.12/24` |
| `nic3`    | Internal `pve-ceph`    | `vmbr3` | `172.20.101.12/24` |

### 4.3 Configure the Interfaces

Edit the file:

```bash
vi /etc/network/interfaces
```

Use this configuration:

```text
auto lo
iface lo inet loopback

# VirtualBox Adapter 1: NAT / Internet
iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.2.15/24
    gateway 10.0.2.2
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 2: Management
iface nic1 inet manual

auto vmbr1
iface vmbr1 inet static
    address 192.168.56.12/24
    bridge-ports nic1
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 3: Cluster / Corosync
iface nic2 inet manual

auto vmbr2
iface vmbr2 inet static
    address 172.20.100.12/24
    bridge-ports nic2
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 4: Ceph storage
iface nic3 inet manual

auto vmbr3
iface vmbr3 inet static
    address 172.20.101.12/24
    bridge-ports nic3
    bridge-stp off
    bridge-fd 0

source /etc/network/interfaces.d/*
```

### 4.4 Validate and Apply

```bash
ifquery --list
ifreload -a
ip -br addr
ip route
```

Test connectivity:

```bash
ping -c 3 192.168.56.1
ping -c 3 1.1.1.1
getent hosts download.proxmox.com
```

From the Ubuntu host:

```bash
ping -c 3 192.168.56.12
```

Management URL:

`https://192.168.56.12:8006`

---

## 5. Static IP Configuration — `pve03-server`

### 5.1 Inspect and Back Up

Run inside `pve03-server`:

```bash
cat /etc/network/interfaces
ip -br addr
ip -br link
ip route
```

Back up the existing files:

```bash
cp -a /etc/network/interfaces \
  "/etc/network/interfaces.backup-$(date +%Y%m%d-%H%M%S)"

cp -a /etc/hosts \
  "/etc/hosts.backup-$(date +%Y%m%d-%H%M%S)"
```

### 5.2 Interface and IP Mapping

| Interface | VirtualBox network     | Bridge  | IP address         |
| --------- | ---------------------- | ------- | ------------------ |
| `nic0`    | NAT                    | `vmbr0` | `10.0.2.15/24`     |
| `nic1`    | Host-only `vboxnet0`   | `vmbr1` | `192.168.56.13/24` |
| `nic2`    | Internal `pve-cluster` | `vmbr2` | `172.20.100.13/24` |
| `nic3`    | Internal `pve-ceph`    | `vmbr3` | `172.20.101.13/24` |

### 5.3 Configure the Interfaces

Edit the file:

```bash
vi /etc/network/interfaces
```

Use this configuration:

```text
auto lo
iface lo inet loopback

# VirtualBox Adapter 1: NAT / Internet
iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.2.15/24
    gateway 10.0.2.2
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 2: Management
iface nic1 inet manual

auto vmbr1
iface vmbr1 inet static
    address 192.168.56.13/24
    bridge-ports nic1
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 3: Cluster / Corosync
iface nic2 inet manual

auto vmbr2
iface vmbr2 inet static
    address 172.20.100.13/24
    bridge-ports nic2
    bridge-stp off
    bridge-fd 0

# VirtualBox Adapter 4: Ceph storage
iface nic3 inet manual

auto vmbr3
iface vmbr3 inet static
    address 172.20.101.13/24
    bridge-ports nic3
    bridge-stp off
    bridge-fd 0

source /etc/network/interfaces.d/*
```

### 5.4 Validate and Apply

```bash
ifquery --list
ifreload -a
ip -br addr
ip route
```

Test connectivity:

```bash
ping -c 3 192.168.56.1
ping -c 3 1.1.1.1
getent hosts download.proxmox.com
```

From the Ubuntu host:

```bash
ping -c 3 192.168.56.13
```

Management URL:

`https://192.168.56.13:8006`

---

## 6. Final Three-Node Network Validation

### 6.1 Verify the Addresses

Run on each node:

```bash
ip -br addr
ip route
```

Confirm the following addresses are assigned:

| Node           | `vmbr0`        | `vmbr1`            | `vmbr2`            | `vmbr3`            |
| -------------- | -------------- | ------------------ | ------------------ | ------------------ |
| `pve01-server` | `10.0.2.15/24` | `192.168.56.11/24` | `172.20.100.11/24` | `172.20.101.11/24` |
| `pve02-server` | `10.0.2.15/24` | `192.168.56.12/24` | `172.20.100.12/24` | `172.20.101.12/24` |
| `pve03-server` | `10.0.2.15/24` | `192.168.56.13/24` | `172.20.100.13/24` | `172.20.101.13/24` |

### 6.2 Test Management Connectivity

From the Ubuntu host:

```bash
ping -c 3 192.168.56.11
ping -c 3 192.168.56.12
ping -c 3 192.168.56.13
```

### 6.3 Test Cluster Network Connectivity

From `pve01-server`:

```bash
ping -c 3 172.20.100.12
ping -c 3 172.20.100.13
```

From `pve02-server`:

```bash
ping -c 3 172.20.100.11
ping -c 3 172.20.100.13
```

From `pve03-server`:

```bash
ping -c 3 172.20.100.11
ping -c 3 172.20.100.12
```

### 6.4 Test Ceph Network Connectivity

From `pve01-server`:

```bash
ping -c 3 172.20.101.12
ping -c 3 172.20.101.13
```

From `pve02-server`:

```bash
ping -c 3 172.20.101.11
ping -c 3 172.20.101.13
```

From `pve03-server`:

```bash
ping -c 3 172.20.101.11
ping -c 3 172.20.101.12
```

### 6.5 Verify the Default Route

Run on every node:

```bash
ip route
```

Confirm that the only default route is through:

```text
default via 10.0.2.2 dev vmbr0
```

### 6.6 Verify Management Web Interfaces

Open these URLs from the Ubuntu host:

- `https://192.168.56.11:8006` — `pve01-server`
- `https://192.168.56.12:8006` — `pve02-server`
- `https://192.168.56.13:8006` — `pve03-server`

## 7. Important Notes

1. Run `VBoxManage` commands on the Ubuntu host, not inside a Proxmox guest.
2. Run the `/etc/network/interfaces` configuration and `ifreload -a` commands inside the corresponding Proxmox guest.
3. Use the VirtualBox console while applying network changes to avoid losing access.
4. Do not assign default gateways to `vmbr1`, `vmbr2`, or `vmbr3`.
5. The cluster and Ceph networks must be connected to the correct VirtualBox internal networks on all three nodes.
6. Do not create the cluster or initialize Ceph until network connectivity and hostname resolution have been verified.
7. Because all three nodes run on one physical host, this is a nested virtualization lab and does not provide physical-host high availability.

# Prepare the Proxmox Cluster

Once all three networks pass connectivity tests, proceed in this order:

1. `Hostname resolution:` Configure consistent /etc/hosts entries on all three nodes.
2. `Time synchronization:` Verify that the nodes have synchronized clocks.
3. `Cluster creation:` Create the Proxmox cluster on pve01-server.
4. `Node joining:` Join pve02-server and pve03-server.
5. `Cluster validation:` Check quorum and Corosync status.
6. `Ceph deployment:` Configure Ceph only after the cluster is healthy.

Target architecture

| Node           | Management      | Cluster / Corosync | Ceph            |
| -------------- | --------------- | ------------------ | --------------- |
| `pve01-server` | `192.168.56.11` | `172.20.100.11`    | `172.20.101.11` |
| `pve02-server` | `192.168.56.12` | `172.20.100.12`    | `172.20.101.12` |
| `pve03-server` | `192.168.56.13` | `172.20.100.13`    | `172.20.101.13` |

> We will use the dedicated cluster network (172.20.100.0/24) for Corosync and reserve 172.20.101.0/24 for Ceph traffic.

## Step 1 — Configure `/etc/hosts` on all nodes

Run on **each Proxmox node**:

```bash
hostnamectl
hostname -s
cat /etc/hosts
```

Back up the current file before editing:

```bash
cp -a /etc/hosts "/etc/hosts.bak.$(date +%F-%H%M%S)"
nano /etc/hosts
```

Ensure the file contains these consistent mappings. Preserve any required IPv6 entries and other valid local entries, but remove or correct conflicting mappings for `pve01`, `pve02`, and `pve03`:

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

## Step 3 — Verify the pve01 certificate before joining nodes

Run on **pve01**:

```bash
hostname -s
hostname -f
getent hosts pve01.example.local
openssl x509 \
  -in /etc/pve/local/pve-ssl.pem \
  -noout -subject -ext subjectAltName
openssl x509 \
  -in /etc/pve/local/pve-ssl.pem \
  -noout -fingerprint -sha256
```

Record the actual SHA-256 fingerprint printed by the command. Do not assume it will remain the same if the certificate is regenerated.

From **pve02** and then **pve03**, verify name resolution and the HTTPS API endpoint:

```bash
getent hosts pve01.example.local
curl -v https://pve01.example.local:8006/api2/json/version
```

The request should complete without a certificate hostname/verification error and return Proxmox version JSON. If it fails certificate validation, stop and resolve the hostname/certificate issue; do not disable TLS verification as a workaround.

## Step 4 — Create the cluster on pve01

First confirm that `pve01` is standalone and is not already part of another cluster:

```bash
pvecm status
pvecm nodes
```

If this is a fresh standalone node and no cluster configuration exists, create the cluster on **pve01**:

```bash
pvecm create homelab-cluster --link0 172.20.100.11
```

Validate before joining any other node:

```bash
pvecm status
pvecm nodes
systemctl status corosync --no-pager
```

Expected initial state: `pve01` is the only cluster node and the cluster is quorate. If `pve01` already belongs to a cluster, do not run `pvecm create` again; inspect the current cluster state first.

## Step 5 — Join pve02

Run the join command **on pve02**, not on pve01. Use the joining node's own Corosync address for `--link0`:

```bash
pvecm add pve01.example.local \
  --link0 172.20.100.12 \
  --fingerprint 'REPLACE_WITH_THE_SHA256_FINGERPRINT_FROM_PVE01'
```

Replace the placeholder with the fingerprint recorded in Step 3. The fingerprint is expected to be supplied as colon-separated hexadecimal, for example `AA:BB:...`; use the actual value from pve01, not the example.

Do **not** use `--link0 172.20.100.11` on pve02: `.11` is pve01's Corosync address, whereas pve02 must advertise its own address, `172.20.100.12`. Prefer the FQDN plus fingerprint approach rather than joining by IP when certificate hostname verification has been an issue.

After the join completes, verify on **pve01**:

```bash
pvecm status
pvecm nodes
corosync-cfgtool -s
```

Do not join pve03 until pve02 appears in the membership list and the cluster is healthy.

## Step 6 — Join pve03

Run the following on **pve03** only after pve02 has joined successfully and the cluster is healthy:

```bash
pvecm add pve01.example.local \
  --link0 172.20.100.13 \
  --fingerprint 'REPLACE_WITH_THE_SHA256_FINGERPRINT_FROM_PVE01'
```

Again, replace the placeholder with the actual pve01 certificate fingerprint. The `--link0` value must be pve03's own Corosync address, `172.20.100.13`.

## Step 7 — Validate the three-node cluster

Run on **pve01**:

```bash
pvecm status
pvecm nodes
corosync-cfgtool -s
systemctl is-active corosync
```

Confirm all three nodes (`pve01`, `pve02`, and `pve03`) are listed, Corosync links are healthy, and the cluster is quorate. Only then proceed to Ceph deployment.

## Troubleshooting notes

- `hostname lookup 'pve03' failed`: fix `/etc/hosts` and ensure `getent hosts pve03` returns `192.168.56.13` before retrying.
- `hostname verification failed`: verify that the FQDN resolves to pve01's management IP and that the certificate SAN includes the hostname used in the URL. A matching fingerprint alone does not prove hostname verification will pass.
- `pvecm status` says the Corosync configuration does not exist: this is expected on a standalone node before cluster creation. It is not proof that the cluster is already created.
- If a join attempt fails, inspect the exact error and the current state of both nodes before retrying. Do not repeatedly rerun `pvecm add` blindly.
- Never initialize the Proxmox system disk as a Ceph OSD. Configure Ceph only after all three nodes are joined and healthy.
- Because all three nodes run as VirtualBox guests on one physical host, this is a nested virtualization lab and does not provide physical-host high availability.

### Cluster health summary

| Validation                    | pve01             | pve02             | pve03             | Status     |
| ----------------------------- | ----------------- | ----------------- | ----------------- | ---------- |
| Cluster name                  | `homelab-cluster` | `homelab-cluster` | `homelab-cluster` | Consistent |
| Cluster configuration version | 3                 | 3                 | 3                 | Consistent |
| Nodes detected                | 3                 | 3                 | 3                 | Healthy    |
| Expected votes                | 3                 | 3                 | 3                 | Correct    |
| Total votes                   | 3                 | 3                 | 3                 | All voting |
| Quorum                        | Yes               | Yes               | Yes               | Quorate    |
| Transport                     | knet              | knet              | knet              | Consistent |
| Secure authentication         | On                | On                | On                | Enabled    |

### Validate cluster communication

```bash
corosync-cfgtool -s
```

```bash
systemctl is-active pve-cluster
systemctl is-active corosync
systemctl is-active pvedaemon
systemctl is-active pveproxy
```

# OSD in Ceph

OSD stands for `Object Storage Daemon`. In Ceph, an OSD is a service that manages data stored on a storage device and participates in distributing, replicating, recovering, and rebalancing data across the cluster. In simple terms, an OSD allows Ceph to use disks across multiple servers as a distributed storage system.

```bash
             Proxmox + Ceph Cluster
        ┌────────────┬────────────┬────────────┐
        │   pve01    │   pve02    │   pve03    │
        │            │            │            │
        │  /dev/sdb  │  /dev/sdb  │  /dev/sdb  │
        │     │      │     │      │     │      │
        │   OSD.0    │   OSD.1    │   OSD.2    │
        └─────┬──────┴─────┬──────┴─────┬──────┘
              │            │            │
              └────────────┼────────────┘
                   Ceph Network
```

### Important Ceph components - An OSD is only one component of Ceph.

| Component | Full name                                     | Responsibility                                                                |
| --------- | --------------------------------------------- | ----------------------------------------------------------------------------- |
| OSD       | Object Storage Daemon                         | Stores data and handles replication, recovery, and rebalancing                |
| MON       | Monitor                                       | Maintains cluster maps and participates in cluster quorum                     |
| MGR       | Manager                                       | Provides monitoring, management modules, and operational metrics              |
| MDS       | Metadata Server                               | Provides metadata services for CephFS                                         |
| CRUSH     | Controlled Replication Under Scalable Hashing | Determines data placement across eligible storage devices and failure domains |

### Create one dedicated OSD disk per node

1. Ensure poweroff of all VMs
2. Make disk

```bash
set -euo pipefail

BASE="$HOME/VirtualBox VMs"

VBoxManage createmedium disk \
    --filename "$BASE/pve01-server/pve01-osd01.vdi" \
    --size 10240 \
    --format VDI \
    --variant Standard

VBoxManage createmedium disk \
    --filename "$BASE/pve02-server/pve02-osd01.vdi" \
    --size 10240 \
    --format VDI \
    --variant Standard

VBoxManage createmedium disk \
    --filename "$BASE/pve03-server/pve03-osd01.vdi" \
    --size 10240 \
    --format VDI \
    --variant Standard
```

```bash
VBoxManage list hdds
```

```bash
for disk in \
    "$HOME/VirtualBox VMs/pve01-server/pve01-osd01.vdi" \
    "$HOME/VirtualBox VMs/pve02-server/pve02-osd01.vdi" \
    "$HOME/VirtualBox VMs/pve03-server/pve03-osd01.vdi"
do
    echo
    echo "========== $disk =========="
    VBoxManage showmediuminfo disk "$disk"
done
```

### Attach the OSD disks

```bash
set -euo pipefail

BASE="$HOME/VirtualBox VMs"

# pve01
VBoxManage storagectl "pve01-server" \
    --name "SATA" \
    --portcount 2

VBoxManage storageattach "pve01-server" \
    --storagectl "SATA" \
    --port 1 \
    --device 0 \
    --type hdd \
    --medium "$BASE/pve01-server/pve01-osd01.vdi"

# pve02
VBoxManage storagectl "pve02-server" \
    --name "SATA" \
    --portcount 2

VBoxManage storageattach "pve02-server" \
    --storagectl "SATA" \
    --port 1 \
    --device 0 \
    --type hdd \
    --medium "$BASE/pve02-server/pve02-osd01.vdi"

# pve03
VBoxManage storagectl "pve03-server" \
    --name "SATA" \
    --portcount 2

VBoxManage storageattach "pve03-server" \
    --storagectl "SATA" \
    --port 1 \
    --device 0 \
    --type hdd \
    --medium "$BASE/pve03-server/pve03-osd01.vdi"

echo "All three OSD disks attached."
```

```bash
for vm in pve01-server pve02-server pve03-server; do
    echo
    echo "========== $vm =========="
    VBoxManage showvminfo "$vm" --machinereadable |
        grep -E '^(storagecontrollerportcount|SATA-[0-9]+-[0-9]+)'
done
```

```bash
VBoxManage startvm "pve01-server" --type headless
VBoxManage startvm "pve02-server" --type headless
VBoxManage startvm "pve03-server" --type headless
```

| Proxmox node | System disk        | Dedicated OSD disk | Status   |
| ------------ | ------------------ | ------------------ | -------- |
| `pve01`      | `pve01-server.vdi` | `pve01-osd01.vdi`  | Attached |
| `pve02`      | `pve02-server.vdi` | `pve02-osd01.vdi`  | Attached |
| `pve03`      | `pve03-server.vdi` | `pve03-osd01.vdi`  | Attached |

### Install Ceph from dashboard

### OSD creation

```bash
pveceph osd create /dev/sdb
```

```bash
ceph -s
ceph osd tree
ceph osd status
```

```bash
ceph -s
ceph osd tree
ceph osd status
ceph df
ceph osd pool ls detail
```

### RBD (RADOS Block Device)

It's Ceph's distributed block-storage technology. It allows Proxmox VE to use storage distributed across your Ceph cluster as virtual disks for VMs and containers.

```bash
             Proxmox VE Cluster
             homelab-cluster
                     |
          +----------+----------+
          |          |          |
        pve01      pve02      pve03
          |          |          |
        osd.0      osd.1      osd.2
          |          |          |
          +----------+----------+
                     |
                Ceph Cluster
                     |
              RBD Pool: vmdata
                     |
              RBD Block Images
                     |
             Proxmox Virtual Disks
                     |
                 Virtual Machines
```

> How RBD works: Think of RBD as a virtual hard disk distributed across multiple Ceph OSDs.

#### RBD vs other storage technologies

| Technology | Storage type              | Typical use                  |
| ---------- | ------------------------- | ---------------------------- |
| RBD        | Distributed block storage | VM disks                     |
| CephFS     | Distributed filesystem    | Shared files and directories |
| RGW        | Object storage            | S3-compatible applications   |
| LVM        | Block storage             | Local VM disks               |
| NFS        | Network filesystem        | Shared files and backups     |
