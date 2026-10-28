# LAB.md - current lab state

Single source of truth for what exists, what runs where, what is broken, and what is next.
Updated at the end of every working session.

**Last updated:** 2026-10-28

---

## Inventory

Full detail: [hardware-inventory](docs/inventory/hardware-inventory.md), [network](docs/inventory/network.md).

| Host | Hardware | Arch | RAM | Storage | Role | Address | Status |
|---|---|---|---|---|---|---|---|
| `fluxcp` | Firebat F1, Ryzen 5 7640HS 6C/12T | amd64 | 18 GiB | 512 GB NVMe + 1 TB USB HDD | Docker host, LAN DNS (AdGuard), planned k3s control plane | 10.0.0.225 | **Live**, 13 containers |
| `k3s-worker-01` | Raspberry Pi 5 | arm64 | 8 GB | 256 GB NVMe | k3s agent | 10.0.0.12 | **Live** |
| `k3s-worker-02` | Raspberry Pi 5 | arm64 | 8 GB | 256 GB NVMe | k3s agent | 10.0.0.240 (eth0) / .13 (wlan0) | **Live**, needs fixes |
| `k3s-worker-03` | Raspberry Pi 5 | arm64 | 8 GB | 256 GB NVMe | k3s agent | 10.0.0.14 | **Live** |
| `ha-01` | Raspberry Pi 5 (repaired, as-is) | arm64 | 4 GB | TBD | Home Assistant OS, outside cluster | 10.0.0.30 planned | Boxed |
| ER-X-SFP | Ubiquiti EdgeRouter X SFP, EdgeOS v3.0.1 | - | - | - | Router, DHCP, firewall | 10.0.0.1 | **Live** |
| LS1005G | TP-Link unmanaged switch | - | - | - | Switch for the 3 Pis (on eth3) | - | Live |
| UAP-AC-LR | Ubiquiti AP | - | - | - | Wi-Fi | 10.0.0.39 | Live |

Retired: Dell OptiPlex (`pve-01`), Mac mini. TP-Link AX1800 is no longer the router.

**arm64 capacity:** 3 x 8 GB. **amd64 capacity:** 18 GiB on `fluxcp`.

---

## What runs where

`fluxcp` runs Docker Compose services: adguard-home (LAN DNS), Paperless-ngx stack (6 containers), jellyfin, gluetun + deluge, dockhand, heimdall, webtop. No Kubernetes workloads. The Pis run nothing yet.

---

## Cluster state

No cluster. k3s is not installed on any node. The control plane was planned as a Proxmox VM on the OptiPlex; with the OptiPlex retired, the control-plane host is an open decision (candidates: a VM or k3s server directly on `fluxcp`, which keeps the amd64 control plane / arm64 agents mixed-architecture property). Note `fluxcp` has swap on and hosts live family services.

---

## Syllabus coverage

| Phase | Topic | Status |
|---|---|---|
| 1 | Linux admin, IP/DNS plan, naming, Git runbook | In progress - inventory re-audited 2026-10-28 |
| 2 | Proxmox VE | **Plan changed** - OptiPlex retired; revisit whether Proxmox on fluxcp is wanted |
| 3 | Docker | Practical exposure on fluxcp (Compose stack live); formal unit not done |
| 4 | Kubernetes (k3s) | Blocked on control-plane decision |
| 5 | Observability | Not started |
| 6 | Automation / IaC | Next - Ansible baseline for the Pis |
| 7 | Reliability, TLS, backup + tested restore | Not started |

---

## Open breakage / known debt

| Item | Impact | Next action |
|---|---|---|
| worker-02 DHCP reservation bound to wlan0 MAC | eth0 sits on unreserved .240; wlan0 holds .13 | Change mapping MAC to 2c:cf:67:19:73:34 and disable wlan0 (Ansible run) |
| worker-02 on 26.04 / kernel 1016, desktop snaps installed | Version skew vs -01/-03; extra RAM use | Ansible: full-upgrade, remove desktop snaps |
| Time sync unverified on Pis | Clock skew breaks TLS and etcd | Check timedatectl on Pis; enable chrony/timesyncd in baseline |
| fluxcp / is 74.5G (33.5G free) | Docker data on / could fill the root | Monitor; consider moving Docker data-root to /home |
| fluxcp is single DNS server for the LAN | Outage = no DNS for all clients | Add secondary resolver or router fallback later |
| Several fluxcp services bind 0.0.0.0 | LAN-wide exposure | Review before reverse proxy / TLS work |
| EdgeRouter PoE state not captured | Passive PoE risk on wrong port | `show configuration commands | match poe` |
| DHCP pool .38-.243 overlaps appliance block | Plan drift | Shrink pool to .100-.200 |

---

## Artifacts shipped

| Artifact | Path |
|---|---|
| SSH key strategy ADR | `docs/decisions/0001-ssh-key-strategy.md` |
| Network architecture ADR | `docs/decisions/0002-network-architecture.md` (needs update: ER-X is now live) |
| Hardware inventory | `docs/inventory/hardware-inventory.md` |
| Network inventory | `docs/inventory/network.md` |
| `k3s-worker-01` baseline | `docs/inventory/k3s-worker-01.md` |
| Runbooks | `docs/runbooks/` |

---

## Next 3 tasks

1. **Ansible setup** - inventory file for the 3 Pis; baseline playbook (disable wlan0, full-upgrade, remove desktop snaps on -02, time sync, swap off, UFW).
2. **Fix worker-02 reservation** on the EdgeRouter (MAC to ...73:34) as step 0 of the Ansible run.
3. **Decide control-plane host** and update ADR 0002 and the plan accordingly.

---

## Decisions log

| ADR | Decision | Status |
|---|---|---|
| 0001 | Ed25519 SSH keys, one per client device per purpose | Accepted |
| 0002 | ER-X-SFP as router/DHCP/firewall; 10.0.0.0/24 held constant; wired-only nodes | Needs revision - migration done, port-count solved with LS1005G |

---

## How I update this file

At the end of every session: update Inventory, Open breakage, Artifacts, and Next 3 tasks.
Commit it with the work it describes, in the same commit where possible. If this file disagrees
with reality, this file is the bug.
