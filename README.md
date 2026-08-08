# homelab

Infrastructure lab: documentation, inventory, and configuration as code.
Built, broken, and repaired deliberately — the runbooks are the point.

## Layout

| Path | Contents |
|---|---|
| `docs/inventory/` | Per-host baseline records: hardware, OS, network, access, firewall |
| `docs/runbooks/` | Symptom → root cause → fix entries from real breakage |
| `docs/decisions/` | Architecture decision records (ADRs) |

## Current state

| Host | Role | Platform | Status |
|---|---|---|---|
| `k3s-worker-01` | Kubernetes worker (future) | Raspberry Pi 5, 8 GB, ARM64 | Baseline in progress |

- Cluster name (planned): `k3s-lab`
- Planned control plane: `k3s-cp-01`
- Future API endpoint: `k3s-api.home.arpa`
- LAN: `10.0.0.0/24`, EdgeRouter ER-X-SFP

## Conventions

- Hostnames: lowercase, role plus two-digit sequence — `k3s-cp-01`, `k3s-worker-01`
- SSH: Ed25519 keys only, key-per-device-per-purpose, password auth disabled
- Addressing: DHCP reservations on the router, never static IPs on hosts
- Secrets: never committed — see `.gitignore`; private IPs (RFC1918) are safe to record

## Runbook format

Every entry answers five questions: symptom, root cause, fix, how to detect it faster,
and what to automate so it does not recur.
