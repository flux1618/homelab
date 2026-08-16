# Network inventory

Created: 2026-08-16

## Addressing plan

- LAN: `10.0.0.0/24`
- Gateway / DNS forwarder: `10.0.0.1`
- Assignment method: **MAC-bound DHCP reservations on the router.** Never static IPs on hosts.
- Reservations bind the **`eth0`** MAC only — see
  [dual-homed-pi-disable-wifi.md](../runbooks/dual-homed-pi-disable-wifi.md).

| Range | Purpose |
|---|---|
| `10.0.0.1` | Router / gateway / DNS |
| `10.0.0.10 – .19` | Cluster infrastructure |
| `10.0.0.20 – .29` | Reserved for growth |
| `10.0.0.30 – .49` | Appliances |
| `10.0.0.100 – .200` | Dynamic DHCP pool |

## Planned reservations

| Host | Address | Role | Status |
|---|---|---|---|
| `er-x-01` | 10.0.0.1 | Router, DHCP, firewall, DNS forwarder | Not configured |
| `pve-01` | 10.0.0.10 | Proxmox VE host | Hardware arrives 2026-08-20 |
| `k3s-cp-01` | 10.0.0.11 | k3s server (VM on pve-01) | Planned |
| `k3s-worker-01` | 10.0.0.12 | k3s agent (arm64) | Live |
| `k3s-worker-02` | 10.0.0.13 | k3s agent, storage node (NVMe) | Live, undocumented |
| `k3s-worker-03` | 10.0.0.14 | k3s agent (arm64) | Boxed |
| `ha-01` | 10.0.0.30 | Home Assistant OS, outside cluster | Boxed |

## Network hardware

| Device | Role today | Role after migration |
|---|---|---|
| TP-Link Archer AX1800 | Router, DHCP authority, WiFi, switch | AP + switch only, or retired |
| Ubiquiti ER-X-SFP | Unconfigured | Router, DHCP, firewall, DNS forwarder |
| Ubiquiti UAP-AC-LR | Unconfigured | WiFi, PoE-powered from the ER-X-SFP |

**Current DHCP authority is the AX1800.** The EdgeRouter migration is planned for the Week 4.5
window — see [ADR 0002](../decisions/0002-network-architecture.md).

## ER-X-SFP hardware facts (verified against vendor documentation)

| Property | Value | Source |
|---|---|---|
| RJ45 ports | 5 (`eth0`–`eth4`) plus 1× 1G SFP | [UISP EdgeRouter X SFP tech specs](https://techspecs.ui.com/uisp/wired/er-x-sfp) |
| PoE output | 24V passive, 2-pair (Pins 4, 5+; 7, 8-) | same |
| PoE-capable ports | **All 5 RJ45 ports** on the SFP variant | [ER-X-SFP quick start guide](https://www.alltrade.co.uk/public/product/er-x-sfp-1-pdf.pdf) |
| PoE budget | 12W per port, 50W total | [EdgeRouter X datasheet](https://dl.ubnt.com/datasheets/edgemax/EdgeRouter_X_DS.pdf) |
| PSU required for PoE output | External 24VDC 2.5A | same |
| Plain ER-X (non-SFP) | PoE passthrough on `eth4` only — **different device** | [OpenWrt device page](https://openwrt.org/toh/ubiquiti/edgerouter_x_er-x_ka) |

## UAP-AC-LR power

| Property | Value | Source |
|---|---|---|
| Accepted input | 802.3af/A PoE **or** 24V passive PoE (Pairs 4, 5+; 7, 8 Return) | [UniFi AC AP datasheet](https://dl.ui.com/datasheets/unifi/UniFi_AC_APs_DS.pdf) |
| Max power draw | 6.5W | same |

**Conclusion:** the ER-X-SFP can power the UAP-AC-LR directly. Matching voltage and pinout,
6.5W against a 12W port budget, no injector required.

⚠️ **Passive PoE does not negotiate.** Enable PoE output on **exactly one port** — the AP's.
Never on a port serving a Raspberry Pi, the Mac mini, or the OptiPlex. Gigabit uses all four
pairs, so there is no spare-pair safety margin. **Label the AP cable physically at both ends.**

## Port budget problem (open)

5 RJ45 ports − 1 WAN = **4 LAN ports for 7 devices.** Two options, decision pending — see the
Open decision section of [ADR 0002](../decisions/0002-network-architecture.md).

## Observed MAC addresses

Both pairs are Raspberry Pi Ltd OUIs, confirming onboard NICs rather than USB adapters. Which
member of each pair is `eth0` vs `wlan0` is **not yet resolved** — run
`ip -brief link show` on each node before entering reservations.

| Host | Observed pair |
|---|---|
| k3s-worker-01 | `D8:3A:DD:C8:88:C4` / `D8:3A:DD:C8:88:C5` |
| k3s-worker-02 | `2C:CF:67:19:73:34` / `2C:CF:67:19:73:35` |

## Open items

- [ ] Capture `show version` on the ER-X-SFP and update its firmware
- [ ] Decide port-count Option A vs B; move ADR 0002 to Accepted
- [ ] Resolve `eth0` vs `wlan0` per node, then bind reservations to `eth0` only
- [ ] Delete stale `wlan0` reservations on the AX1800: worker-01 → `10.0.0.11`
      (collides with planned `k3s-cp-01`), worker-02 → `10.0.0.33`
- [ ] Confirm WAN type (DHCP vs PPPoE) and record the AX1800 WAN MAC for cloning
- [ ] Confirm `k3s-worker-02`'s current address before moving it to `.13`
