# ADR 0002 — Lab network architecture and the EdgeRouter migration

- **Status:** Proposed
- **Date:** 2026-08-16
- **Scope:** router, DHCP authority, addressing plan, WiFi, and the migration path
- **Supersedes:** nothing. **Superseded by:** nothing.

## Context

The lab currently runs behind a **TP-Link Archer AX1800** acting as router, NAT gateway, DHCP
server, WiFi access point, and switch. Two better-suited Ubiquiti devices are already owned and
sitting unconfigured: an **ER-X-SFP** router and a **UAP-AC-LR** access point.

The lab is about to grow from one node to six, and to acquire a hypervisor with bridged VMs.
Every layer above the network — Proxmox bridges, k3s node IPs, the API server's TLS SANs,
ingress hostnames — inherits whatever addressing decision gets made now.

## Decision

Four decisions, taken together.

### 1. Move routing, DHCP, and firewall to the ER-X-SFP

The consumer router works. It is being replaced for one reason that matters and several that
help:

**The reason that matters:** a consumer web UI produces no reviewable artifact. EdgeOS produces
`show configuration commands` — a plain-text, diffable, commitable config. That converts the
network layer from an undocumented dependency into version-controlled infrastructure, which is
the entire premise of this repo.

Supporting reasons: real firewall rulesets instead of checkboxes; VLAN-aware switching when
segmentation becomes the next project; and 24V passive PoE output, which removes a power brick.

### 2. Keep `10.0.0.0/24` and gateway `10.0.0.1` unchanged

Non-negotiable constraint on the migration. Holding the subnet and gateway constant reduces the
change from a renumbering project to re-entering six DHCP reservations. Nothing above layer
three has to change.

**Addressing plan:**

| Range | Purpose |
|---|---|
| `10.0.0.1` | Gateway / router / DNS forwarder |
| `10.0.0.10 – .19` | Cluster infrastructure (hypervisor, control plane, workers) |
| `10.0.0.20 – .29` | Reserved for growth |
| `10.0.0.30 – .49` | Appliances (Home Assistant, AP, switches) |
| `10.0.0.100 – .200` | Dynamic DHCP pool (laptops, phones, guests) |

Reservations are **MAC-bound DHCP reservations on the router**, not per-host static
configuration. One place to read the whole address plan, and rebuilding a node from a fresh
image does not require remembering its address. Every reservation binds the **`eth0`** MAC —
see [dual-homed-pi-disable-wifi.md](../runbooks/dual-homed-pi-disable-wifi.md) for why that
qualifier is load-bearing.

### 3. Cluster nodes are wired-only

Onboard WiFi and Bluetooth disabled at device-tree level on every Raspberry Pi that joins the
cluster. Rationale and procedure in the runbook above. Wireless is for client devices, never
for nodes.

### 4. Sequence the migration at Week 4.5 — after Docker, before Proxmox

The network is the bottom layer. Migrating it **before** Proxmox and k3s exist means the only
things that move are six DHCP reservations. Migrating it **after** would move node IPs, the
`k3s-api.home.arpa` TLS SAN, every kubeconfig, and all reservations simultaneously — on a
system where you cannot isolate which layer broke.

It is not Week 1 because Weeks 1-4 need nothing from the router that the AX1800 does not
already provide.

## Open decision — port count

The ER-X-SFP has 5 RJ45 ports. One becomes WAN, leaving **4 LAN ports for 7 devices**
(3 cluster Pis, 1 Home Assistant Pi, OptiPlex, Mac mini, AP). Two options:

| | Option A — AX1800 as AP + switch | Option B — unmanaged switch + UAP-AC-LR |
|---|---|---|
| Cost | $0 | ~$20 |
| WiFi | AX1800 (WiFi 6) | UAP-AC-LR (WiFi 5) |
| Ports gained | 3-4 (AX1800 LAN ports) | 5-8 (dedicated switch) |
| Extra learning | None | UniFi Network Application in Docker Compose — a genuine stateful-volume exercise |
| Downside | UAP-AC-LR stays in a box; consumer AP in the path | Slower WiFi; another box to power |

**Leaning Option B**, because the controller deployment doubles as a Week 3 Compose exercise
with real persistent state, and because a dedicated switch is the shape a production network
actually has. **Not yet decided.** This ADR moves to Accepted when it is.

## Consequences

**Positive**

- Network config becomes a committed, diffable artifact.
- VLAN segmentation becomes possible without buying anything.
- PoE for the AP removes a power adapter (see the hardware note below).
- A documented address plan exists before six nodes need addresses.

**Negative / accepted risk**

- **Single point of failure.** One router, one uplink, one DHCP server, no VRRP, no DHCP
  failover. Accepted for a lab; named here so it is a known gap rather than an oversight.
- **Migration is network-disrupting.** Requires a maintenance window and a rehearsed rollback.
- **Still one flat segment.** No VLANs at cutover. Deliberately deferred, not forgotten —
  management/workload/storage separation is the next network project.
- **EdgeOS is a learning curve.** A committed config is worth more than a familiar UI, but the
  first configuration will be slower than clicking through TP-Link's wizard.

## Hardware verification (done before committing to the PoE plan)

| Fact | Value | Source |
|---|---|---|
| ER-X-SFP PoE output | 24V passive, 2-pair (Pins 4, 5+; 7, 8-), **all 5 RJ45 ports**, 12W/port, 50W total | [UISP EdgeRouter X SFP tech specs](https://techspecs.ui.com/uisp/wired/er-x-sfp) |
| Requires external PSU | 24VDC 2.5A | [EdgeRouter X datasheet](https://dl.ubnt.com/datasheets/edgemax/EdgeRouter_X_DS.pdf) |
| Plain ER-X (non-SFP) | PoE passthrough on `eth4` **only** | [OpenWrt device page](https://openwrt.org/toh/ubiquiti/edgerouter_x_er-x_ka) |
| UAP-AC-LR input | 802.3af/A **or** 24V passive PoE (Pairs 4, 5+; 7, 8 Return) | [UniFi AC AP datasheet](https://dl.ui.com/datasheets/unifi/UniFi_AC_APs_DS.pdf) |
| UAP-AC-LR max draw | 6.5W | same |

**Conclusion:** the ER-X-SFP can power the UAP-AC-LR directly. 6.5W against a 12W port budget,
matching voltage and pinout, no injector needed.

⚠️ **Passive PoE does not negotiate.** The router energizes the pairs whether the far end
asked or not, and Gigabit Ethernet uses all four pairs — there is no spare-pair margin.
Therefore: enable PoE output on **exactly one port**, the AP's, and physically label that cable
at both ends. Never on a port serving a Raspberry Pi, the Mac mini, or the OptiPlex.

## Alternatives considered

| Option | Verdict |
|---|---|
| Keep the AX1800 as router | Rejected — no exportable config, no VLANs, no real firewall |
| ER-X-SFP as router, AX1800 retired entirely | Deferred — depends on the Option A/B port decision |
| Renumber to a new subnet during migration | **Rejected** — multiplies blast radius for zero benefit |
| Per-host static IP configuration | Rejected — scatters the address plan across seven hosts |
| pfSense / OPNsense on spare hardware | Rejected — costs the OptiPlex, which Weeks 5-7 need |
| VLAN segmentation at cutover | Deferred — one change at a time; do it as its own project |

## Implementation

Full step-by-step procedure lives in the Week 4.5 build plan; the migration runbook and the
redacted router config will land at:

- `docs/runbooks/edgerouter-migration.md`
- `docs/network/edgerouter-config.md`

⚠️ Before committing router config: redact the WAN address, PPPoE/ISP credentials, WiFi PSKs,
and `system login` password hashes.
