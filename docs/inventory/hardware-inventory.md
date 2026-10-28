# Hardware Inventory - flux1618 Homelab

Authoritative node list from a live read-only audit on 2026-10-28. Supersedes the Dell OptiPlex (`pve-01`), Mac mini, and TP-Link AX1800 assumptions in earlier docs.

Last updated: 2026-10-28
Confirmed by: Bao (live SSH audit)

---

## 1. Compute nodes

| Hostname | Hardware | Arch | RAM / Swap | Storage | OS / Kernel | Role | Wired IP | Status |
|---|---|---|---|---|---|---|---|---|
| `fluxcp` | Firebat F1 (AMD Ryzen 5 7640HS, 6C/12T, AMD-V) | amd64 | 18 GiB / 8 GiB swap | 512 GB NVMe (1G /boot/efi, 74.5G /, 401.4G /home) + 1 TB USB HDD (ntfs, /srv/media) | Ubuntu 26.04.1 LTS / 7.0.0-38-generic | Docker Compose host, LAN DNS, planned k3s control plane | 10.0.0.225 (Tailscale 100.118.38.93) | LIVE, 13 containers |
| `k3s-worker-01` | Raspberry Pi 5 | arm64 | 7.7 GiB / none | 256 GB NVMe (SKHynix HFS256GEJ9X164N) | Ubuntu 26.04.1 LTS / 7.0.0-1020-raspi | k3s agent (not yet joined) | 10.0.0.12 | LIVE |
| `k3s-worker-02` | Raspberry Pi 5 | arm64 | 7.7 GiB / none | 256 GB NVMe (ORICO-J10-2230) | Ubuntu 26.04 LTS / 7.0.0-1016-raspi | k3s agent (not yet joined) | 10.0.0.240 on eth0 (reservation .13 bound to wlan0) | LIVE, needs fixes |
| `k3s-worker-03` | Raspberry Pi 5 | arm64 | 7.7 GiB / none | 256 GB NVMe (SKHynix HFS256GEJ9X164N) | Ubuntu 26.04.1 LTS / 7.0.0-1017-raspi | k3s agent (not yet joined) | 10.0.0.14 | LIVE |
| `ha-01` | Raspberry Pi 5 (4 GB, repaired, as-is) | arm64 | 4 GB | TBD | Home Assistant OS (planned) | Home Assistant, outside cluster | 10.0.0.30 (planned) | Boxed |

Retired from the lab: Dell OptiPlex (`pve-01`) and Mac mini.

Notes:
- All three Pis boot and run from NVMe (238G ext4 root). Any worker can host persistent volumes.
- Pi model, M.2 HAT and NVMe-only boot are inferred from the Cortex-A76, 7.7 GiB RAM and nvme0n1 root. Confirm board/HAT details on the next physical check.
- `systemd-timesyncd` reported inactive on all three Pis; time sync is NOT yet verified on them (it is verified on `fluxcp`).

---

## 2. Interfaces and MACs

| Host | Wired | Wired MAC | Wired IP | Wireless | Wireless MAC | Wireless state |
|---|---|---|---|---|---|---|
| `fluxcp` | enp2s0 | 68:1d:ef:64:c9:46 | 10.0.0.225 | wlp3s0 | 44:90:ba:32:6e:bb | DOWN |
| `k3s-worker-01` | eth0 | 2c:cf:67:83:7f:e2 | 10.0.0.12 | wlan0 | 2c:cf:67:83:7f:e3 | DOWN (NO-CARRIER) |
| `k3s-worker-02` | eth0 | 2c:cf:67:19:73:34 | 10.0.0.240 | wlan0 | 2c:cf:67:19:73:35 | UP at 10.0.0.13 |
| `k3s-worker-03` | eth0 | 88:a2:9e:ef:9d:e5 | 10.0.0.14 | wlan0 | 88:a2:9e:ef:9d:e6 | DOWN (NO-CARRIER) |

The earlier MAC `D8:3A:DD:C8:88:C4/C5` recorded for worker-01 no longer matches; the board now reports 2c:cf:67:83:7f:e2/e3.

---

## 3. Workloads on fluxcp (Docker, up 12 days at audit time)

| Group | Containers | Bindings |
|---|---|---|
| DNS | adguard-home | 10.0.0.225:53 tcp/udp, :3000, :8080 |
| Paperless-ngx | paperless-webserver-1, paperless-db-1 (postgres:18), paperless-broker-1 (valkey:9-alpine), paperless-tika-1, paperless-gotenberg-1, paperless-telegram-bot-1 | 0.0.0.0:8000 |
| Media / VPN | jellyfin, gluetun, deluge (via gluetun) | 0.0.0.0:8096, 10.0.0.225:8112 |
| Ops / UI | dockhand v1.0.49, heimdall, webtop | 10.0.0.225:3001, 0.0.0.0:8085, 0.0.0.0:3011 |

Several ports bind 0.0.0.0 (paperless, jellyfin, heimdall, webtop). Review exposure before any port-forward or reverse proxy work. `fluxcp` is the LAN DNS server: if it goes down, DNS for every client goes down.

---

## 4. Findings from the 2026-10-28 audit

1. `k3s-worker-02` is dual-homed: eth0 at 10.0.0.240 (unreserved pool lease), wlan0 UP at 10.0.0.13. The router reservation for .13 is bound to the wlan0 MAC (…73:35), not eth0 (…73:34).
2. `k3s-worker-02` runs Ubuntu 26.04 LTS (kernel 1016) while -01 and -03 run 26.04.1; it also has desktop snaps installed (firefox, thunderbird, gnome-46-2404, mesa-2404) and uses about 800 MiB RAM at idle versus about 450 MiB on the others.
3. `fluxcp` root (/) is 74.5G with 33.5G free; /home is 401.4G. Docker data under /var/lib/docker lives on /. Watch root usage.
4. `fluxcp` has 8 GiB swap enabled. Swap must be off if k3s ever runs on this host.
5. Time sync on the three Pis is unverified.
