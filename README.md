# homelab

Infrastructure lab: documentation, inventory, and configuration as code.
Built, broken, and repaired deliberately — the runbooks are the point.

## Layout

| Path | Contents |
|---|---|
| `LAB.md` | Current lab state: inventory, open breakage, artifacts, next 3 tasks |
| `docs/inventory/` | Per-host baseline records: hardware, OS, network, access, firewall |
| `docs/runbooks/` | Symptom → root cause → fix entries from real breakage |
| `docs/decisions/` | Architecture decision records (ADRs) |

## Current state

See **[LAB.md](LAB.md)** for the authoritative, per-session state: full inventory, open
breakage, artifacts shipped, and next tasks.

At a glance:

| Host | Role | Platform | Status |
|---|---|---|---|
| `k3s-worker-01` | Kubernetes worker (future) | Raspberry Pi 5, 8 GB, arm64 | Baseline done, hardened |
| `k3s-worker-02` | Kubernetes worker, storage node | Raspberry Pi 5, 8 GB, arm64, 256 GB NVMe | Live, **undocumented** |
| `k3s-worker-03` | Kubernetes worker (future) | Raspberry Pi 5, 8 GB, arm64 | Boxed |
| `ha-01` | Home Assistant, outside cluster | Raspberry Pi 5, 4 GB, arm64 | Boxed |
| `pve-01` | Proxmox VE host | Dell OptiPlex, amd64 | Arrives 2026-08-20 |

- Cluster name (planned): `k3s-lab`
- Planned control plane: `k3s-cp-01` — an **amd64** VM, deliberately, so the cluster is
  genuinely mixed-architecture
- Future API endpoint: `k3s-api.home.arpa`
- LAN: `10.0.0.0/24`. Routing and DHCP are currently on a consumer TP-Link AX1800; migration to
  the Ubiquiti ER-X-SFP is planned — see [ADR 0002](docs/decisions/0002-network-architecture.md)

## Conventions

- Hostnames: lowercase, role plus two-digit sequence — `k3s-cp-01`, `k3s-worker-01`
- SSH: Ed25519 keys only, key-per-device-per-purpose, password auth disabled
- Addressing: DHCP reservations on the router, never static IPs on hosts
- Secrets: never committed — see `.gitignore`; private IPs (RFC1918) are safe to record

## Runbook format

Every entry answers five questions: symptom, root cause, fix, how to detect it faster,
and what to automate so it does not recur.
