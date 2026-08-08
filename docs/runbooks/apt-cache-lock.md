# apt cache lock on first boot

- **Host:** k3s-worker-01 (Raspberry Pi 5, Ubuntu Server 26.04 LTS, ARM64)
- **Date:** 2026-08-07
- **Layer:** guest OS / package management
- **Severity:** cosmetic — no outage, only a blocked operator

## Symptom

```
$ sudo apt full-upgrade -y
Waiting for cache lock: Could not get lock /var/lib/dpkg/lock-frontend...
```

Command appears to hang indefinitely on a freshly imaged system.

## Root cause

`apt`/`dpkg` permit only one process to modify the package database at a time and
enforce that with a lock file. On first boot, Ubuntu Server's `unattended-upgrades`
and the `apt-daily` / `apt-daily-upgrade` systemd timers fire automatically and
start pulling security updates. The manual command is correctly queued behind a
legitimate process — this is the system working as designed, not a stale lock.

## Fix

Wait. Confirm a real process holds the lock first:

```bash
# on k3s-worker-01
pgrep -af '/usr/bin/[u]nattended-upgrade|[a]pt-get|[d]pkg'
```

Watch it drain:

```bash
watch -n 5 'pgrep -af "[a]pt|[d]pkg|[u]nattended-upgrade" || echo FREE'
```

When it prints `FREE`, retry:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

Typical wait on a Pi 5 over microSD: 5–15 minutes. SD cards are poor at random
writes, so this is slower than the same operation on NVMe.

## Detect faster

```bash
sudo lsof /var/lib/dpkg/lock-frontend   # names the PID actually holding the lock
```

One command turns "is it hung or working?" into a definitive answer.

## Verify clean state afterwards

```bash
sudo dpkg --audit          # expect no output
sudo apt list --upgradable # expect none
systemctl --failed         # expect 0 loaded units failed
```

`dpkg --audit` is the check that catches half-configured packages — exactly the
damage caused by deleting locks carelessly.

## Do NOT do this

Forum advice universally suggests `rm /var/lib/dpkg/lock-frontend`. Deleting a lock
while a live `dpkg` transaction is in flight can leave packages half-configured and
genuinely break the system. Killing apt inexpertly does not release the lock properly
either.

Escalate only if there is no CPU usage and no progress for 30+ minutes:

```bash
sudo lsof /var/lib/dpkg/lock-frontend   # 1. identify the real PID
sudo kill <PID>                          # 2. graceful termination first
sudo dpkg --configure -a                 # 3. repair partial state
```

Removing lock files is the last resort, only after confirming nothing holds them.

## Automate

In Ansible, use a retry / wait-for-lock pattern rather than forcing:

```yaml
- name: Update and upgrade packages
  ansible.builtin.apt:
    update_cache: true
    upgrade: full
    lock_timeout: 300   # wait rather than fail on a held lock
  register: apt_result
  retries: 5
  delay: 30
  until: apt_result is succeeded
```

## Related note

`unattended-upgrades` ships installed and enabled on Ubuntu Server, so the
"install unattended-upgrades" baseline step is redundant. Verify instead:

```bash
systemctl is-enabled unattended-upgrades
```

## Interview framing

This exact race is why production automation waits for the package lock instead of
forcing it — a credible, specific detail that shows real hands-on time rather than
tutorial-following.
