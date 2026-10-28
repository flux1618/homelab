# Network inventory

Last audited: 2026-10-28 (live read-only audit of EdgeRouter and fluxcp)

## Addressing

- LAN: `10.0.0.0/24`, gateway `10.0.0.1` (EdgeRouter `switch0`)
- DHCP authority: EdgeRouter ER-X-SFP, shared-network `LAN`, pool `10.0.0.38 - 10.0.0.243`, lease 86400 s
- DNS handed to clients: `10.0.0.225` (AdGuard Home on `fluxcp`)
- Assignment method: MAC-bound DHCP reservations on the router; no static IPs on hosts.

## Router

| Property | Value |
|---|---|
| Model | Ubiquiti EdgeRouter X SFP 6-Port |
| Hostname | EdgeRouter-X-SFP-6-Port |
| Firmware | EdgeOS v3.0.1, build 5862409 (2025-09-24) |
| Uptime at audit | 12 days |
| WAN | `eth0`, 192.168.1.65/24 (private address; an upstream gateway sits in front of it) |
| LAN | `switch0` 10.0.0.1/24 bridging `eth1`-`eth4`; vlan-aware disabled |
| SFP | `eth5` link down, unused |

## Physical topology

| Port | Connected to |
|---|---|
| `eth0` | WAN / upstream gateway |
| `eth3` | TP-Link LS1005G unmanaged gigabit switch -> k3s-worker-01, -02, -03 |
| `eth1`, `eth2`, `eth4` | Not yet recorded (fluxcp and UAP-AC-LR are among them) |
| `eth5` (SFP) | Unused |

The `show poe status` command is invalid on EdgeOS; PoE state per port has NOT been captured. Passive 24V PoE does not negotiate: it must be enabled only on the port feeding the UAP-AC-LR, never on the LS1005G uplink (`eth3`) or any Pi/PC port.

## DHCP static mappings (as configured on the router)

| Name | IP | MAC | Interface bound | Audit |
|---|---|---|---|---|
| UnifyAP (UAP-AC-LR) | 10.0.0.39 | fc:ec:da:31:c1:b6 | eth0 | OK |
| fluxcp | 10.0.0.225 | 68:1d:ef:64:c9:46 | enp2s0 | OK |
| k3s-worker-01 | 10.0.0.12 | 2c:cf:67:83:7f:e2 | eth0 | OK |
| k3s-worker-02 | 10.0.0.13 | 2c:cf:67:19:73:35 | **wlan0** | **WRONG - should be 2c:cf:67:19:73:34 (eth0)** |
| k3s-worker-03 | 10.0.0.14 | 88:a2:9e:ef:9d:e5 | eth0 | OK |

Fix is deferred to the Ansible baseline run (see LAB.md). Planned, not yet applied: change the worker-02 mapping MAC to 2c:cf:67:19:73:34 and disable wlan0 on the node.

## Range plan vs reality

| Range | Planned purpose | Reality |
|---|---|---|
| 10.0.0.1 | Gateway | Matches |
| 10.0.0.10 - .19 | Cluster infrastructure | Pis at .12-.14; .10/.11 now unused (OptiPlex retired) |
| 10.0.0.30 - .49 | Appliances | UAP-AC-LR at .39 sits inside the dynamic pool; `ha-01` planned at .30 |
| DHCP pool | `.100 - .200` (old plan) | Actual pool is `.38 - .243`; fluxcp (.225) and UAP (.39) are reservations inside it |

The pool overlaps the appliance block. Consider shrinking the pool to `.100 - .200` so it matches the plan.

## Other devices

| Device | Role | Notes |
|---|---|---|
| TP-Link LS1005G | Unmanaged 5-port gigabit switch on `eth3` | Aggregates the three Pis |
| Ubiquiti UAP-AC-LR | Wi-Fi AP at 10.0.0.39 | Power method (passive PoE vs injector) unverified |
| TP-Link Archer AX1800 | Former router | Not in the path per audit; role unconfirmed |

## Open items

- [ ] Re-bind worker-02 reservation to eth0 MAC and disable wlan0 (via Ansible)
- [ ] Capture PoE state: `show configuration commands | match poe`
- [ ] Record what is on eth1, eth2, eth4
- [ ] Decide on shrinking DHCP pool to match range plan
- [ ] Decide fate of AX1800 and the stale ghost leases (.229, .233, .76)
- [ ] Verify the ER-X-SFP firewall ruleset and WAN type
