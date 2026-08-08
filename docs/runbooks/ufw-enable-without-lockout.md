# Enabling UFW without locking yourself out

- **Host:** k3s-worker-01 (Raspberry Pi 5, Ubuntu Server 26.04 LTS)
- **Date:** 2026-08-07
- **Layer:** network / host firewall
- **Severity:** high risk during the change, low after

## Decision

UFW **enabled**: default-deny inbound, allow outbound, SSH permitted only from the LAN.

Rationale: the router blocks inbound from the internet but does nothing about lateral
movement. On a flat home LAN, a compromised IoT device can reach every port on the Pi.
A host firewall is defence in depth at essentially zero cost, and it is the answer an
interviewer expects.

## Procedure

```bash
# on k3s-worker-01
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.0.0.0/24 to any port 22 proto tcp comment 'SSH from LAN'
sudo ufw enable
sudo ufw status verbose
```

Scoping SSH to the subnet is better practice than a bare `ufw allow ssh` —
allow-listing management access to trusted networks is standard hardening.

Expected:

```
Status: active
Default: deny (incoming), allow (outgoing), deny (routed)
10.0.0.0/24 22/tcp   ALLOW IN    Anywhere    # SSH from LAN
```

## The lockout rule (non-negotiable)

1. Add the SSH allow rule **before** `ufw enable`.
2. Keep the current session open.
3. Open a **second** session from a new terminal and confirm it authenticates.
4. Only then close the first.

Rollback: `sudo ufw disable`.

Get the CIDR from the machine rather than guessing — a wrong subnet means the rule
matches nothing and SSH dies at `ufw enable`:

```bash
ip -4 -brief addr show eth0        # e.g. 10.0.0.12/24  → 10.0.0.0/24
ip -4 route show dev eth0 | head -1  # prints the subnet directly
```

## Known future conflict 1 — Docker bypasses UFW

Docker manipulates iptables directly and its rules are evaluated **before** UFW's.
Publishing a port with `-p 8080:80` exposes it to the whole LAN even while
`ufw status` reports deny. This is a notorious trap.

Interim handling: bind published ports to loopback explicitly.

```bash
docker run -p 127.0.0.1:8080:80 ...
```

Then put a reverse proxy in front rather than trusting UFW to protect containers.

Detection:

```bash
sudo ss -tlnp     # services listening on 0.0.0.0 that ufw never authorized
```

## Known future conflict 2 — k3s

UFW will interfere with pod and service networking unless rules exist for the cluster
CIDRs and NodePorts. Symptom presents as mysterious broken DNS inside pods. That is a
configuration task, not a reason to disable the firewall — write the rules deliberately
when k3s is installed.

## What breaks in the real world

- **Lockout from a wrong subnet** in the allow rule. Symptom: existing session fine, new
  connections time out. Only fix is physical console.
- **False sense of security** from UFW while Docker quietly publishes ports.
- **Silent rule creep** as services are added and old rules never removed. Audit
  periodically with `sudo ufw status numbered`.

## Production comparison

In a real environment UFW is never hand-edited. Host firewalls come from configuration
management, network segmentation does the heavy lifting via VLANs, and workload traffic
control moves to Kubernetes NetworkPolicy. The flat LAN is the homelab shortcut here;
moving the lab onto its own VLAN on the ER-X is a worthwhile later exercise and a strong
resume line.

## Interview framing

**"Do you bother with host firewalls when you're already behind a perimeter firewall?"**

"Yes — perimeter controls stop north-south traffic but do nothing for east-west. On my
lab nodes I default-deny inbound and allow management access only from the management
subnet. The nuance I'd raise is that Docker writes iptables rules directly and bypasses
UFW, so a published port can be exposed even when the firewall says otherwise. I bind
published ports to loopback and put a reverse proxy in front rather than trusting UFW
alone. At scale I'd enforce this through configuration management and shift
workload-level controls to NetworkPolicy."

## Automate

- Encode these rules in an Ansible role using `community.general.ufw` so node six is
  identical to node one.
- Add a scheduled rollback during risky changes: `sudo sh -c 'sleep 300 && ufw disable' &`
  cancelled once a second session is proven.
- Never expose port 22 to the internet. Remote access goes through Tailscale or
  WireGuard, never a port forward.
