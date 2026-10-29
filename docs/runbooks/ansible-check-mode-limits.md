# Clean `--check --diff` run that would not have worked (what dry runs cannot see)

- **Symptom:** `ansible-playbook --check --diff` reported `failed=0` for a task writing an apt sources file, while the diff appeared to contain a malformed URL. Separately, a `command` task showed `skipping` in check mode and a handler showed `changed` without running.
- **Root cause:** Check mode answers "would this change?", not "is the content valid?". `command`/`shell` are skipped unless `check_mode: false`; handlers report as if they ran; file content is never validated. (The URL turned out to be a display artifact of the copy path, confirmed by `grep -c`, but the dry run could not have told the difference.)
- **Fix:** add `check_mode: false` to read-only validation tasks; verify content with counts (`grep -cF`) rather than by eye when the viewing path can distort text; after applying, test the consumer (`apt-cache policy`, `sshd -T`), not the file.
- **Faster detection:** treat a clean dry run as a forecast. The real check is the post-apply verification command.
- **Automate:** every baseline task that writes config gets a matching read-only verification task that runs in check mode too.
