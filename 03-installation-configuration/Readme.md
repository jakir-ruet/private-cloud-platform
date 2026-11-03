# Module 03 — Proxmox VE Installation & Configuration

## 1. Overview

Proxmox Virtual Environment (VE) is a Debian-based virtualization platform that integrates:

- KVM/QEMU for virtual machines.
- LXC for system containers.
- Linux networking for bridges, VLANs, and routing.
- Storage plugins for local and shared storage.
- Ceph integration for distributed storage.
- Cluster services for centralized management.
- Web-based administration and REST API access.

A reliable Proxmox deployment requires correct configuration of the operating system, networking, DNS, time synchronization, repositories, storage, and cluster communication.

### Learning objectives

By the end of this module, you should be able to:

1. Explain the Proxmox VE installation process.
2. Understand installation prerequisites.
3. Identify Proxmox system partitions and storage layout.
4. Configure hostnames and DNS resolution.
5. Understand Proxmox repository configuration.
6. Inspect management networking and Linux bridges.
7. Understand time synchronization.
8. Inspect SSH and administrative access.
9. Validate essential Proxmox services.
10. Perform post-installation health checks.
11. Document a repeatable Proxmox node deployment procedure.

---

# 2. Proxmox VE Installation Architecture

## 2.1 Installation architecture

Proxmox VE is installed directly on a physical server or supported virtual machine.

```text
                Physical Server / Virtual Machine
                            │
                            ▼
                     Proxmox VE Installer
                            │
                            ▼
                   Debian-based Linux OS
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       KVM/QEMU          Linux Network      Storage
          │                 │                 │
          ▼                 ▼                 ▼
     Virtual Machines   Linux Bridges     LVM / ZFS
     and Devices        VLANs              Ceph
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                            ▼
                    Proxmox VE Services
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
          Web UI/API     Cluster       HA Services
                         Services
```

### Prepare VM (Confirmed VM specification)

| VM name               |                `pve01-server` |                `pve02-server` |                `pve03-server` |  Total |
| --------------------- | ----------------------------: | ----------------------------: | ----------------------------: | -----: |
| vCPU                  |                             2 |                             2 |                             2 | 6 vCPU |
| RAM                   |                         3 GiB |                         3 GiB |                         3 GiB |  9 GiB |
| System disk           |                         45 GB |                         45 GB |                         45 GB | 135 GB |
| Initial Ceph OSD disk |                         10 GB |                         10 GB |                         10 GB |  30 GB |
| Network adapters (4)  | NAT, host-only, cluster, Ceph | NAT, host-only, cluster, Ceph | NAT, host-only, cluster, Ceph |     12 |
| Hypervisor            |                    Proxmox VE |                    Proxmox VE |                    Proxmox VE |      — |
| Installation ISO      |   `/opt/proxmox-ve_9.2-1.iso` |   `/opt/proxmox-ve_9.2-1.iso` |   `/opt/proxmox-ve_9.2-1.iso` |      — |

### Node IP addressing plan

| Adapter   | VirtualBox Network                 | Network Purpose                     | Subnet            | PVE01              | PVE02              | PVE03              |
| --------- | ---------------------------------- | ----------------------------------- | ----------------- | ------------------ | ------------------ | ------------------ |
| Adapter 1 | NAT                                | Internet access                     | VirtualBox DHCP   | DHCP               | DHCP               | DHCP               |
| Adapter 2 | Host-only (`vboxnet0`)             | Management and administration       | `192.168.56.0/24` | `192.168.56.11/24` | `192.168.56.12/24` | `192.168.56.13/24` |
| Adapter 3 | Internal Network (`pve-cluster`)   | Corosync cluster communication      | `172.20.100.0/24` | `172.20.100.11/24` | `172.20.100.12/24` | `172.20.100.13/24` |
| Adapter 4 | Internal Network (`pve-ceph`)      | Ceph public and replication traffic | `172.20.101.0/24` | `172.20.101.11/24` | `172.20.101.12/24` | `172.20.101.13/24` |
| Adapter 5 | Internal Network (`pve-migration`) | VM live migration traffic           | `172.20.102.0/24` | `172.20.102.11/24` | `172.20.102.12/24` | `172.20.102.13/24` |

```bash
                    Ubuntu 24.04 Host
                    VirtualBox 7.2
                           |
       +-------------------+----------------------+
       |                   |                      |
   NAT Network         Host-only           Internal Networks
       |              vboxnet0                    |
       |                   |                      |
       |            Management Network            |
       |              192.168.56.0/24             |
       |                   |                      |
       +-------------------+----------------------+
                           |
                +----------+----------+
                |                     |
           Proxmox VE Cluster: homelab-cluster
                |                     |
       +--------+---------+-----------+
       |                  |           |
    PVE01              PVE02        PVE03
       |                  |           |
       +------------------+-----------+
                          |
        +-----------------+------------------+
        |                 |                  |
   Management          Cluster             Ceph
   192.168.56.0/24     172.20.100.0/24     172.20.101.0/24
        |                 |                  |
        +-----------------+------------------+
                          |
                    Migration Network
                      172.20.102.0/24
```

| Network    | Main Component            | Responsibility                                                |
| ---------- | ------------------------- | ------------------------------------------------------------- |
| Management | Proxmox Web UI, SSH, API  | Administrative access and node management                     |
| Cluster    | Corosync / KNET           | Cluster membership, messaging, and quorum communication       |
| Ceph       | MON, OSD, MGR             | Ceph client/public and storage replication traffic            |
| Migration  | Proxmox migration service | VM memory and, where applicable, local disk migration         |
| Internet   | NAT                       | Package updates, external connectivity, and repository access |

Important Configuration Rules

1. Default gateway: Use the NAT interface for the default route. Do not configure separate default gateways on the internal networks.
2. Management: Keep the host-only network reachable from the Ubuntu host.
3. Cluster: Use the dedicated cluster network for Corosync/KNET communication.
4. Ceph: Use the dedicated Ceph network for Ceph traffic. A single Ceph network can carry both public and replication traffic in this lab.
5. Migration: Configure the dedicated migration network in Proxmox migration settings.
6. Interface mapping: Identify interfaces using MAC addresses and ip -br link; do not assume interface names such as enp0s3 or enp0s8.
7. Network isolation: Internal networks should not have unnecessary gateways or external routing.

## 2.2 Installation requirements

Before installing Proxmox VE, evaluate the following:

| Component          | Requirement / Consideration                                  |
| ------------------ | ------------------------------------------------------------ |
| CPU                | 64-bit processor with hardware virtualization support        |
| Virtualization     | Intel VT-x or AMD-V                                          |
| Memory             | Sufficient RAM for the hypervisor and planned workloads      |
| Storage            | Dedicated installation disk and appropriate workload storage |
| Network            | Reliable management network connectivity                     |
| DNS                | Correct hostname and name resolution                         |
| Time               | Reliable NTP synchronization                                 |
| Firmware           | Virtualization enabled in BIOS/UEFI                          |
| Installation media | Compatible Proxmox VE ISO                                    |
| Backup             | Existing data protected before installation                  |

For a production environment, storage redundancy, network redundancy, hardware compatibility, and independent failure domains must also be considered.

## 2.3 Nested virtualization considerations

Your lab uses:

```text
Ubuntu 24.04.5 LTS
       │
       ▼
VirtualBox
       │
       ├── pve01-server
       ├── pve02-server
       └── pve03-server
```

Each Proxmox node is itself a virtual machine.

The architecture is:

```text
Physical Laptop
     │
     ▼
Ubuntu Linux
     │
     ▼
VirtualBox Hypervisor
     │
     ├── Proxmox VE Node 01
     │       └── KVM/QEMU
     │              └── Nested VMs
     │
     ├── Proxmox VE Node 02
     │       └── KVM/QEMU
     │              └── Nested VMs
     │
     └── Proxmox VE Node 03
             └── KVM/QEMU
                    └── Nested VMs
```

Nested virtualization introduces additional overhead and performance constraints.

Your three Proxmox nodes share one physical host. Therefore, they are useful for learning cluster operations but do not represent independent physical failure domains.

---

# 3. Proxmox VE Installation Workflow

For a new node, follow this deployment sequence.

```text
Hardware Assessment
        │
        ▼
BIOS/UEFI Configuration
        │
        ▼
Download Proxmox VE ISO
        │
        ▼
Create Installation Media
        │
        ▼
Boot Installer
        │
        ▼
Select Installation Disk
        │
        ▼
Configure Locale and Time Zone
        │
        ▼
Configure Management Network
        │
        ▼
Set Administrator Password
        │
        ▼
Install Proxmox VE
        │
        ▼
Reboot
        │
        ▼
Initial Node Validation
        │
        ▼
Repository and Update Configuration
        │
        ▼
Storage and Network Validation
        │
        ▼
Ready for Cluster Integration
```

## 3.1 Installation disk selection

The installation target must be selected carefully.

Possible installation filesystems include:

- ext4
- XFS
- ZFS

The installer offers different storage layouts depending on the selected filesystem and installation options.

**Important:** Installing Proxmox VE on a disk normally destroys existing data on the selected installation target.

For your lab, do not rerun the installer against any existing Proxmox VM unless you intentionally plan to rebuild that node and have verified all relevant backups and dependencies.

## 3.2 Installation network configuration

During installation, configure:

- Management interface.
- Hostname (FQDN).
- IP address.
- Network mask.
- Gateway.
- DNS server.

Example only:

| Setting       | Example                            |
| ------------- | ---------------------------------- |
| Hostname      | pve01.example.local                |
| Management IP | 192.168.56.11                      |
| Netmask       | 255.255.255.0                      |
| Gateway       | Appropriate lab gateway            |
| DNS           | Appropriate reachable DNS resolver |

These are illustrative values. Verify your actual lab configuration before making changes.

---

# 4. Initial Node Configuration

After installation, validate the node identity and operating-system configuration.

## 4.1 Hostname

Inspect:

```bash
hostname
hostname -f
hostnamectl
```

Inspect hostname configuration:

```bash
cat /etc/hostname
cat /etc/hosts
```

A cluster node should have a stable hostname and consistent name resolution.

Example `/etc/hosts` structure:

```text
127.0.0.1       localhost

192.168.56.11   pve01.example.local pve01
192.168.56.12   pve02.example.local pve02
192.168.56.13   pve03.example.local pve03
```

This is an example only. Use the actual IP addresses and names from your environment.

**Do not replace your current `/etc/hosts` file with this example.**

## 4.2 DNS resolution

Inspect resolver configuration:

```bash
cat /etc/resolv.conf
resolvectl status
```

Test node resolution:

```bash
getent hosts pve01
getent hosts pve02
getent hosts pve03
```

Test DNS resolution for an external hostname:

```bash
getent hosts deb.debian.org
```

Validate that:

- Node names resolve consistently.
- Management addresses are correct.
- DNS servers are reachable.
- No unexpected hostname-to-IP mapping exists.

## 4.3 Network identity

Inspect:

```bash
hostname -I
ip -br addr
ip route
```

Record the management interface and its IP address.

---

# 5. Proxmox Repository Configuration

Proxmox VE uses APT repositories to distribute updates.

Repository configuration is important for:

- Proxmox VE updates.
- Security patches.
- Kernel updates.
- QEMU/KVM updates.
- Ceph packages.
- Debian package dependencies.

## 5.1 Repository types

The primary repository choices include:

| Repository      | Purpose                                                            |
| --------------- | ------------------------------------------------------------------ |
| Enterprise      | Enterprise repository for eligible subscriptions                   |
| No-Subscription | Repository intended for non-enterprise subscription environments   |
| Test            | Testing repository; not recommended for routine production updates |

Repository availability and supported configuration depend on the installed Proxmox VE release.

## 5.2 Inspect configured repositories

Run:

```bash
echo "===== APT SOURCES ====="
find /etc/apt/sources.list.d -maxdepth 1 -type f -print

echo "===== MAIN SOURCES ====="
cat /etc/apt/sources.list

echo "===== SOURCE FILE CONTENTS ====="
grep -R -nE '^(deb |Types:|URIs:|Suites:|Components:|Enabled:)' \
    /etc/apt/sources.list \
    /etc/apt/sources.list.d/ 2>/dev/null
```

## 5.3 Enterprise repository authentication

If an installation is configured to use the Proxmox Enterprise repository without an eligible subscription, APT may return an HTTP 401 Unauthorized error.

This does not automatically mean that the Proxmox installation is broken.

Inspect the repository configuration before changing it.

## 5.4 No-Subscription repository

For a non-subscription lab, the No-Subscription repository is commonly used.

Before configuring it:

1. Confirm the installed Proxmox VE release.
2. Confirm the matching Debian release.
3. Inspect existing enterprise repository files.
4. Ensure the repository is not duplicated.
5. Use the official repository configuration instructions for the installed release.

Do not blindly append repository definitions. Duplicate or incompatible repository entries can create package-management problems.

Official documentation:

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/wiki/Package_Repositories

## 5.5 Package update workflow

Once repositories have been validated:

```bash
apt update
```

Review available upgrades:

```bash
apt list --upgradable
```

For a maintenance window, use the package-management procedure appropriate to your installed release.

Do not initiate a kernel or major platform upgrade while performing a basic discovery exercise.

---

# 6. Management Network Configuration

Proxmox VE uses Linux networking.

Common components include:

- Physical network interfaces.
- Linux bridges.
- VLAN interfaces.
- Bond interfaces.
- Linux routing.
- Firewall integration.

## 6.1 Linux bridge architecture

A Linux bridge connects virtual machine interfaces to a physical or logical network.

```text
                 Physical Network
                       │
                       ▼
                 Physical NIC
                       │
                       ▼
                   vmbr0
                 Linux Bridge
                  │       │
                  │       │
                  ▼       ▼
                VM 100  VM 101
                vNIC    vNIC
```

A typical bridge configuration may resemble:

```text
auto lo
iface lo inet loopback

iface enp1s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.56.11/24
    gateway 192.168.56.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

This is an illustrative configuration, not a configuration to paste into your node.

Your actual interfaces, bridge names, addresses, and gateways may differ.

## 6.2 Inspect network interfaces

```bash
ip -br link
ip -br addr
```

## 6.3 Inspect bridge configuration

```bash
bridge link
bridge vlan show
```

## 6.4 Inspect Proxmox network configuration

```bash
cat /etc/network/interfaces
```

If additional configuration files are referenced:

```bash
find /etc/network/interfaces.d -maxdepth 1 -type f -print
```

## 6.5 Network configuration safety

Network changes can immediately disconnect a Proxmox node.

Before changing network configuration:

1. Confirm the management interface.
2. Confirm the current management IP.
3. Confirm the gateway and routing.
4. Ensure console access through VirtualBox is available.
5. Back up the existing network configuration.
6. Understand the effect on Corosync and Ceph networks.
7. Apply changes during a controlled maintenance window.

Do not restart networking or modify bridge configuration during this discovery lab.

---

# 7. Time Synchronization

Time synchronization is important for:

- Cluster operations.
- Authentication.
- Certificate validity.
- Logs and event correlation.
- Distributed storage.
- Monitoring and troubleshooting.

Inspect time:

```bash
date
timedatectl
```

Inspect available time synchronization services:

```bash
systemctl status chrony
systemctl status systemd-timesyncd
```

Depending on the installed configuration, one of these services may not be installed or active.

If Chrony is installed:

```bash
chronyc tracking
chronyc sources -v
```

Check that the nodes maintain consistent time.

Do not install or enable another time service without checking the existing configuration.

---

# 8. SSH and Administrative Access

SSH is commonly used for administration and automation.

Inspect SSH service:

```bash
systemctl status ssh
```

Inspect listening sockets:

```bash
ss -lntp
```

Inspect SSH configuration:

```bash
sshd -T | grep -E \
'permitrootlogin|passwordauthentication|pubkeyauthentication'
```

For production environments, use an appropriate SSH key-management and access-control strategy.

Recommended practices:

- Use SSH keys.
- Restrict administrative access.
- Avoid unnecessary password-based remote access.
- Maintain secure access to the Proxmox web interface.
- Protect private keys.
- Audit privileged access.

Do not change SSH authentication settings during the read-only lab.

---

# 9. Proxmox Core Services

Proxmox VE uses several system services.

| Service      | Responsibility                           |
| ------------ | ---------------------------------------- |
| pveproxy     | Web interface and API proxy              |
| pvedaemon    | Privileged management operations         |
| pvestatd     | Collects resource and status information |
| pve-cluster  | Proxmox cluster filesystem service       |
| corosync     | Cluster communication and membership     |
| pve-ha-crm   | HA Cluster Resource Manager              |
| pve-ha-lrm   | HA Local Resource Manager                |
| pvescheduler | Scheduled task execution                 |

Inspect services:

```bash
systemctl status pveproxy
systemctl status pvedaemon
systemctl status pvestatd
systemctl status pve-cluster
systemctl status corosync
```

Inspect HA services:

```bash
systemctl status pve-ha-crm
systemctl status pve-ha-lrm
```

Not every service will necessarily be active in every deployment or at every moment.

## 9.1 Service logs

Example:

```bash
journalctl -u pveproxy --since "1 hour ago"
journalctl -u pvedaemon --since "1 hour ago"
journalctl -u pvestatd --since "1 hour ago"
journalctl -u corosync --since "1 hour ago"
```

Use logs to investigate errors and service behavior.

---

# 10. Proxmox Web Interface

The Proxmox web interface normally uses HTTPS on TCP port 8006.

Example:

```text
https://<proxmox-node-ip>:8006
```

The interface provides:

- Datacenter configuration.
- Node administration.
- VM and container management.
- Storage administration.
- Network configuration.
- Cluster management.
- HA configuration.
- Firewall administration.
- Backup and restore operations.
- Task history and logs.

Check the listener:

```bash
ss -lntp | grep 8006
```

Test local HTTPS connectivity:

```bash
curl -k -I https://127.0.0.1:8006
```

The `-k` option skips certificate verification. Use it only for this diagnostic check; it is not a recommended general TLS validation practice.

---

# 11. Initial Storage Validation

After installation, validate the storage configuration.

```bash
pvesm status
```

Inspect the storage configuration:

```bash
cat /etc/pve/storage.cfg
```

Inspect block devices:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Inspect filesystem capacity:

```bash
df -hT
```

For LVM:

```bash
pvs
vgs
lvs -a
```

For Ceph:

```bash
ceph -s
ceph health detail
ceph osd tree
```

Do not initialize, format, or reconfigure storage as part of the installation validation exercise.

---

# 12. Post-Installation Validation

A newly installed node should pass basic operating-system, network, service, and storage checks before cluster integration.

## 12.1 Operating system

```bash
hostnamectl
pveversion -v
uname -r
```

## 12.2 CPU and memory

```bash
lscpu
free -h
```

## 12.3 Network

```bash
ip -br addr
ip route
getent hosts "$(hostname)"
```

## 12.4 Storage

```bash
pvesm status
lsblk
df -hT
```

## 12.5 Services

```bash
systemctl is-active pveproxy
systemctl is-active pvedaemon
systemctl is-active pvestatd
systemctl is-active pve-cluster
```

## 12.6 Cluster

For an already-clustered node:

```bash
pvecm status
pvecm nodes
```

Do not run cluster-creation commands against your existing cluster.

---

# 13. Hands-on Lab — Existing Environment Discovery

This lab is specifically designed for your existing three-node Proxmox environment.

**Execution location:** PVE01, PVE02, and PVE03 as appropriate.

**Execution policy:** Read-only inspection. No installation, reinstallation, repository modifications, network changes, service restarts, or cluster reconfiguration.

## Lab 01 — Node identity

Run on each node:

```bash
echo "===== HOSTNAME ====="
hostname

echo "===== FQDN ====="
hostname -f

echo "===== HOSTNAME CONFIGURATION ====="
hostnamectl

echo "===== HOST IP ADDRESSES ====="
hostname -I

echo "===== HOSTS FILE ====="
cat /etc/hosts
```

## Lab 02 — Proxmox version

```bash
pveversion -v
uname -r
```

## Lab 03 — Repository configuration

```bash
echo "===== APT SOURCES ====="
find /etc/apt/sources.list.d -maxdepth 1 -type f -print

echo "===== REPOSITORY DEFINITIONS ====="
grep -R -nE '^(deb |Types:|URIs:|Suites:|Components:|Enabled:)' \
    /etc/apt/sources.list \
    /etc/apt/sources.list.d/ 2>/dev/null
```

## Lab 04 — Network configuration

```bash
echo "===== NETWORK INTERFACES ====="
ip -br addr

echo "===== ROUTING ====="
ip route

echo "===== NETWORK CONFIGURATION ====="
cat /etc/network/interfaces

echo "===== BRIDGES ====="
bridge link
```

## Lab 05 — Time synchronization

```bash
date
timedatectl
systemctl status chrony --no-pager
systemctl status systemd-timesyncd --no-pager
```

## Lab 06 — SSH

```bash
systemctl status ssh --no-pager
ss -lntp
```

## Lab 07 — Core services

```bash
for service in \
    pveproxy \
    pvedaemon \
    pvestatd \
    pve-cluster \
    corosync
do
    echo "===== $service ====="
    systemctl is-active "$service"
done
```

## Lab 08 — Web interface

```bash
ss -lntp | grep 8006
curl -k -I https://127.0.0.1:8006
```

## Lab 09 — Storage

```bash
pvesm status
lsblk
df -hT
```

## Lab 10 — Cluster validation

```bash
pvecm status
pvecm nodes
```

---

# 14. Node Configuration Evidence

Record the actual configuration of each node.

## 14.1 Node inventory

| Attribute            | PVE01 | PVE02 | PVE03 |
| -------------------- | ----- | ----- | ----- |
| Hostname             |       |       |       |
| Proxmox VE version   |       |       |       |
| Kernel version       |       |       |       |
| Management IP        |       |       |       |
| Management interface |       |       |       |
| Default gateway      |       |       |       |
| DNS configuration    |       |       |       |
| Time synchronization |       |       |       |
| Web interface        |       |       |       |

## 14.2 Repository inventory

| Node  | Enterprise Repository | No-Subscription Repository | APT Update Status |
| ----- | --------------------- | -------------------------- | ----------------- |
| PVE01 |                       |                            |                   |
| PVE02 |                       |                            |                   |
| PVE03 |                       |                            |                   |

## 14.3 Service inventory

| Service     | PVE01 | PVE02 | PVE03 |
| ----------- | ----- | ----- | ----- |
| pveproxy    |       |       |       |
| pvedaemon   |       |       |       |
| pvestatd    |       |       |       |
| pve-cluster |       |       |       |
| corosync    |       |       |       |
| pve-ha-crm  |       |       |       |
| pve-ha-lrm  |       |       |       |

---

# 15. Troubleshooting Reference

| Problem                     | Initial Investigation              |
| --------------------------- | ---------------------------------- |
| Web interface unavailable   | `systemctl status pveproxy`        |
| Hostname resolution failure | `getent hosts <hostname>`          |
| APT authentication error    | Inspect repository configuration   |
| Node unreachable            | `ip -br addr`, `ip route`, `ping`  |
| Incorrect gateway           | Inspect `ip route`                 |
| Time synchronization issue  | `timedatectl`, `chronyc tracking`  |
| Storage unavailable         | `pvesm status`                     |
| Cluster communication issue | `pvecm status`, Corosync logs      |
| SSH connection failure      | `systemctl status ssh`, `ss -lntp` |
| Resource pressure           | `free -h`, `lscpu`, `df -h`        |

Troubleshoot one layer at a time:

1. Operating system.
2. Hostname and DNS.
3. Network interfaces and routing.
4. Time synchronization.
5. Proxmox services.
6. Storage.
7. Cluster communication.
