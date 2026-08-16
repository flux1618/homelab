# A Pi showing two IP addresses (dual-homed node) — and disabling onboard WiFi

- **Host:** k3s-worker-01, k3s-worker-02 (Raspberry Pi 5, Ubuntu Server)
- **Date:** 2026-08-16
- **Layer:** network / link layer
- **Severity:** low now, high once Kubernetes exists

## Symptom

The router's DHCP client list showed **two entries per Raspberry Pi** — two different IP
addresses, two different MAC addresses, same physical device. Both were reachable. It was not
obvious which address SSH was actually using, or which one anything else would use.

## Root cause

The Pi was connected to **both** Ethernet and WiFi at the same time. `eth0` and `wlan0` each
requested a DHCP lease and each got one. Nothing was broken — the host was simply
**dual-homed**, with two paths into it and a routing table choosing between them by metric.

Both observed MAC address pairs belong to Raspberry Pi Ltd OUIs, confirming they were the
onboard NICs rather than a USB adapter:

| Host | Observed MAC pair |
|---|---|
| k3s-worker-01 | `D8:3A:DD:C8:88:C4` / `D8:3A:DD:C8:88:C5` |
| k3s-worker-02 | `2C:CF:67:19:73:34` / `2C:CF:67:19:73:35` |

## Why this matters more than it looks

On a single host, dual-homing is a curiosity. On a Kubernetes node it is a real fault:

- The kubelet infers a node IP. If it picks `wlan0`, every pod-to-pod path for that node
  crosses WiFi — variable latency, packet loss, and intermittent `NotReady` flaps.
- DHCP reservations get made against the wrong MAC, so the "static" address silently stops
  applying after a reboot.
- Troubleshooting becomes ambiguous: two candidate paths means every network symptom has two
  possible explanations.

**Analogy:** a patient with two IV lines running and no labels. Nothing is wrong yet, but the
moment you need to know which line delivered what, you can't.

Wired only, for every cluster node. No exceptions.

## Diagnosis — read-only first

```bash
# on the Pi
ip -brief link show      # which interfaces exist and which are UP
ip -4 -brief addr show   # which address sits on which interface
ip route show            # which interface wins the default route, and its metric
echo $SSH_CONNECTION     # 3rd field = the local address THIS session arrived on
```

`$SSH_CONNECTION` is the one that answers "which address am I actually using right now" —
it prints `<client-ip> <client-port> <server-ip> <server-port>`.

## Soft test before committing

Confirm the host stays reachable over Ethernet alone, without a reboot:

```bash
# on the Pi
sudo rfkill block wifi
ip -brief link show      # wlan0 should now be DOWN
```

If SSH survives, Ethernet is genuinely carrying the session. Undo with:

```bash
sudo rfkill unblock wifi
```

⚠️ `rfkill` state can persist across reboot via `systemd-rfkill`, so treat it as a test, not
the fix.

## Fix — disable the radio at the firmware level

```bash
# on the Pi
sudo cp /boot/firmware/config.txt /boot/firmware/config.txt.bak
sudo nano /boot/firmware/config.txt
```

Append:

```ini
dtoverlay=disable-wifi
dtoverlay=disable-bt
```

Then reboot.

The `disable-wifi` overlay is documented as "Disable onboard WLAN on WiFi-capable Raspberry
Pis" in the [Raspberry Pi firmware overlay README](https://github.com/raspberrypi/firmware/blob/master/boot/overlays/README).
On the Pi 5's BCM2712, firmware automatically substitutes the `disable-wifi-pi5` variant, so
the generic name is correct to write — see the
[Raspberry Pi configuration documentation](https://www.raspberrypi.com/documentation/computers/configuration.html).

This removes the interface at device-tree level, which is cleaner than fighting NetworkManager
or masking a service: the OS never sees a wireless device at all.

## Verification

```bash
# on the Pi, after reboot
ip -brief link show
# expect: lo and eth0 only. No wlan0. No bluetooth device.

ip route show default
# expect: exactly one default route, via eth0
```

Then, on the router: delete the now-stale `wlan0` DHCP reservations. On this lab, worker-01's
`wlan0` was reserved at `10.0.0.11` — which collides with the planned `k3s-cp-01` address —
and worker-02's at `10.0.0.33`.

## Rollback

```bash
# on the Pi
sudo cp /boot/firmware/config.txt.bak /boot/firmware/config.txt
sudo reboot
```

Keeping the `.bak` is the point. `/boot/firmware/config.txt` is read before userspace exists,
so a typo here can produce a host that does not come up at all — and the only recovery is
pulling the card or drive and editing it on another machine.

## Detect faster

`ip -brief link show` on every node, as a line in the provisioning checklist. Two UP
interfaces on a node that is supposed to be wired is a one-second read.

## Automate

- Set `dtoverlay=disable-wifi` in the Ubuntu **cloud-init** template or the imaging step, so
  no node is ever born dual-homed.
- Encode it in the Ansible base role alongside swap-off and NTP, so it is enforced rather
  than remembered.
- Add a pre-join assertion before any `k3s` install: fail loudly if more than one non-loopback
  interface is UP.

## Production comparison

In production, servers **are** deliberately multi-homed — separate management, storage, and
workload networks, often bonded (LACP) for redundancy. The difference is that those paths are
designed, documented, and explicitly bound: the kubelet gets `--node-ip` set, not inferred.
The homelab shortcut is not "one NIC," it's "no accidental second path." Disable what you
didn't plan for.

## Interview framing

**"A Kubernetes node keeps flapping between Ready and NotReady. Where do you start?"**

"Bottom-up, and I'd start at the link layer before I touch the control plane. `ip -brief link
show` and `ip route show` on the node — I've seen a node come up dual-homed on Ethernet and
WiFi, where the kubelet inferred the wireless address and every pod path inherited WiFi's
latency and loss. The tell is two UP interfaces and a default route you didn't choose.

The immediate fix is disabling the unplanned interface, but the real fix is upstream: set
`--node-ip` explicitly rather than letting kubelet infer it, disable onboard radios in the
image template so no node is born that way, and assert on interface count before join. That
turns a class of intermittent, hard-to-reproduce failures into something that can't happen."
