# OpenShift Virtualization Async DR (VolSync + Ansible)

This repo provides role-based playbooks to configure **asynchronous DR** for KubeVirt VMs using **VolSync** and optionally **MetalLB**.

Key capabilities:
- Install required operators (VolSync and optional MetalLB).
- Discover VM disks → create `ReplicationSource` (source) and `ReplicationDestination` (dest).
- Schedule periodic syncs (every X minutes).
- Automatically pick up **new VM disks** with a re-scan.
- **Capture sanitized VM definitions** from the source and store them on the destination for **exact restoration** (same CPU, memory, devices, interfaces, MACs).
- Failover workflow pauses RD, then restores **the same VM spec** at the DR site, pointing disks to the `*-dst` PVCs.

> Works with Ansible Core and AWX.
