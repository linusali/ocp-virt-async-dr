> ⚠️ **Disclaimer**: This README is Work In Progress. Please verify all commands, variable names, and steps before using it in **any** environment.

# OpenShift Virtualization Async DR (VolSync + Ansible)

Role-based Ansible playbooks for asynchronous Disaster Recovery of KubeVirt VMs using VolSync's **rsyncTLS** transport. Disks are replicated on a schedule from a source cluster to a destination cluster; VM specs are stored at the DR site so VMs can be restored exactly (same CPU, memory, NICs, MACs) during failover.

---

## How it works

```
Source cluster                          Destination cluster
──────────────────────────────────      ────────────────────────────────────
VM  →  DV  →  PVC                       PVC  (pre-created to match source)
               │                               │
        ReplicationSource  ──rsyncTLS──▶  ReplicationDestination
        (cron schedule)                   (LoadBalancer endpoint)

VM spec (sanitized)  ──stored as──▶  VirtualMachine (runStrategy: Halted)
```

1. **Setup** – VolSync operator (and optionally MetalLB) is installed on both clusters via OLM.
2. **Configure sync** – A shared TLS PSK is generated and pushed as a Secret to every DR namespace on both clusters. PVCs are discovered by walking VM → DataVolume → PVC. A `ReplicationDestination` (RD) is created on the dest for each PVC; a `ReplicationSource` (RS) is created on the source pointing at the RD's LoadBalancer address. The sanitized VM spec is stored on the dest cluster as a halted `VirtualMachine` object.
3. **Failover** – All RDs on the dest are paused to freeze the replicated data, then the stored VM definitions are applied and (optionally) started.

---

## Prerequisites

```bash
dnf install ansible-core git python3-pip
pip3 install kubernetes
ansible-galaxy collection install -r requirements.yml
```

`requirements.yml` installs: `kubernetes.core`, `ansible.posix`, `ansible.utils`, `community.general`.

---

## Inventory (`inventories/lab/inventory.yml`)

| Variable | Description |
|---|---|
| `k8s_src_context` | kubeconfig context for the source cluster |
| `k8s_dest_context` | kubeconfig context for the destination cluster |
| `k8s_auth_verify_ssl` | Set `false` to skip TLS verification (lab use) |
| `dr_vms` | List of `{name, namespace}` VMs to protect |
| `volsync_every_minutes` | Replication interval (cron `*/N * * * *`) |
| `volsync_copy_method` | `Snapshot` (default) or `Direct` |
| `src_storage_class` | StorageClass for RS snapshots on source |
| `src_snapshot_class` | VolumeSnapshotClass on source |
| `dest_storage_class` | StorageClass for destination PVCs |
| `dest_snapshot_class` | VolumeSnapshotClass on destination |
| `dest_access_modes` | Access modes for destination PVCs |
| `volsync_tls_secret_name` | Name of the PSK Secret (same on both clusters) |
| `volsync_tls_key_id` | Numeric ID prefix in the PSK file |
| `volsync_tls_key_file` | Path to the PSK file — **store outside the repo** |
| `start_on_restore` | `true` to start VMs immediately after failover restore |
| `enable_metallb` | `true` to install MetalLB on the dest cluster |
| `metallb_ip_pool_cidr` | CIDR for the MetalLB `IPAddressPool` (e.g. `192.168.1.230/29`) |
| `metallb_pool_name` | Name of the `IPAddressPool` and `L2Advertisement` objects |
| `metallb_l2_ns` | Namespace where MetalLB CRs are created (typically `metallb-system`) |

---

## Playbooks

### 1. `setup.yml` — Install operators

Installs the **VolSync** operator on both clusters via OLM. Optionally installs **MetalLB** on the destination cluster (needed when the dest has no cloud LB).

```bash
ansible-playbook playbooks/setup.yml
```

### 2. `configure-sync.yml` — Set up replication

Run after `setup.yml`. Does the full configuration in order:

1. Generates a PSK key file (if absent) and pushes it as a Secret to every DR namespace on both clusters (`tls_psk` role).
2. Discovers PVCs for all `dr_vms` on the source by walking VM → DataVolume → PVC (`discover` role, source mode). VMs that have no `dataVolumeTemplates` (standalone PVCs) are skipped with a warning.
3. Pre-creates matching PVCs on the destination and applies `ReplicationDestination` CRs, then waits up to 6 minutes for each RD's LoadBalancer address to be assigned (`rd_dest` role).
4. Creates `ReplicationSource` CRs on the source pointing at each RD address (`rs_source` role).
5. Ensures all RS objects carry the configured cron schedule; patches any that differ (`schedule` role).
6. Exports each VM's spec from the source (sanitized: no `resourceVersion`, no `uid`, `runStrategy: Halted`) and applies it on the destination so it is ready for instant restore (`vm_def` role).

```bash
ansible-playbook playbooks/configure-sync.yml
```

Re-running this playbook is safe and idempotent — new disks added to protected VMs will be picked up automatically.

### 3. `failover.yml` — Failover to destination

> **Requires explicit confirmation.** Pass `--extra-vars confirm_failover=true` or the play will abort.

1. Discovers all `ReplicationDestination` CRs on the dest cluster (`discover` role, dest mode).
2. Patches every RD to `paused: true` to freeze replication at the last consistent snapshot.
3. Starts the stored VM definitions on the dest cluster (`restore` role). Whether VMs are started is controlled by `start_on_restore` in inventory.

```bash
ansible-playbook playbooks/failover.yml -e confirm_failover=true
```

### 4. `tls-psk.yml` — Rotate / re-push PSK only

Re-generates (if absent) and re-pushes the PSK Secret to both clusters without touching any VolSync or VM objects. Useful after adding a new namespace or rotating the key.

```bash
ansible-playbook playbooks/tls-psk.yml
```

---

## Role reference

| Role | Responsibility |
|---|---|
| `prereqs` | OLM Subscriptions for VolSync and (optionally) MetalLB; waits for each CSV to reach `Succeeded`; creates MetalLB instance, `IPAddressPool`, and `L2Advertisement` when `enable_metallb: true` |
| `tls_psk` | Generates PSK file, creates `volsync-tls-psk` Secret in every DR namespace on both clusters |
| `discover` | Source mode: VM → DV → PVC walk; Dest mode: collects existing RD CRs |
| `rd_dest` | Creates destination PVCs + `ReplicationDestination` CRs; waits for LB address |
| `rs_source` | Creates `ReplicationSource` CRs using RD addresses resolved by `rd_dest` |
| `schedule` | Idempotent schedule patch — ensures all RS objects match `volsync_every_minutes` |
| `vm_def` | Exports sanitized VM specs from source, stores as halted VMs on destination |
| `restore` | Applies stored VM definitions on dest; starts VMs if `start_on_restore: true` |

---

## Typical run order

```
setup.yml          (once, or when adding a new cluster)
    │
configure-sync.yml (initial setup, then re-run whenever dr_vms changes)
    │
    │  ← normal operations: VolSync syncs on cron schedule →
    │
failover.yml -e confirm_failover=true   (during a DR event)
```

---

## Security notes

- The PSK file path (`volsync_tls_key_file`) defaults to `~/.ocp-virt-async-dr/keys/psk.txt`. Keep this file **outside the repository**.
- `k8s_auth_verify_ssl: false` is fine for lab clusters but should be `true` (and proper CA certs configured) in production.
- The failover playbook defaults `confirm_failover: false`; a failover will abort unless you explicitly pass `-e confirm_failover=true`.
