# Ansible refused by every host at once: Permission denied (publickey) from fluxcp

- **Symptom:** Minutes after a successful playbook run, `ansible k3s_workers -m ping` and plain `ssh` failed on all three Pis with `Permission denied (publickey)`. New terminal window on fluxcp.
- **Root cause:** The Ansible key (`~/.ssh/id_ed25519_ansible`) is passphrase-protected and had been loaded only into the previous terminal's ssh-agent. `ssh-add -l` in the new shell: "The agent has no identities." `ansible.cfg` did not pin `private_key_file`, so Ansible had only ever worked via the agent. Ansible cannot prompt for a passphrase, so it reports a key failure that looks identical to a server-side lockout.
- **Fix:** `ssh-add ~/.ssh/id_ed25519_ansible`; pinned `private_key_file` and `remote_user = flux` in `ansible.cfg`; added a `Host` block for the Pis in `~/.ssh/config` with `IdentityFile`, `IdentitiesOnly yes`, `AddKeysToAgent yes` so the first manual ssh of a session caches the passphrase.
- **Faster detection:** "All hosts fail at once" points at the client, not the servers. Run `ssh-add -l` and `ssh -v -i <key> -o IdentitiesOnly=yes` before touching any node. `Server accepts key` in the verbose output proves the server side is fine.
- **Automate:** pre-flight target (`make ping`) that runs `ssh-add -l` and refuses to continue on an empty agent.
