# Hostname set but `<host>.local` does not resolve

- **Host:** k3s-worker-01 (Raspberry Pi 5, Ubuntu Server 26.04 LTS)
- **Date:** 2026-08-07
- **Layer:** network / name resolution
- **Severity:** low — cosmetic until it costs you time during an outage

## Symptom

Hostname was set correctly at imaging, but from the MacBook:

```
$ ping -c 3 k3s-worker-01.local
ping: cannot resolve k3s-worker-01.local: Unknown host
```

## Root cause

Ubuntu Server for Raspberry Pi does not install `avahi-daemon`. The hostname exists on
the machine, but nothing broadcasts it to the network, so no other host can resolve it.
Raspberry Pi OS ships Avahi by default, which is why most tutorials assume `.local` works.

**Analogy:** the patient has a wristband with their name on it, but nobody entered them
into the census. The identity is real; the lookup is not.

## Finding the node without mDNS

**Option 1 — router DHCP lease table (most reliable).** The Pi appears in the ER-X leases
with the hostname set in Imager, next to its IP.

**Option 2 — ARP from the Mac:**

```bash
# on MacBook
ping -c 3 <SUBNET>.255      # populate the ARP table first
arp -a | grep -i 'b8:27:eb\|dc:a6:32\|e4:5f:01\|2c:cf:67'
```

Those are Raspberry Pi Foundation MAC prefixes; `2c:cf:67` is common on the Pi 5.

**Option 3 — from the console, if attached:**

```bash
# on k3s-worker-01
hostnamectl        # Static hostname: k3s-worker-01
hostname -I        # capital I — prints IP addresses, not the name
ip -4 -brief addr show eth0
```

## Fix — install Avahi (optional, interim)

```bash
# on k3s-worker-01
sudo apt update && sudo apt install -y avahi-daemon
systemctl is-active avahi-daemon
```

`ssh <LAB_USER>@k3s-worker-01.local` then works from macOS.

This is an interim convenience, not the real answer. mDNS does not cross subnets and
will not survive VLAN segmentation later. Real internal DNS is the durable fix.

## If the hostname itself is wrong

If it shows `ubuntu`, the Imager customization did not apply:

```bash
# on k3s-worker-01
sudo hostnamectl set-hostname k3s-worker-01
sudo sed -i 's/127.0.1.1.*/127.0.1.1\tk3s-worker-01/' /etc/hosts
sudo reboot
```

The `/etc/hosts` edit matters — skipping it makes `sudo` emit
`unable to resolve host` warnings on every command.

## Do this regardless: pin the address

Set a **DHCP reservation on the ER-X** rather than a static IP on the host, so the
router remains the single source of truth for the address plan.

```bash
# on k3s-worker-01 — get the MAC
ip -brief link show eth0
```

Reserve, reboot, confirm:

```bash
ip -4 addr show eth0 | grep inet
```

Rollback: delete the reservation; the host returns to dynamic addressing with no
change on the node itself.

Nodes that change IPs break Kubernetes clusters in tedious ways. Pin before k3s.

## Address plan (fill and keep current)

```
LAN CIDR:              10.0.0.0/24
Gateway (ER-X):        <ip route show default>
DHCP pool range:       <check ER-X>
Reserved for lab nodes: 10.0.0.20–10.0.0.30
Future MetalLB pool:    10.0.0.240–10.0.0.250
k3s-worker-01:          10.0.0.12 (TODO: move into reserved range)
```

Reading CIDR: the suffix is the count of leading bits identifying the network. `/24`
means the first three octets are the network — equivalent to netmask `255.255.255.0`,
the standard home LAN. Derive it directly:

```bash
ip -4 route show dev eth0 | head -1    # first field is the subnet in CIDR form
```

## Detect faster

`ip -4 -brief addr show eth0` on the node, or the router's lease table, answers
"where is it?" faster than any scanning. Reach for the source of truth, not a scan.

## Automate

- DHCP reservations for every lab node, recorded in `docs/inventory/`.
- Later: internal DNS (Pi-hole, Unbound, or CoreDNS) serving a `home.arpa` zone, with
  records under configuration management instead of mDNS guesswork.
- Reserve `k3s-api.home.arpa` as the future control-plane endpoint name.
