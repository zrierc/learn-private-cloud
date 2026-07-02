# OpenStack Networking Troubleshooting Journal

- **Date findings:** July 2nd, 2026

> Personal documentation for kolla-ansible based OpenStack deployments
> OpenStack Release: 2025.2 | Neutron plugin: OVS (OpenVSwitch)

---

## Table of Contents

1. [Lab Environment Overview](#1-lab-environment-overview)
2. [Understanding External Network in OpenStack](#2-understanding-external-network-in-openstack)
3. [Network Architecture (Neutron + OVS)](#3-network-architecture-neutron--ovs)
4. [Problem 1 — Cannot SSH/Ping Floating IP from Controller](#4-problem-1--cannot-sshping-floating-ip-from-controller)
5. [Problem 2 — VM Cannot Ping Internet](#5-problem-2--vm-cannot-ping-internet)
6. [Full Reproducible Setup Checklist](#6-full-reproducible-setup-checklist)
7. [Useful Reference Commands](#7-useful-reference-commands)
8. [Official References](#8-official-references)

---

## 1. Lab Environment Overview

### kolla-ansible globals.yml

```yaml
kolla_base_distro: "ubuntu"
openstack_release: "2025.2"
network_interface: "ens3"
neutron_external_interface: "ens4"
kolla_internal_vip_address: "192.168.122.100"
neutron_plugin_agent: "openvswitch"
enable_keystone: "yes"
enable_horizon: "yes"
enable_neutron_provider_networks: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
cinder_cluster_skip_precheck: "yes"
```

### Node Layout

| Node                    | Role       | ens3 (mgmt)    | ens4 (external)   | ens5   |
| ----------------------- | ---------- | -------------- | ----------------- | ------ |
| pod-username-controller | Controller | 192.168.122.21 | no IP (OVS owned) | unused |
| pod-username-compute1   | Compute    | 192.168.122.x  | no IP             | unused |
| pod-username-compute2   | Compute    | 192.168.122.x  | no IP             | unused |

- **VIP:** `192.168.122.100` (on ens3, shared virtual IP for HA)
- **Internet access:** via `ens3` → default gateway `192.168.122.1`

### Network Configuration Used in This Lab

| Network      | Subnet           | Gateway       | Purpose                     |
| ------------ | ---------------- | ------------- | --------------------------- |
| internal-net | 192.168.168.0/24 | 192.168.168.1 | VM-to-VM private network    |
| external-net | 20.168.168.0/24  | 20.168.168.1  | Floating IPs, router uplink |

> Member ID in this lab is `168`, so external network is `20.168.168.0/24`.

### Network Topology Summary

```
[Your Laptop]
      |
      | (SSH)
      v
[VM Bastion]
      |
      | (SSH)
      v
[VM Controller]  192.168.122.21 (ens3)
[VM Compute1 ]  192.168.122.x  (ens3)
[VM Compute2 ]  192.168.122.x  (ens3)
      |
      | (kolla-ansible deploy → Docker containers)
      v
[OpenStack Services: Nova, Neutron, Glance, Cinder, Keystone, Horizon...]
      |
      | (Nova → libvirt → KVM on compute nodes)
      v
[OpenStack Instances/VMs created by users]
```

### Key Concept: Docker vs KVM — Two Separate Layers

This is a common confusion when using kolla-ansible. There are **two completely separate layers**:

| Layer             | What runs here                                           | Technology             |
| ----------------- | -------------------------------------------------------- | ---------------------- |
| Docker containers | OpenStack **services** (Nova API, Neutron, Glance, etc.) | Kolla-Ansible + Docker |
| KVM/libvirt VMs   | Actual **instances** created by OpenStack users          | Nova → libvirt → KVM   |

**The flow when you create an OpenStack instance:**

```
openstack server create ...
        |
        v
nova-api container (controller) receives request
        |
        v
nova-scheduler picks a compute node
        |
        v
nova-compute container (on compute node) calls libvirt
        |
        v
libvirt/KVM spawns a REAL VM directly on compute node
        (NOT inside Docker, NOT inside any container)
```

You can verify this on a compute node after spawning an instance:

```bash
sudo virsh list --all
# You will see your OpenStack instance listed as a KVM domain
```

---

## 2. Understanding External Network in OpenStack

This section answers three important conceptual questions about how external networks work with kolla-ansible + OVS.

### Q1: Does the external network subnet need to match the IP on ens4?

**No, it does NOT need to match.**

Here is why: when kolla-ansible configures `neutron_external_interface: "ens4"`, it hands `ens4` over to OVS (OpenVSwitch) completely. OVS removes any IP from `ens4` and makes it a **raw bridge port** — essentially just a physical cable plugged into a virtual switch (`br-ex`).

```
BEFORE kolla-ansible deploy:
  ens4: inet 192.168.1.10/24   ← has IP

AFTER kolla-ansible deploy:
  ens4: (no IP, master ovs-system)  ← OVS took it over
  br-ex: virtual switch sitting on top of ens4
```

Since `ens4` no longer has an IP at all, there is nothing to "match". The external network subnet you define in OpenStack (e.g. `20.168.168.0/24`) is a **virtual range** that lives on top of `br-ex`, independent of what IP `ens4` originally had.

### Q2: What if ens4 exists but has no IP? (Like this lab)

This is **expected and correct behavior** after kolla-ansible deploy. It means OVS has successfully taken ownership of `ens4`.

```bash
# This is what a correctly configured ens4 looks like after deploy:
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system ...
    link/ether 52:54:00:4e:db:09 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::...   ← only IPv6 link-local, NO inet (IPv4)
```

The key indicator is `master ovs-system` — this confirms OVS owns the interface. No IPv4 on `ens4` is not a problem; it's proof the configuration is working.

### Q3: What is best practice for external network subnet? Same range or different? Is manual routing normal?

There are **two scenarios** in real deployments:

#### Scenario A — Real routable external network (Production)

```
Internet
    |
Physical router/switch
    |
ens4 (physical uplink, e.g. 203.0.113.0/24)
    |
br-ex
    |
OpenStack external network: 203.0.113.0/24  ← SAME as physical range
    |
Floating IPs: 203.0.113.50 ~ 203.0.113.100
```

In production, the external network subnet **should match** the physical upstream network range that `ens4` is connected to. This way floating IPs are directly routable — no manual NAT or routing tricks needed. The router/switch upstream handles all routing.

#### Scenario B — Lab/fictitious range (This lab)

```
Internet
    |
ens3 (192.168.122.1 → actual internet path)
    |
Controller
    |
br-ex (fictitious range: 20.168.168.0/24)
    |
OpenStack external network: 20.168.168.0/24  ← fictitious, not upstream
```

In a lab where you're forced to use a fictitious range (like `20.168.168.0/24`) that isn't actually routed anywhere, you **must manually set up routing and NAT** to bridge between the fictitious external network and the real internet-connected interface (`ens3`). This is **not standard production practice** — it's a lab workaround.

#### Summary Table

| Scenario       | External subnet matches ens4?         | Manual NAT needed?        | Used in          |
| -------------- | ------------------------------------- | ------------------------- | ---------------- |
| Production     | Yes (matches upstream physical range) | No                        | Real deployments |
| Lab/fictitious | No (arbitrary range)                  | Yes (iptables MASQUERADE) | Learning labs    |

> **In this lab:** we use `20.168.168.0/24` as forced by the task, and manually NAT through `ens3` to reach the internet. This is a lab-only workaround, not something you'd do in production.

---

## 3. Network Architecture (Neutron + OVS)

### Physical Node Layer

```
+---------------------------------------------------------------+
|                   HOST / HYPERVISOR                           |
|                                                               |
|  +-------------------+  +-------------+  +-------------+     |
|  |    Controller     |  |  Compute 1  |  |  Compute 2  |     |
|  |                   |  |             |  |             |     |
|  | ens3 192.168.X.21 |  | ens3        |  | ens3        |     |
|  | ens4 (OVS/br-ex)  |  | ens4        |  | ens4        |     |
|  | ens5 (unused)     |  | ens5        |  | ens5        |     |
|  +-------------------+  +-------------+  +-------------+     |
+---------------------------------------------------------------+
         |                      |                  |
         +---------- ens3 management network ------+
         |            (also carries VXLAN tunnels)
         |
    192.168.122.1
    (default gateway → internet)
```

### Neutron + OVS Layer (on Controller)

```
Internet
    |
    | default route via 192.168.122.1
    |
+---+-----------------------------------------------+
|               Controller Node                     |
|                                                   |
|  ens3: 192.168.122.21  ← management + internet   |
|                                                   |
|  ens4: (no IP, master ovs-system)                 |
|    |                                              |
|  +-+----------+                                   |
|  |  OVS br-ex |  ← virtual bridge over ens4      |
|  |  needs IP! |    e.g. 20.168.168.2/24           |
|  +-----+------+                                   |
|        |  (patch port: phy-br-ex ↔ int-br-ex)    |
|        |                                          |
|  +-----+------------------------------------+     |
|  |         qrouter-<uuid> namespace         |     |
|  |                                          |     |
|  |  qg-xxx: 20.168.168.49/24  (gw IP)      |     |
|  |           20.168.168.100/32 (float IP)   |     |
|  |                                          |     |
|  |  default route: via 20.168.168.2 (br-ex)|     |
|  |  (NOT via 20.168.168.1 = fictitious!)   |     |
|  |                                          |     |
|  |  qr-xxx: 192.168.168.1/24               |     |
|  |          (internal default gateway)      |     |
|  +------------------+-------------------+--+     |
|                     |                   |         |
|  +------------------+-+   +-----------+---------+ |
|  | qdhcp-<uuid> ns   |   |  iptables MASQUERADE | |
|  | 192.168.168.10    |   |  src:20.168.168.0/24 | |
|  | (DHCP server)     |   |  src:192.168.168.0/24| |
|  +-------------------+   |  out: ens3           | |
|                           +----------------------+ |
+---------------------------------------------------+
         |
         | VXLAN tunnel over ens3
         |
+------------------+      +------------------+
|   Compute Node 1 |      |   Compute Node 2 |
|                  |      |                  |
|  [OpenStack VM]  |      |  [OpenStack VM]  |
|  eth0            |      |  eth0            |
|  192.168.168.38  |      |  192.168.168.xx  |
+------------------+      +------------------+
```

### Packet Flow: VM → Internet

```
[OpenStack VM on Compute]
  eth0: 192.168.168.38
  gw:   192.168.168.1
        |
        | (VXLAN tunnel over ens3)
        v
[qrouter namespace on Controller]
  qr-xxx receives packet (192.168.168.1)
  SNAT: src IP changed to 20.168.168.49 (router gw IP)
  forwarded to qg-xxx
        |
        | (via br-ex patch port)
        v
[br-ex: 20.168.168.2]
  packet exits namespace
        |
        | (iptables MASQUERADE)
        | src IP changed to 192.168.122.21 (ens3 IP)
        v
[ens3] → 192.168.122.1 (gateway) → Internet
```

### Packet Flow: Controller → Floating IP (SSH/Ping)

```
[Controller main namespace]
  ping 20.168.168.100
  route: 20.168.168.0/24 dev br-ex
        |
        v
[br-ex: 20.168.168.2]
  ARP: who has 20.168.168.100?
        |
        v
[qrouter namespace]
  qg-xxx answers: I have 20.168.168.100
  DNAT: dst changed to 192.168.168.38 (VM internal IP)
        |
        | (VXLAN tunnel)
        v
[OpenStack VM: 192.168.168.38]
```

---

## 4. Problem 1 — Cannot SSH/Ping Floating IP from Controller

### Symptom

```
$ ping -c 3 <floating-ip>
PING <floating-ip> 56(84) bytes of data.
^C
--- <floating-ip> ping statistics ---
2 packets transmitted, 0 received, 100% packet loss
```

SSH either hangs indefinitely or immediately refuses connection.

### Root Cause

```
Controller main namespace
+---------------------------------------------+
|  ip route show:                             |
|  default via 192.168.122.1 dev ens3         |
|  20.168.168.0/24 dev br-ex  ← route exists |
|                                             |
|  br-ex  ← state DOWN  ← PROBLEM 1          |
|           no IP assigned ← PROBLEM 2       |
+---------------------------------------------+
         |
         | br-ex DOWN = no path into qrouter namespace
         |
+---------------------------------------------+
|  qrouter namespace                          |
|  20.168.168.49  (router gateway IP)         |
|  20.168.168.100 (floating IP as /32)        |
|  ← reachable ONLY from inside namespace     |
+---------------------------------------------+
```

Two problems:

1. `br-ex` is `state DOWN` after kolla-ansible deploy — it's created but not brought up
2. `br-ex` has no IP address — no ARP responder exists for the external range in the main namespace

### Diagnosis Steps

```bash
# Step 1: Check br-ex state
ip link show br-ex
# BAD:  state DOWN
# GOOD: state UNKNOWN or UP

# Step 2: Check br-ex has an IP
ip addr show br-ex
# Should have an inet address in your external subnet range

# Step 3: Check if qrouter namespace exists
sudo ip netns list
# Should show: qrouter-<router-uuid>

# Step 4: Test ping from INSIDE qrouter namespace
sudo ip netns exec qrouter-<router-uuid> ping -c 3 <floating-ip>
# If this works but external ping fails → br-ex is definitely the issue

# Step 5: Confirm OVS bridge has ens4 as port
sudo docker exec openvswitch_vswitchd ovs-vsctl show
# Should show:
#   Bridge br-ex
#     Port ens4
#     Port phy-br-ex
#     Port br-ex
```

### Fix

```bash
# Step 1: Bring br-ex up
sudo ip link set br-ex up

# Step 2: Assign an IP to br-ex (use any unused IP in external subnet)
sudo ip addr add <external-prefix>.2/24 dev br-ex
# Example: sudo ip addr add 20.168.168.2/24 dev br-ex

# Step 3: Add route for external range via br-ex (if not already there)
sudo ip route add <external-prefix>.0/24 dev br-ex

# Step 4: Clear stale SSH known_hosts if VM was recreated
ssh-keygen -f '/home/<user>/.ssh/known_hosts' -R '<floating-ip>'

# Step 5: Test
ping -c 3 <floating-ip>

# Step 6: SSH
ssh -i ~/.ssh/id_rsa <vm-user>@<floating-ip>
# CirrOS: ssh cirros@<floating-ip>  then password: gocubsgo
# Or force password: ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no cirros@<floating-ip>
```

### Verify Fix

```bash
ip link show br-ex         # should show UP or UNKNOWN (not DOWN)
ip addr show br-ex         # should show your assigned IP
ping -c 3 <floating-ip>    # should get replies
```

> ⚠️ These changes are NOT persistent across reboots. See [Section 6](#6-full-reproducible-setup-checklist) for persistence.

---

## 5. Problem 2 — VM Cannot Ping Internet

### Symptom

Inside the OpenStack VM:

```
$ ping 8.8.8.8
From 192.168.168.1 icmp_seq=1 Destination Host Unreachable
# or
connect: Network is unreachable
```

From controller, inside qrouter namespace:

```
$ sudo ip netns exec qrouter-<uuid> ping -c 3 8.8.8.8
From 20.168.168.49 icmp_seq=1 Destination Host Unreachable
```

### Root Cause

There are two sub-problems that must both be fixed.

#### Sub-problem A — qrouter default route points to fictitious gateway

```
[qrouter namespace]
  default route: via 20.168.168.1 dev qg-xxx
                        |
                        | ARP: who has 20.168.168.1?
                        | ...silence... nobody answers
                        | 20.168.168.1 is FICTITIOUS, doesn't exist
                        |
                   packet dies here
                   "Destination Host Unreachable"

  tcpdump on br-ex: 0 packets  ← packet never even leaves namespace
  tcpdump on ens4:  0 packets
```

OpenStack sets the qrouter's default route to the gateway you defined when creating the external subnet (e.g. `20.168.168.1`). But since this is a fictitious range, `20.168.168.1` doesn't actually exist anywhere — no device answers ARP for it, so packets die immediately.

#### Sub-problem B — No NAT between br-ex and ens3

```
Even if packets exit qrouter correctly via br-ex...

[br-ex: 20.168.168.2]
        |
        | packet src: 20.168.168.49
        | dst: 8.8.8.8
        v
[main namespace routing]
  no MASQUERADE rule → packet forwarded to ens3 with src 20.168.168.49
        |
        v
[192.168.122.1 gateway]
  "I don't know how to route back to 20.168.168.49"
  reply packets lost → no internet
```

Without `iptables MASQUERADE`, the controller forwards packets out `ens3` with the original `20.168.168.x` source IP, which the upstream gateway has no route back to.

### Diagnosis Steps

```bash
# Step 1: Test internet from inside qrouter namespace
sudo ip netns exec qrouter-<router-uuid> ping -c 3 8.8.8.8
# "Destination Host Unreachable" → Sub-problem A (fictitious gateway)
# Packet loss with no error     → Sub-problem B (NAT missing)

# Step 2: Check qrouter default route
sudo ip netns exec qrouter-<router-uuid> ip route show
# Look for: default via X.X.X.X dev qg-xxx
# If X.X.X.X doesn't actually exist → Sub-problem A

# Step 3: Use tcpdump to locate where packet dies
# Terminal 1:
sudo tcpdump -i br-ex -n icmp
# Terminal 2:
sudo ip netns exec qrouter-<router-uuid> ping -c 3 8.8.8.8
# 0 packets on br-ex → packet never left qrouter ns → Sub-problem A
# packets seen on br-ex → packet left but dies after → Sub-problem B

# Step 4: Check NAT rules exist
sudo iptables -t nat -L POSTROUTING -n -v
# Look for MASQUERADE rules covering your subnets
```

### Fix

#### Fix Sub-problem A: Replace qrouter default route

```bash
# Find the current qrouter namespace and qg interface
sudo ip netns list                                          # get qrouter-<uuid>
sudo ip netns exec qrouter-<router-uuid> ip addr show      # get qg-<interface-id>

# Replace default route: point to br-ex IP instead of fictitious gateway
sudo ip netns exec qrouter-<router-uuid> \
  ip route replace default via <external-prefix>.2 dev qg-<interface-id>

# Example:
# sudo ip netns exec qrouter-aaa67ed8-... \
#   ip route replace default via 20.168.168.2 dev qg-a2140b6d-b0
```

#### Fix Sub-problem B: Add NAT/MASQUERADE on controller

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# NAT both subnets through the management/internet interface
sudo iptables -t nat -A POSTROUTING -s <internal-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s <external-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE

# Allow forwarding between br-ex and mgmt interface
sudo iptables -A FORWARD -i br-ex -o <mgmt-nic> -j ACCEPT
sudo iptables -A FORWARD -i <mgmt-nic> -o br-ex -j ACCEPT

# Example with real values:
# sudo iptables -t nat -A POSTROUTING -s 192.168.168.0/24 -o ens3 -j MASQUERADE
# sudo iptables -t nat -A POSTROUTING -s 20.168.168.0/24  -o ens3 -j MASQUERADE
# sudo iptables -A FORWARD -i br-ex -o ens3 -j ACCEPT
# sudo iptables -A FORWARD -i ens3  -o br-ex -j ACCEPT
```

### Verify Fix

```bash
# Test from qrouter namespace first
sudo ip netns exec qrouter-<router-uuid> ping -c 3 8.8.8.8
# Should get replies

# Then test from inside the VM
ping -c 3 8.8.8.8
ping -c 3 google.com
# Should get replies
```

---

## 6. Full Reproducible Setup Checklist

Use this every time you set up OpenStack networking from scratch.

### Step 1: Create Networks and Subnets

```bash
# Internal (self-service) network
openstack network create --share internal-net-<username>

openstack subnet create \
  --network internal-net-<username> \
  --allocation-pool start=<internal-prefix>.10,end=<internal-prefix>.254 \
  --dns-nameserver 8.8.8.8 \
  --gateway <internal-prefix>.1 \
  --subnet-range <internal-prefix>.0/24 \
  internal-subnet-<username>

# External (provider) network
openstack network create \
  --share \
  --external \
  --provider-physical-network physnet1 \
  --provider-network-type flat \
  external-net-<username>

openstack subnet create \
  --network external-net-<username> \
  --allocation-pool start=<external-prefix>.10,end=<external-prefix>.254 \
  --dns-nameserver 8.8.8.8 \
  --gateway <external-prefix>.1 \
  --subnet-range <external-prefix>.0/24 \
  --no-dhcp \
  external-subnet-<username>
```

> **Subnet range rules:** Do NOT pick ranges that overlap with your physical node NICs:
>
> - ens3 range (e.g. 192.168.122.0/24) — management network
> - ens4 original range — now owned by OVS
> - Any other NIC on your nodes

### Step 2: Create Router and Attach Networks

```bash
openstack router create router-<username>

openstack router set \
  --external-gateway external-net-<username> \
  router-<username>

openstack router add subnet router-<username> internal-subnet-<username>
```

### Step 3: Fix br-ex (required after every deploy or reboot)

```bash
sudo ip link set br-ex up
sudo ip addr add <external-prefix>.2/24 dev br-ex
sudo ip route add <external-prefix>.0/24 dev br-ex
```

### Step 4: Setup NAT for Internet Access (required after every reboot)

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s <internal-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s <external-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE
sudo iptables -A FORWARD -i br-ex -o <mgmt-nic> -j ACCEPT
sudo iptables -A FORWARD -i <mgmt-nic> -o br-ex -j ACCEPT
```

### Step 5: Fix qrouter Default Route (required after every reboot)

```bash
# Get current qrouter UUID and qg interface name
sudo ip netns list
sudo ip netns exec qrouter-<router-uuid> ip addr show

# Replace fictitious gateway with br-ex IP
sudo ip netns exec qrouter-<router-uuid> \
  ip route replace default via <external-prefix>.2 dev qg-<interface-id>
```

### Step 6: Create Security Group Rules

```bash
# Allow SSH inbound
openstack security group rule create \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0 \
  <security-group-name>

# Allow ICMP (ping) inbound
openstack security group rule create \
  --protocol icmp \
  --remote-ip 0.0.0.0/0 \
  <security-group-name>
```

### Step 7: Create Floating IP and Assign to VM

```bash
openstack floating ip create external-net-<username>
openstack server add floating ip <vm-name> <floating-ip>
```

### Step 8: SSH into VM

```bash
# Clear stale known_hosts entry (important after VM recreation)
ssh-keygen -f '~/.ssh/known_hosts' -R '<floating-ip>'

# SSH with keypair
ssh -i ~/.ssh/id_rsa <vm-user>@<floating-ip>

# SSH with password (force password auth — useful for CirrOS)
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <vm-user>@<floating-ip>
```

### Making Changes Persistent Across Reboots

Steps 3 and 4 are not persistent. Add them to `/etc/rc.local`:

```bash
sudo tee /etc/rc.local << 'EOF'
#!/bin/bash

# Fix br-ex
ip link set br-ex up
ip addr add <external-prefix>.2/24 dev br-ex
ip route add <external-prefix>.0/24 dev br-ex

# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1

# NAT rules
iptables -t nat -A POSTROUTING -s <internal-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE
iptables -t nat -A POSTROUTING -s <external-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE
iptables -A FORWARD -i br-ex -o <mgmt-nic> -j ACCEPT
iptables -A FORWARD -i <mgmt-nic> -o br-ex -j ACCEPT

exit 0
EOF

sudo chmod +x /etc/rc.local
```

> ⚠️ **Step 5 (qrouter default route) cannot be in rc.local** because the qrouter UUID changes if you delete and recreate the router. Run it manually, or write a script that dynamically detects the current qrouter UUID:
>
> ```bash
> QROUTER=$(sudo ip netns list | grep qrouter | awk '{print $1}')
> QG=$(sudo ip netns exec $QROUTER ip addr show | grep qg- | awk '{print $NF}')
> sudo ip netns exec $QROUTER ip route replace default via <external-prefix>.2 dev $QG
> ```

---

## 7. Useful Reference Commands

### Checking Network State

```bash
# List all network namespaces
sudo ip netns list

# Show interfaces inside qrouter namespace
sudo ip netns exec qrouter-<router-uuid> ip addr show

# Show routing table inside qrouter namespace
sudo ip netns exec qrouter-<router-uuid> ip route show

# Show OVS bridge configuration
sudo docker exec openvswitch_vswitchd ovs-vsctl show

# Check br-ex status and IP
ip link show br-ex
ip addr show br-ex

# Check main namespace routing table
ip route show

# Check iptables NAT rules
sudo iptables -t nat -L POSTROUTING -n -v
```

### Checking OpenStack Resources

```bash
openstack network list
openstack subnet list
openstack router show <router-name>
openstack floating ip list
openstack port list --network <network-name>
openstack server show <vm-name>
openstack console log show <vm-name>   # view VM boot log
openstack compute service list         # verify nova-compute agents
openstack network agent list           # verify neutron agents
```

### Debugging Connectivity

```bash
# Packet capture on br-ex
sudo tcpdump -i br-ex -n icmp

# Packet capture on external NIC
sudo tcpdump -i <neutron-external-interface> -n icmp

# Ping from inside router namespace
sudo ip netns exec qrouter-<router-uuid> ping -c 3 <ip>

# Ping from inside DHCP namespace
sudo ip netns exec qdhcp-<network-uuid> ping -c 3 <ip>

# ARP check inside qrouter namespace
sudo ip netns exec qrouter-<router-uuid> arping -c 3 <floating-ip>
```

### CirrOS Specific

```bash
# Default credentials
username: cirros
password: gocubsgo

# SSH with forced password auth
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no cirros@<floating-ip>

# SSH with keypair
ssh -i ~/.ssh/id_rsa cirros@<floating-ip>

# Clear stale known_hosts after VM recreation
ssh-keygen -f '~/.ssh/known_hosts' -R '<floating-ip>'
```

---

## 8. Official References

| Topic                                 | URL                                                                                |
| ------------------------------------- | ---------------------------------------------------------------------------------- |
| Kolla-Ansible Quickstart              | <https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html>             |
| Neutron Networking Concepts           | <https://docs.openstack.org/neutron/latest/admin/intro-os-networking.html>         |
| OpenStack Provider Networks (OVS)     | <https://docs.openstack.org/neutron/latest/admin/deploy-ovs-provider.html>         |
| OpenStack Self-Service Networks (OVS) | <https://docs.openstack.org/neutron/latest/admin/deploy-ovs-selfservice.html>      |
| Neutron Router Namespaces             | <https://docs.openstack.org/neutron/latest/admin/intro-network-components.html>    |
| Get Cloud Images (official list)      | <https://docs.openstack.org/image-guide/obtain-images.html>                        |
| CentOS Stream Cloud Images            | <https://cloud.centos.org>                                                         |
| Ubuntu Cloud Images                   | <https://cloud-images.ubuntu.com>                                                  |
| CirrOS Test Images                    | <https://download.cirros-cloud.net>                                                |
| OpenStack Security Groups             | <https://docs.openstack.org/nova/latest/admin/security-groups.html>                |
| Floating IPs Guide                    | <https://docs.openstack.org/neutron/latest/admin/config-fip-port-forwardings.html> |

---

_Last updated: July 2026 | OpenStack Release: 2025.2 | Deployment: kolla-ansible multinode (1 controller + 2 compute)_
