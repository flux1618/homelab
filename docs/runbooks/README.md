# Runbooks

Symptom → root cause → fix entries from real breakage in this lab. Every entry
follows the same five-line core so it stays scannable at 3 a.m.:

1. **Symptom** — what I actually saw
2. **Root cause** — what was really wrong
3. **Fix** — the commands that resolved it
4. **Detect faster** — the one command that would have shortened diagnosis
5. **Automate** — how this stops recurring

## Index

| Entry | Layer | Host |
|---|---|---|
| [apt-cache-lock.md](apt-cache-lock.md) | Guest OS / packaging | k3s-worker-01 |
| [pi5-no-hdmi-output.md](pi5-no-hdmi-output.md) | Hardware / display | k3s-worker-01 |
| [ssh-permission-denied-publickey.md](ssh-permission-denied-publickey.md) | Access | MacBook → k3s-worker-01 |
| [github-ssh-remote-failures.md](github-ssh-remote-failures.md) | Access / Git | MacBook → GitHub |
| [host-discovery-mdns.md](host-discovery-mdns.md) | Network / DNS | k3s-worker-01 |
| [ufw-enable-without-lockout.md](ufw-enable-without-lockout.md) | Network / firewall | k3s-worker-01 |
| [docker-socket-permission-denied.md](docker-socket-permission-denied.md) | Container runtime / permissions | k3s-worker-01 |
| [dual-homed-pi-disable-wifi.md](dual-homed-pi-disable-wifi.md) | Network / link layer | k3s-worker-01, k3s-worker-02 |

## Conventions

- Private RFC1918 addresses are safe to record here. Public IPs, tokens, kubeconfigs,
  and anything starting with `-----BEGIN` are not.
- State which host each command runs on.
- Record the software version — several of these are version-sensitive.
