# k3s-worker-01

Role: Kubernetes worker node (ARM64), lab
Created: 2026-08-07

## Hardware
- Device: Raspberry Pi 5, 8 GB RAM
- Architecture: aarch64
- Storage: microSD (TODO: migrate to NVMe before stateful workloads)
- Power: <PSU model>

## OS
- Ubuntu Server 26.04 LTS (64-bit ARM)
- Kernel: <output of `uname -r`>
- Hostname: k3s-worker-01

## Network
- Interface: eth0 (wired)
- IP: 10.0.0.12 (TODO: DHCP reservation on ER-X)
- MAC: <output of `ip -brief link show eth0`>
- LAN CIDR: 10.0.0.0/24
- Gateway: <output of `ip route show default`>

## Access
- SSH: key-only, password auth disabled
- Key: id_ed25519_homelab (MacBook)
- Listener unit: ssh.socket

## Firewall
- UFW enabled, default deny incoming / allow outgoing
- 22/tcp allowed from 10.0.0.0/24 only

## Open items
- [ ] DHCP reservation
- [ ] Confirm cloud-init status = done
- [ ] Verify time sync (timedatectl)
