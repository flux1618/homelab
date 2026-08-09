# Docker Socket Permission Denied After usermod

**Host:** k3s-worker-01 (Raspberry Pi 5, ARM64)
**Date:** 2026-08-09

## Symptom
`docker run hello-world` fails with:
permission denied while trying to connect to the Docker API at unix:///var/run/docker.sock
Works fine when prefixed with `sudo`.

## Root Cause
`sudo usermod -aG docker $USER` updates `/etc/group` but does not refresh the
current shell session's group membership. The active session was still
running with the old group list.

## Fix
newgrp docker
# or fully log out and back in / reboot

## How to Detect Faster
Run `groups` immediately after any `usermod -aG` command — if the new group
isn't listed, the session hasn't picked it up yet, before assuming the fix failed.

## What to Automate
None needed — this is a one-time session quirk. Just remember: re-login is
required after any group membership change, not just a new terminal tab.
