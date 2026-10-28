# k3s-worker-01

Role: Kubernetes worker node (ARM64), lab
Created: 2026-08-07
Last audited: 2026-10-28

## Hardware
- Device: Raspberry Pi 5, 8 GB RAM (7.7 GiB usable)
- Architecture: aarch64, 4x Cortex-A76
- Storage: 256 GB NVMe (SKHynix HFS256GEJ9X164N); p1 512M vfat /boot/firmware, p2 238G ext4 /
- Swap: none
- Power: <PSU model>

## OS
- Ubuntu 26.04.1 LTS (arm64)
- Kernel: 7.0.0-1020-raspi
- Hostname: k3s-worker-01

## Network
- Interface: eth0 (wired), MAC 2c:cf:67:83:7f:e2
- IP: 10.0.0.12 (DHCP reservation on the EdgeRouter, verified)
- wlan0: MAC 2c:cf:67:83:7f:e3, state DOWN (no carrier)
- LAN CIDR: 10.0.0.0/24, gateway 10.0.0.1
- Uplink: TP-Link LS1005G -> EdgeRouter eth3

## Access
- SSH: key-only, password auth disabled
- Key: id_ed25519_homelab (MacBook)
- Listener unit: ssh.socket

## Firewall
- UFW active, default deny incoming / allow outgoing
- 22/tcp allowed from 10.0.0.0/24 only

## Services at audit
- k3s, k3s-agent, docker, tailscaled, systemd-timesyncd: inactive

## Open items
- [x] DHCP reservation
- [ ] Verify time sync (timesyncd inactive; check chrony/timedatectl)
- [ ] Confirm cloud-init status = done
- [ ] Confirm wlan0 is disabled at config.txt level (dtoverlay=disable-wifi), not just down
