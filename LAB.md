# LAB.md — current lab state

Single source of truth for what exists, what runs where, what is broken, and what is next.
Updated at the end of every working session.

**Last updated:** 2026-08-16

---

## Inventory

| Host | Hardware | Arch | RAM | Storage | Role | Address | Status |
|---|---|---|---|---|---|---|---|
| `er-x-01` | Ubiquiti ER-X-SFP | — | — | — | Router, DHCP, firewall | 10.0.0.1 | **Owned, unconfigured** |
| `pve-01` | Dell OptiPlex | amd64 | TBD | TBD | Proxmox VE host | 10.0.0.10 | **Not yet owned — arrives 2026-08-20** |
| `k3s-cp-01` | Ubuntu VM on `pve-01` | amd64 | 4 GB planned | virtual | k3s server | 10.0.0.11 | Planned (Week 6) |
| `k3s-worker-01` | Raspberry Pi 5 | arm64 | 8 GB | 64 GB microSD | k3s agent | 10.0.0.12 | **Live**, hardened |
| `k3s-worker-02` | Raspberry Pi 5 + M.2 HAT+ | arm64 | 8 GB | **256 GB NVMe** | k3s agent, storage node | 10.0.0.13 | **Live**, undocumented |
| `k3s-worker-03` | Raspberry Pi 5 (BCM2712) | arm64 | 8 GB | TBD | k3s agent | 10.0.0.14 | Boxed |
| `ha-01` | Raspberry Pi 5 (repaired, as-is) | arm64 | 4 GB | TBD | Home Assistant OS, **outside cluster** | 10.0.0.30 | Boxed |
| — | TP-Link Archer AX1800 | — | — | — | **Current** router/DHCP/WiFi/switch | 10.0.0.1 | Live |
| — | Ubiquiti UAP-AC-LR | — | — | — | WiFi AP | 10.0.0.3x | Owned, unconfigured |
| — | Mac mini | amd64 | TBD | TBD | Workstation / Docker host | DHCP pool | Live |

**arm64 capacity:** 24 GB across three 8 GB Pis. **amd64 capacity:** unknown until 2026-08-20.

---

## What runs where

Nothing containerized or orchestrated yet. Two hardened-ish Linux hosts and a consumer router.

---

## Cluster state

No cluster. `k3s` is not installed on any node. Planned name `k3s-lab`, planned API endpoint
`k3s-api.home.arpa`, control plane deliberately **amd64** so the cluster is genuinely
mixed-architecture.

---

## Syllabus coverage

| Phase | Topic | Status |
|---|---|---|
| 1 | Linux admin, IP/DNS plan, naming, Git runbook | In progress — plan and conventions exist, node 2 undocumented |
| 2 | Proxmox VE | **Blocked** — hardware arrives 2026-08-20 |
| 3 | Docker | Not started |
| 4 | Kubernetes (k3s) | **Blocked** on phase 2 |
| 5 | Observability | Not started |
| 6 | Automation / IaC | Not started |
| 7 | Reliability, TLS, backup + tested restore | Not started |

---

## Open breakage / known debt

| Item | Impact | Next action |
|---|---|---|
| `k3s-worker-02` undocumented and unhardened | A live node with no inventory record, no confirmed SSH policy, no firewall | Week 1 task 4.10 — inventory doc + Ed25519 key-only SSH + UFW + swap off + NTP |
| Pis were dual-homed on Ethernet **and** WiFi | Ambiguous node addressing; would break kubelet node IP inference | Runbook written 2026-08-16. Apply `dtoverlay=disable-wifi`, verify, then delete stale `wlan0` reservations |
| Stale `wlan0` DHCP reservations | worker-01's `wlan0` is reserved at `10.0.0.11`, **colliding with planned `k3s-cp-01`** | Delete on the AX1800 before the EdgeRouter migration |
| `eth0` vs `wlan0` MAC unresolved per node | Reservations could bind the wrong interface | `ip -brief link show` on each node |
| ER-X-SFP firmware version unknown | Blocks writing verified migration commands — EdgeOS v1.x and v2.x differ | Capture `show version`, update firmware |
| Port-count decision open | 4 LAN ports for 7 devices | Decide Option A vs B; move [ADR 0002](docs/decisions/0002-network-architecture.md) to Accepted |
| `pve-01` specs unconfirmed | Blocks Proxmox sizing; VT-x/AMD-V is a hard gate | 2026-08-20+: `lscpu`, `free -h`, `lsblk`, `dmidecode` |
| Ubuntu release parity between worker-01 and -02 unverified | Divergent nodes are a bad base for a cluster | Check `lsb_release -a` on both |

---

## Artifacts shipped

| Artifact | Path |
|---|---|
| SSH key strategy ADR | `docs/decisions/0001-ssh-key-strategy.md` |
| Network architecture ADR | `docs/decisions/0002-network-architecture.md` |
| Network inventory | `docs/inventory/network.md` |
| `k3s-worker-01` baseline | `docs/inventory/k3s-worker-01.md` |
| 8 runbooks | `docs/runbooks/` |

---

## Next 3 tasks

1. **Document and harden `k3s-worker-02`** (~90 min) — inventory doc, DHCP reservation at
   `.13`, Ed25519 key-only SSH, default-deny UFW with 22/tcp scoped to `10.0.0.0/24`, swap off,
   NTP verified, and confirm `findmnt /` reports the NVMe device rather than the microSD.
2. **Disable WiFi on both live Pis** and delete the stale `wlan0` reservations — procedure in
   `docs/runbooks/dual-homed-pi-disable-wifi.md`.
3. **Capture `pve-01` specs** on 2026-08-20 when the OptiPlex arrives. Hard gate: `lscpu` must
   report VT-x or AMD-V under `Virtualization`. Do not install Proxmox until sizing is agreed.

---

## Decisions log

| ADR | Decision | Status |
|---|---|---|
| 0001 | Ed25519 SSH keys, one per client device per purpose | Accepted |
| 0002 | ER-X-SFP as router/DHCP/firewall; `10.0.0.0/24` held constant; wired-only nodes; migrate at Week 4.5 | **Proposed** — port-count option open |

**Standing decisions not yet in an ADR:**

- Control plane stays **amd64 on the OptiPlex**, not on a Pi. Preserves the mixed-architecture
  property, which is the most interview-valuable thing about this cluster. Accepted consequence:
  the OptiPlex is a single point of failure for the plan.
- `k3s-worker-02` is the **storage node**. Write-heavy workloads (PostgreSQL, Prometheus TSDB)
  get pinned there via `nodeSelector: {storage: nvme}` — a **label**, not a hostname, so the
  role can move. Never run write-heavy workloads on the microSD-booted `k3s-worker-01`.
- The 4 GB repaired Pi runs **Home Assistant OS outside the cluster**. Least reliable hardware,
  family-facing, and not on the syllabus. Automatic backups configured on day one.
- If the OptiPlex is dead on arrival, the replacement is a **used mini PC, not another Pi.**
  The mixed-architecture story cannot be bought back with more arm64.

---

## How I update this file

At the end of every session: update Inventory, Open breakage, Artifacts, and Next 3 tasks.
Commit it with the work it describes, in the same commit where possible. If this file disagrees
with reality, this file is the bug.
