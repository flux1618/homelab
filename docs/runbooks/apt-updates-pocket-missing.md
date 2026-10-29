# Fleet looked fully patched but never received bug-fix updates (missing -updates pocket)

- **Symptom:** `apt full-upgrade` reported nothing to do on all three k3s workers, yet `base-files` was `14ubuntu6` on worker-02 and `14ubuntu6.2` on -01/-03, and `lsb_release` disagreed (`26.04` vs `26.04.1`). `apt-cache policy base-files` on -02 showed a single source: `resolute/main`.
- **Root cause:** The Ubuntu 26.04 Raspberry Pi image's `/etc/apt/sources.list.d/ubuntu.sources` (2415 bytes, stock header, dated to each image build) lists only `resolute` and `resolute-security`. `resolute-updates` is absent on every node. -01/-03 had `.2` only because their newer image was built after it shipped. Nothing on any node could fetch non-security bug fixes.
- **Fix:** `ansible/playbooks/baseline.yml` adds `/etc/apt/sources.list.d/ubuntu-updates.sources` (deb822, `Suites: resolute-updates`, archive keyring) and refreshes the cache via handler. `patch.yml` then converged all three to `base-files 14ubuntu6.2`.
- **Faster detection:** `apt-cache policy base-files` and look for a `-updates` line in the version table; or `grep -h Suites: /etc/apt/sources.list.d/*.sources`. "Nothing to upgrade" is only meaningful if the sources are complete.
- **Automate:** baseline.yml owns the pocket file; a future guard task should assert `-updates` and `-security` both appear in `apt-cache policy` output.
