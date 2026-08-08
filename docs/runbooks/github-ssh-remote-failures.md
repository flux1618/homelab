# Git push to GitHub fails: three distinct causes

- **Hosts:** MacBook → github.com/flux1618/homelab
- **Date:** 2026-08-07
- **Layer:** access / Git remote
- **Severity:** blocking for artifact publishing

Three separate failures were hit in sequence. They produce different errors and have
different fixes — telling them apart in ten seconds is the actual skill here.

## Triage: two commands distinguish all three

```bash
# on MacBook
git remote -v            # is the URL well formed and pointing at the right repo?
ssh -T git@github.com    # is the SSH layer authenticating at all?
```

---

## Cause 1 — commits exist locally but nothing appears on GitHub

**Symptom:** `git commit` succeeds; the GitHub web UI shows an empty repo.

**Root cause:** `git commit` writes only to the local repository. No remote was
configured, so nothing left the machine.

**Fix:**

```bash
cd ~/homelab
git log --oneline          # commits exist locally
git remote -v              # prints nothing → no remote
git remote add origin git@github.com:flux1618/homelab.git
git branch -M main
git push -u origin main    # -u sets upstream; later pushes are just `git push`
```

Create the remote repo first at github.com/new — **Private**, and **do not**
initialize with README/.gitignore/license. An initialized remote creates divergent
histories and forces an awkward merge.

**Mental model:** `commit` is saving the chart; `push` is uploading it to the hospital
system. Local work is invisible to everyone until the second step.

---

## Cause 2 — malformed SSH URL

**Symptom:**

```
fatal: 'git@github.com/flux1618/homelab.git' does not appear to be a git repository
fatal: Could not read from remote repository.
```

**Root cause:** SSH URLs use `host:path`. A slash after the hostname makes the string
unparseable as an SSH target.

```
wrong:  git@github.com/flux1618/homelab.git
right:  git@github.com:flux1618/homelab.git
                      ^ colon
```

**Fix:**

```bash
git remote set-url origin git@github.com:flux1618/homelab.git
git remote -v
```

---

## Cause 3 — key not registered with GitHub

**Symptom:**

```
git@github.com: Permission denied (publickey).
```

**Evidence:**

```bash
ssh -vT git@github.com 2>&1 | grep -Ei 'offering|identity file|Authentications|Server accepts'
ls -l ~/.ssh/id_ed25519_github ~/.ssh/id_ed25519_github.pub
```

- No `Offering` line → the key file does not exist or is not mapped in `~/.ssh/config`.
- `Offering public key: ~/.ssh/id_ed25519_github` then denied → the key is not on the
  GitHub account.

**Fix — generate if missing:**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github -C "macbook-github-2026-08"
```

**Register the public key:**

```bash
pbcopy < ~/.ssh/id_ed25519_github.pub
```

github.com/settings/keys → New SSH key → key type **Authentication Key** → paste → Add.

Gotcha worth remembering: adding it as a **Signing** key instead of an Authentication
key still produces `Permission denied (publickey)`, and it is maddening to spot.

**Load into the agent:**

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_github
ssh-add -l
```

**Confirm config mapping:**

```bash
ssh -G github.com | grep -i identityfile   # expect ~/.ssh/id_ed25519_github
```

```
Host github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  IdentitiesOnly yes
  AddKeysToAgent yes
  UseKeychain yes
```

---

## Verify the push actually landed

```bash
git fetch origin
git log origin/main --oneline   # proof the commit is on the remote, not just local
git status                      # expect "up to date with 'origin/main'"
```

## Two silent traps

- **Branch mismatch** — pushed `master` while GitHub displays `main` by default, so the
  files look absent. Check the branch dropdown.
- **Email mismatch** — commits authored with an email not verified on the GitHub account
  appear in the repo but do not link to the profile or contribution graph. For a portfolio
  meant to show months of steady work, this matters.

```bash
git config user.email    # must be a verified address on the GitHub account
```

## Do NOT do

Search results push personal access tokens in the remote URL, or switching to HTTPS.
Both work; both are worse. A token pasted into a URL is written to `.git/config` in
plaintext — a secret on disk, and exactly the habit an interviewer flags. SSH is the
professional default and is required anyway for k3s, Ansible, and Argo CD.

## Detect faster

`git remote -v` plus `ssh -T git@github.com` separates malformed URL, missing repo,
and unauthorized key in about ten seconds.

## Automate

`.gitignore` committed on day one — secrets in Git history are effectively permanent
without rewriting history:

```
*.key
*.pem
id_*
!id_*.pub
kubeconfig
*.kubeconfig
.env
*.tfstate
*.tfstate.backup
.DS_Store
```

Next hardening step: enable commit signing and add a `gitleaks` pre-commit hook.
