# RoCE / PFC Lossless Fabric for Ceph on Proxmox VE
### Mellanox NICs + Mellanox Switches (CLAG/MLAG Bond Setup)

---

## Overview

This guide configures a lossless Layer 2 fabric for Ceph cluster network traffic on Proxmox VE nodes using Mellanox ConnectX NICs bonded in LACP (802.3ad) to a pair of Mellanox switches running in CLAG/MLAG. The result is:

- Dedicated lossless buffer pool for Ceph replication traffic
- PFC (Priority Flow Control) on priority 3 to prevent drops under congestion
- ECN marking before drops occur
- DSCP 26 marking at the host so the switch classifies traffic correctly
- Jumbo frames (MTU 9000/9216) end-to-end

This does **not** use RDMA/RoCE at the application layer — Ceph's RDMA messenger is abandoned upstream. The benefit is a lossless TCP fabric for OSD replication traffic.

---

## Prerequisites

- Proxmox VE (Debian 13 Trixie base)
- Ceph installed and healthy
- Mellanox ConnectX-5 or newer NICs (ConnectX-6 DX recommended)
- Two Mellanox switches in CLAG/MLAG with `nv set qos roce` support
- A dedicated VLAN for Ceph cluster network (this guide uses VLAN 15, subnet 172.19.15.0/24)

Install required packages on all PVE nodes:

```bash
apt install -y rdma-core ibverbs-utils infiniband-diags iproute2
```

---

## Architecture

```
NIC1 + NIC2 (LACP bond) → bond0 → vmbr0 (VLAN-aware bridge, MTU 9216)
                                        └── vmbr0.15  (172.19.15.x/24, MTU 9000)  ← Ceph cluster_network
                                        └── vmbr0.10  (172.19.10.x/24)             ← Management / public_network
```

Both NICs bond into `bond0`, which is the sole port on a VLAN-aware Linux bridge `vmbr0`. Ceph cluster traffic rides a dedicated VLAN subinterface.

---

## Step 1 — Network Interfaces

Edit `/etc/network/interfaces` on each PVE node. Only the `address` lines differ per node.

```
auto lo
iface lo inet loopback

auto nic1
iface nic1 inet manual
        mtu 9216

auto nic2
iface nic2 inet manual
        mtu 9216

auto bond0
iface bond0 inet manual
        bond-slaves nic1 nic2
        bond-miimon 100
        bond-mode 802.3ad
        bond-xmit-hash-policy layer3+4
        bond-lacp-rate 1
        mtu 9216

auto vmbr0
iface vmbr0 inet static
        address 172.19.10.X/24
        gateway 172.19.10.1
        bridge-ports bond0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094
        mtu 9216

auto vmbr0.15
iface vmbr0.15 inet static
        address 172.19.15.X/24
        mtu 9000
        post-up dcb pfc set dev nic1 prio-pfc 3:on
        post-up dcb pfc set dev nic2 prio-pfc 3:on
        post-up dcb ets set dev nic1 prio-tc 0:0 1:1 2:2 3:3 4:4 5:5 6:6 7:7 tc-tsa 0:strict 1:strict 2:strict 3:strict 4:strict 5:strict 6:strict 7:strict tc-bw 0:0 1:0 2:0 3:0 4:0 5:0 6:0 7:0
        post-up dcb ets set dev nic2 prio-tc 0:0 1:1 2:2 3:3 4:4 5:5 6:6 7:7 tc-tsa 0:strict 1:strict 2:strict 3:strict 4:strict 5:strict 6:strict 7:strict tc-bw 0:0 1:0 2:0 3:0 4:0 5:0 6:0 7:0

source /etc/network/interfaces.d/*
```

Replace `nic1`/`nic2` with your actual interface names (e.g. `enp153s0f0np0`). Replace `X` with each node's address octet.

Apply on each node:

```bash
ifreload -a
```

---

## Step 2 — Verify DCB/PFC on NICs

Confirm PFC is active on both physical NICs:

```bash
dcb pfc show dev nic1
dcb pfc show dev nic2
```

Expected output includes `prio-pfc 3:on`. Also verify ETS:

```bash
dcb ets show dev nic1
```

Expected: `prio-tc 3:3`, all TCs set to `strict`.

---

## Step 3 — Sysctl Tuning

Create `/etc/sysctl.d/99-ceph-roce.conf` on all nodes:

```
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.core.netdev_max_backlog = 250000
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_ecn = 1
```

Apply:

```bash
sysctl -p /etc/sysctl.d/99-ceph-roce.conf
```

---

## Step 4 — Switch Configuration

On both switches, enable RoCE lossless mode. This single command configures PFC on priority 3, ECN on TC 0 and TC 3, DSCP 26 → SP3 mapping, and dedicated lossless buffer pools automatically:

```bash
nv set qos roce
nv config apply
```

Ensure the Ceph cluster VLAN is present in the default bridge and tagged on all host-facing bonds:

```bash
nv set bridge domain br_default vlan 15
nv set interface <bond-name> bridge domain br_default vlan 15
nv config apply
```

Verify RoCE counters on a host-facing port:

```bash
nv show interface <swpX> qos roce counters
```

> **Note:** Query on bond member ports (e.g. `swp2`), not the bond interface itself — the command will error on bond interfaces.

---

## Step 5 — Jumbo Frame Verification

Verify end-to-end jumbo frames between all node pairs on VLAN 15:

```bash
ping -M do -s 8972 172.19.15.Y
```

All pings must succeed. The `-s 8972` value accounts for 28 bytes of IP+ICMP overhead against a 9000 MTU. If any fail, check MTU settings at each layer: switch port, bond, bridge, and VLAN subinterface.

---

## Step 6 — Ceph cluster_network

Add or update `cluster_network` in `/etc/ceph/ceph.conf`:

```ini
[global]
        ...
        public_network = 172.19.10.0/24
        cluster_network = 172.19.15.0/24
```

If using PVE's Ceph integration, this file is auto-synced to all nodes. Otherwise copy it manually.

Restart OSDs on all nodes:

```bash
systemctl restart ceph-osd.target
```

Verify OSDs are bound to cluster network addresses:

```bash
ceph osd dump | grep "cluster_addr"
```

Each OSD should show a `172.19.15.x` cluster address alongside its public address.

---

## Step 7 — DSCP Marking (nftables)

The switch classifies traffic as RoCE based on DSCP 26. Ceph does not set socket priority, and Linux's VLAN-aware bridge does not propagate egress QoS maps through to the physical NIC, so marking must be applied via nftables in the kernel mangle hook. This runs entirely in-kernel and has negligible overhead at 100G.

Create `/etc/nftables.conf` on all nodes:

```
#!/usr/sbin/nft -f
flush ruleset

table ip ceph_dscp {
    chain output {
        type route hook output priority mangle; policy accept;
        ip daddr 172.19.15.0/24 ip dscp set 26
    }
}
```

Enable and apply:

```bash
systemctl enable nftables
systemctl restart nftables
```

Deploy to all nodes:

```bash
for node in pve-node2 pve-node3 pve-node4; do
  scp /etc/nftables.conf root@$node:/etc/nftables.conf
  ssh root@$node "systemctl enable nftables && systemctl restart nftables"
done
```

---

## Step 8 — Verify DSCP Marking

On any node, confirm all outbound cluster traffic is marked `tos 0x68` (DSCP 26):

```bash
tcpdump -i vmbr0.15 -v -c 10 src 172.19.15.X 2>/dev/null | grep tos
```

Every line should show `tos 0x68`. If any show `tos 0x0`, check that nftables is running and the rule is active:

```bash
nft list ruleset
```

---

## Step 9 — Verify Switch Classification

After a few minutes of Ceph activity, check that the switch is classifying traffic into the RoCE buffer pool. Clear counters first for a clean reading:

```bash
nv action clear interface <swpX> qos roce counters
# wait ~60 seconds
nv show interface <swpX> qos roce counters
```

A healthy result shows:

- `rx-roce-packets` climbing with substantial byte counts
- `no-buffer-discard: 0` on both RoCE and non-RoCE pools
- `pause-packets: 0` or very low — PFC triggered rarely or not at all
- `cnp-packets` near zero — ECN congestion notifications minimal

---

## Step 10 — Load Test

Run simultaneous write benchmarks on all nodes to stress the cluster network:

```bash
rados bench -p <pool-name> 60 write --no-cleanup -t 16 -b 4M
```

Under load, watch for:

- Sustained throughput with low latency variance
- `no-buffer-discard` remaining at 0 on the switch
- PFC pauses are acceptable under heavy load — that is the system working as designed
- `no-buffer-discard` incrementing would indicate the lossless setup is not functioning correctly

Clean up after the benchmark:

```bash
rados -p <pool-name> cleanup
```

---

## Troubleshooting

**DCB/PFC not applying after reboot**
The `post-up` commands in `/etc/network/interfaces` run when `vmbr0.15` comes up. If DCB is not set, confirm `ifreload -a` completed without errors and that `ifupdown2` is installed (standard on PVE/Debian 13).

**DSCP not marked (`tos 0x0` on outbound traffic)**
Confirm nftables service is running: `systemctl status nftables`. Check the ruleset is loaded: `nft list ruleset`. The rule must target the cluster subnet exactly.

**Switch shows all traffic as non-RoCE**
DSCP marking is not reaching the switch. Verify with `tcpdump` on the cluster VLAN interface that outbound packets show `tos 0x68` before investigating the switch side.

**Jumbo frames failing**
Work through the path layer by layer: physical NIC MTU → bond MTU → bridge MTU → switch port MTU. All must be ≥ 9216. The VLAN subinterface (`vmbr0.15`) can be set to 9000 as the usable payload MTU.

**OSD not using cluster_network**
Confirm `cluster_network` is in `ceph.conf` and OSDs were restarted after the change. Verify with `ceph osd dump | grep cluster_addr` — if addresses still show the public network, the restart did not pick up the config.
