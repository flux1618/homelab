# SSH: Permission denied (publickey) to a lab node

- **Hosts:** MacBook → k3s-worker-01 (10.0.0.12)
- **Date:** 2026-08-07
- **Layer:** access / authentication
- **Severity:** blocking — password auth is disabled, so this is a hard lockout

## Symptom

```
$ ssh <LAB_USER>@10.0.0.12
<LAB_USER>@10.0.0.12: Permission denied (publickey).
```

## Evidence first — one command

```bash
# on MacBook
ssh -v <LAB_USER>@10.0.0.12 2>&1 | grep -E 'Offering|Authentications|Trying private|Will attempt'
```

Two lines carry the diagnosis:

- `Authentications that can continue: publickey` → the server refuses passwords;
  key auth is the only path.
- `Offering public key: ...` → which key the client actually sent.
  - Key **not** listed → client-side problem.
  - Key listed and still denied → the key is not installed on the server.

## Ranked hypotheses

| # | Hypothesis | Cheapest test |
|---|---|---|
| 1 | Wrong username (typed `pi` or `ubuntu`, Imager created something else) | `ssh -i ~/.ssh/id_ed25519_homelab <LAB_USER>@10.0.0.12` |
| 2 | Custom-named key never offered (not default `id_ed25519`, no config entry) | the `-i` flag above; `ssh -G <host> \| grep -i identityfile` |
| 3 | Imager/cloud-init never installed the public key | requires console; check `~/.ssh/authorized_keys` on the Pi |

A key cannot authenticate into a different user's home directory — hypothesis 1 is
both the most common and the cheapest to eliminate.

## Client-side fixes

```bash
# on MacBook
ls -l ~/.ssh/
ssh-add -l
```

SSH silently ignores private keys readable by others:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_homelab
chmod 644 ~/.ssh/id_ed25519_homelab.pub
```

Load into the agent (macOS Keychain integration):

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_homelab
ssh-add -l
```

Because the key is not the default filename, map it explicitly in `~/.ssh/config`:

```
Host k3s-* pi
  HostName 10.0.0.12
  User <LAB_USER>
  IdentityFile ~/.ssh/id_ed25519_homelab
  IdentitiesOnly yes
  AddKeysToAgent yes
  UseKeychain yes
```

`IdentitiesOnly yes` stops SSH offering every known key to every host, which
otherwise trips `MaxAuthTries` and produces this same error.

## Server-side fix (hypothesis 3 — needs console)

There is no network path in when key auth is broken and passwords are disabled.
Attach monitor and keyboard, log in with the Imager-set password:

```bash
# on k3s-worker-01 console
whoami
ls -la ~/.ssh/
cat ~/.ssh/authorized_keys
sudo journalctl -u ssh -n 50 --no-pager
```

The journal states the real reason: bad permissions, no matching key, or no such user.

If `authorized_keys` is missing or empty, install the key:

```bash
# on MacBook — copy this single line
cat ~/.ssh/id_ed25519_homelab.pub
```

```bash
# on k3s-worker-01
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys      # paste the single line
chmod 600 ~/.ssh/authorized_keys
ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys
```

Expected: `drwx------` on the directory, `-rw-------` on the file, both owned by the
login user — **not** root. Root-owned files in a user's `~/.ssh` are a frequent silent
cause.

## Verify

```bash
# on MacBook
ssh <LAB_USER>@10.0.0.12 'hostnamectl; uname -m'
```

Expected: `k3s-worker-01` and `aarch64`.

## Server-side SSH state reference (Ubuntu 24.04/26.04)

```bash
systemctl is-active ssh.socket    # 24.04+ uses ssh.socket, not ssh.service
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

Expected hardened output:

```
permitrootlogin prohibit-password
pubkeyauthentication yes
passwordauthentication no
```

Hardening belongs in a drop-in, not the main config file:

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
EOF
sudo sshd -t && sudo systemctl restart ssh.socket ssh
```

Rollback: `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl restart ssh.socket ssh`

## Do NOT do

- Re-enable password authentication as a shortcut.
- `ssh pi:@address`, `PubkeyAcceptedAlgorithms +ssh-rsa`, or clearing `known_hosts` —
  common forum advice that does not apply. Ed25519 is modern and well supported, and a
  `known_hosts` mismatch produces a completely different, much louder warning.
- `sudo ssh` — makes SSH look for root's keys on the Mac.

## Detect faster

`ssh -v ... | grep Offering` distinguishes all three hypotheses in one command,
before touching any config.

## Automate

- Two-session rule: never close the working session until a second, new session
  authenticates.
- Manage `authorized_keys` with Ansible (`ansible.posix.authorized_key`) so key
  installation is idempotent and never depends on Imager/cloud-init succeeding.
- One key per client device and purpose; never copy private keys between machines.
