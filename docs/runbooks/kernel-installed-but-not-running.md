# Node patched on paper but running an old kernel (reboot never happened)

- **Symptom:** `apt full-upgrade` reported nothing to do on k3s-worker-02, yet `uname -r` showed `7.0.0-1016-raspi` while `-01` ran `1020`. `/var/run/reboot-required.pkgs` listed `linux-image-7.0.0-1020-raspi`, `libc6`, `linux-base`.
- **Root cause:** `unattended-upgrades` installed the new kernel and libc, but nothing ever rebooted the node, so the running kernel and the installed kernel diverged. Package state said "current"; the kernel in memory was weeks old.
- **Fix:** `ansible/playbooks/patch.yml` (serial: 1) checks `/var/run/reboot-required` after upgrading and reboots only when it exists, then waits for SSH and re-reads facts. Report line confirmed `1016 -> 1020`.
- **Faster detection:** compare `uname -r` against `ls /boot/vmlinuz-* | sort -V | tail -1`, or just `ls /var/run/reboot-required`. Pending-reboot should be a monitored metric (node-exporter textfile collector later).
- **Automate:** run patch.yml on a schedule; once k3s is live, add drain/uncordon around the reboot.
