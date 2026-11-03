# Module 01: Virtualization Fundamentals

## 1. Learning Objectives

By completing this module, you should be able to:

- Understand virtualization concepts and business value.
- Differentiate physical and virtual infrastructure.
- Understand Type 1 and Type 2 hypervisors.
- Explain bare-metal and hosted virtualization.
- Understand CPU, memory, storage, and network virtualization.
- Understand hardware-assisted virtualization.
- Explain virtual machines, hypervisors, and guest operating systems.
- Understand nested virtualization.
- Explain the relationship between KVM and QEMU.

## 2. Core Concepts

### 2.1 What is Virtualization?

Virtualization is a technology that abstracts physical computing `resources—CPU`, `memory`, `storage`, and `networking` and presents them as logical resources to multiple isolated virtual machines.

Each virtual machine can run its own operating system and applications independently of other virtual machines.

### 2.2 Physical vs Virtual Infrastructure

| Component        | Physical Infrastructure | Virtual Infrastructure       |
| ---------------- | ----------------------- | ---------------------------- |
| Server           | Physical server         | Virtual machine              |
| CPU              | Physical processor      | vCPU                         |
| Memory           | Physical RAM            | Virtual RAM                  |
| Storage          | Physical disk           | Virtual disk                 |
| Network          | Physical NIC            | Virtual NIC                  |
| Operating System | Installed on hardware   | Installed inside VM          |
| Isolation        | Physical separation     | Hypervisor-managed isolation |

### 2.3 Hypervisor Architecture

A hypervisor is the software layer responsible for creating, managing, and isolating virtual machines.

#### Type 1: Bare-Metal Hypervisor

Runs directly on physical hardware.

Examples:

- Proxmox VE
- VMware ESXi
- Microsoft Hyper-V
- Xen

#### Type 2: Hosted Hypervisor

Runs as an application on top of a host operating system.

Examples:

- Oracle VirtualBox
- VMware Workstation
- VMware Fusion

### 2.4 Architecture Comparison

```text
TYPE 1: BARE-METAL

+-----------------------------+
|        Virtual Machines     |
|  VM1     VM2      VM3       |
+-----------------------------+
|          Hypervisor         |
+-----------------------------+
|       Physical Hardware     |
| CPU | RAM | Disk | Network  |
+-----------------------------+


TYPE 2: HOSTED

+-----------------------------+
|        Virtual Machines     |
|  VM1     VM2      VM3       |
+-----------------------------+
|      Hosted Hypervisor      |
+-----------------------------+
|       Host Operating System |
+-----------------------------+
|       Physical Hardware     |
+-----------------------------+
```

### 2.5 Current Lab Architecture

Our hands-on environment uses nested virtualization.

```text
Ubuntu 24.04.5 LTS (Physical Host)
              |
              v
       Oracle VirtualBox
              |
       +------+------+------+
       |             |       |
       v             v       v
    PVE Node 01   PVE Node 02   PVE Node 03
       |             |             |
       +-------------+-------------+
                     |
               Proxmox Cluster
                     |
                    Ceph
                     |
               Virtual Machines
```

**Note:** Proxmox VE runs as a virtual machine inside VirtualBox, and Proxmox manages additional virtual machines. This is known as nested virtualization.

---

## 3. Hands-on Lab 01: Virtualization Environment Discovery

### 3.1 Lab Objectives

- Identify physical CPU architecture.
- Verify hardware virtualization support.
- Identify host hypervisor and virtualization modules.
- Inspect Proxmox VE virtualization components.
- Identify KVM and QEMU.
- Inspect virtual machines and containers.
- Analyze CPU, memory, and storage resources.

### 3.2 Lab Environment

| Layer               | Technology          |
| ------------------- | ------------------- |
| Physical Host       | Ubuntu 24.04.5 LTS  |
| Host Hypervisor     | VirtualBox 7.2.x    |
| Nested Hypervisor   | Proxmox VE 9.2.x    |
| Cluster             | homelab-cluster     |
| Nodes               | pve01, pve02, pve03 |
| Distributed Storage | Ceph                |

### 3.3 Exercise 1: Identify Physical CPU Architecture

Run these commands on the Ubuntu host:

```bash
lscpu
```

Check CPU virtualization support:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

Interpretation:

- `vmx` indicates Intel VT-x support.
- `svm` indicates AMD-V support.
- A result greater than zero indicates that the CPU exposes virtualization capability to Linux.

Install the CPU checker:

```bash
sudo apt install -y cpu-checker
```

Check KVM availability:

```bash
sudo kvm-ok
```

**Note:** `kvm-ok` checks whether the Ubuntu host can use KVM. It does not determine whether VirtualBox can run nested Proxmox successfully.

### 3.4 Exercise 2: Identify Host Virtualization Modules

On Ubuntu:

```bash
lsmod | grep -E 'kvm|vbox'
```

Check VirtualBox version:

```bash
VBoxManage --version
```

List registered virtual machines:

```bash
VBoxManage list vms
```

### 3.5 Exercise 3: Inspect Proxmox Virtualization Environment

Log in to `pve01` using SSH or the Proxmox shell.

Check Proxmox version:

```bash
pveversion -v
```

Identify CPU virtualization capabilities:

```bash
lscpu | grep -E 'Architecture|Virtualization|Model name'
```

Check KVM modules:

```bash
lsmod | grep kvm
```

Check QEMU version:

```bash
qemu-system-x86_64 --version
sudo apt install qemu-system-x86
```

List virtual machines:

```bash
qm list
```

List containers:

```bash
pct list
```

### 3.6 Exercise 4: Inspect Proxmox Management Services

Run:

```bash
systemctl status pvedaemon
systemctl status pveproxy
systemctl status pvestatd
```

| Service   | Responsibility                              |
| --------- | ------------------------------------------- |
| pvedaemon | Proxmox management API and local operations |
| pveproxy  | Web interface and API proxy                 |
| pvestatd  | Collects and publishes resource statistics  |

### 3.7 Exercise 5: Inspect Hardware Resources

On `pve01`, run:

Check memory:

```bash
free -h
```

Check CPU:

```bash
lscpu
```

Check block devices:

```bash
lsblk
```

Check filesystem usage:

```bash
df -h
```

Observe available CPU, memory, disk, and filesystem resources.
