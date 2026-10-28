# SSH password login still enabled despite hardening (cloud-init drop-in override)

- **Symptom:** Docs and `/etc/ssh/sshd_config` said key-only, but `ssh -o PubkeyAuthentication=no flux@<pi>` still offered a password prompt on all three k3s workers (Ubuntu 26.04, Pi 5).
- **Root cause:** `/etc/ssh/sshd_config.d/50-cloud-init.conf` sets `PasswordAuthentication yes`. Ubuntu's `sshd_config` includes `sshd_config.d/*.conf` at the top, and sshd keeps the first value it reads, so the drop-in silently won.
- **Fix:** `ansible/playbooks/baseline.yml` installs `/etc/ssh/sshd_config.d/00-hardening.conf` (sorts before `50-`, so it wins) with `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`, validates with `sshd -t`, then reloads `ssh`. Verified with `sshd -T` on each node.
- **Faster detection:** Never trust the file. Check the effective config: `sudo sshd -T | grep -i passwordauthentication`.
- **Automate:** Baseline playbook is idempotent (`changed=0` on re-run); re-run it on every new node and after any manual sshd edit.
