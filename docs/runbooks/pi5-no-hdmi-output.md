# Raspberry Pi 5: no HDMI output on Ubuntu Server

- **Host:** k3s-worker-01 (Raspberry Pi 5, Ubuntu Server 26.04 LTS, ARM64)
- **Date:** 2026-08-07
- **Layer:** hardware / display (out-of-band access path)
- **Severity:** low if SSH works, critical if it does not — this is the recovery console

## Symptom

Monitor connected to the Pi's micro-HDMI shows nothing. No boot messages, no login
prompt. A text console *should* appear even on a server install, so a black screen is
not "expected headless behaviour."

## Triage: separate "no video" from "not booting"

```bash
# on MacBook
ping -c 3 <RPI5_IP>
ssh <LAB_USER>@<RPI5_IP>
```

- SSH works → node is healthy, this is a display-output problem only.
- SSH fails → the real problem is boot or imaging; video is a symptom, not the disease.

## Ranked hypotheses

1. **Cable or adapter (most likely).** Micro-HDMI adapters frequently fail to pass EDID
   back to the Pi. With no EDID the driver picks no mode and outputs nothing. A
   single-piece micro-HDMI-to-HDMI cable is strongly preferred over adapter stacks.
2. **Wrong HDMI port.** Use the port nearest the USB-C power connector (HDMI0) — it is
   the primary output.
3. **KMS/EDID negotiation failure.** The kernel mode-setting driver is less forgiving of
   missing or malformed EDID than older firmware paths. This is the documented failure
   pattern for Ubuntu Server on Pi 5.

## Cheapest test to isolate cable vs OS

Power off, **remove the SD card**, power on with the monitor attached. A Pi 5 with no
bootable media displays a boot diagnostics screen.

- Diagnostics screen appears → cable and monitor are fine; the issue is OS/driver side.
- Still black → cable, adapter, monitor, PSU, or board. Swap the cable first.

Confirm hotplug detection over SSH:

```bash
# on k3s-worker-01
sudo sh -c 'cat /sys/kernel/debug/dri/?/hdmi0_regs' | grep -i hotplug
```

Zero on both HDMI outputs points at cabling.

## Fix if diagnostics work but Ubuntu stays dark

Force a resolution instead of relying on EDID.

```bash
# on k3s-worker-01, or edit the SD card boot partition from the Mac
sudo nano /boot/firmware/cmdline.txt
```

Append to the **existing single line** — never add a new line:

```
video=HDMI-A-1:1280x720@60D
```

Reboot. If still dark, the documented fallback for Ubuntu Server on Pi 5 is adding
`nomodeset` to that same line, disabling kernel mode setting.

## Rollback

Remove the added parameters from that single line in `/boot/firmware/cmdline.txt`
and reboot.

## Detect faster

Prove the recovery console works *before* touching SSH hardening or the firewall.
A known-good cable + monitor combination, tested once, converts a future lockout from
a reimage into a five-minute fix.

## What breaks in the real world

- **Underpowered PSU** causes erratic boot and display failures. Use the official 27 W
  USB-C PD supply for a Pi 5.
- **Cable poorly seated**, especially inside a case — easy to miss on the Pi end.
- **Locked out with no console** after a firewall or SSH change. This is precisely why
  video is verified before UFW is enabled.

## Production comparison

At work this is the out-of-band management question. A real server has iDRAC, iLO, or
IPMI. "I can only reach the box over the network I might have just broken" is how a
homelab turns into a trip to the closet — and how a datacenter turns into a remote-hands
ticket. Long-term homelab equivalent: a USB-serial console, or a networked PDU/KVM.

## Automate / harden

- Keep a tested rescue path documented per host in `docs/inventory/`.
- Prefer changes that self-revert: for firewall work, schedule a rollback
  (`sleep 300 && ufw disable`) that you cancel once a second session is proven.
