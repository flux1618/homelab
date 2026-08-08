# ADR 0001 — SSH key algorithm and naming strategy

- **Status:** Accepted
- **Date:** 2026-08-07
- **Scope:** all lab hosts and code-hosting access

## Decision

Use **Ed25519** keys, scoped **one key per client device per purpose**, named
`id_ed25519_<purpose>`, with a comment identifying machine and mint date.

```bash
# on MacBook
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_homelab -C "macbook-homelab-2026-08"
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github  -C "macbook-github-2026-08"
```

## Why Ed25519

1. **Strong security at a fraction of the size.** Elliptic-curve (EdDSA) rather than
   RSA's large primes. A 256-bit Ed25519 key is practically comparable to RSA-3000+
   and is the current professional default.
2. **Faster key generation, signing, and verification.** Nuance worth stating
   precisely: the key algorithm affects authentication and session setup, not bulk
   transfer speed — throughput depends on the session cipher, configured separately.
3. **Safer generation.** Any random number is a valid Ed25519 private key. RSA must
   generate two large primes, and that code is complex enough that implementations
   have historically produced weak keys. Fewer moving parts, fewer silent failures.
4. **Short enough to handle.** One line that survives `pbcopy` into Raspberry Pi
   Imager, `authorized_keys`, or a GitHub form without wrapping or truncation.

## Alternatives considered

| Key type | Practical security | Length | Verdict |
|---|---|---|---|
| Ed25519 | Excellent | ~80 chars | **Chosen** |
| RSA 4096 | Excellent | Very long | Fallback for legacy gear that cannot do Ed25519 |
| RSA 2048 | Acceptable but dated | Long | Rejected — avoid for new keys |
| ECDSA | Good | Short | Rejected — largely superseded |
| RSA 1024 / DSA | Broken | — | Never |

## Naming convention

`id_ed25519_<purpose>` (suffix style) keeps keys grouped alphabetically in `~/.ssh/`.
Prefix style (`homelab_ed25519`) is equally defensible. The wrong answers are `key1`,
`mykey`, or one key reused everywhere.

**Scoping rule:** one key per **client device and context**, never one key per server.
The MacBook's homelab key authorizes to every lab node — that is correct. Private keys
are **never** copied between machines; each machine gets its own key and both public
keys go into `authorized_keys`.

**Analogy:** each clinician gets their own badge, and the badge opens many doors. You
do not issue a separate badge per door, and you never share a badge.

## Consequences

Non-default filenames are not found automatically, so `~/.ssh/config` must map them:

```
Host k3s-* pi optiplex macmini
  User <LAB_USER>
  IdentityFile ~/.ssh/id_ed25519_homelab
  IdentitiesOnly yes
  AddKeysToAgent yes
  UseKeychain yes

Host github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  IdentitiesOnly yes
  AddKeysToAgent yes
  UseKeychain yes
```

- `IdentitiesOnly yes` — stops SSH offering every known key to every host, which
  otherwise trips `MaxAuthTries` limits and causes confusing lockouts.
- `UseKeychain yes` — macOS-specific; stores the passphrase in Keychain.

Private keys carry a passphrase held in the macOS Keychain. A passphrase-less key is a
bearer token sitting on a laptop.

## Recognizing the two files

| File | Content | Location |
|---|---|---|
| `id_ed25519_homelab` | starts `-----BEGIN OPENSSH PRIVATE KEY-----` | MacBook only, forever |
| `id_ed25519_homelab.pub` | one line, starts `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5...` | pasted into `authorized_keys`, Imager, GitHub |

The public key is the name badge — handed out freely. The private key is the credential.
A multi-line `BEGIN`/`END` block means you are holding the credential, not the badge.

```bash
ssh-keygen -l -f ~/.ssh/id_ed25519_homelab.pub   # validity check before use
ssh -G k3s-worker-01 | grep -i identityfile      # which key SSH *would* use
```

## Verification / file permissions

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_homelab ~/.ssh/config
chmod 644 ~/.ssh/id_ed25519_homelab.pub
```

SSH silently refuses group- or world-readable private keys.

## Rotation

Mint date is embedded in the key comment. **Review August 2027.** The comment lands in
`authorized_keys` on every server, so it should answer "which machine is this, and how
old is this credential?" — the question actually asked during an audit.

## Open items

- [ ] Separate key for the Mac mini when it joins the lab (never copy private keys)
- [ ] Consider SSH certificate authority once node count exceeds ~5
- [ ] Enable Git commit signing
