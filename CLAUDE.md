# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A standalone Bash installer, run as root on a Proxmox VE node, that creates an unprivileged Debian LXC loaded with the Claude Code CLI and a dev toolchain. It follows community-scripts.org conventions but is deliberately self-contained: it runs its own `pct create` and does **not** source community-scripts' `build.func`. There is no build system. The repo holds three shell files.

## Validation

You can't run it end to end off a Proxmox host. To check locally:

```bash
bash -n ct/claude-code.sh install/claude-code-install.sh misc/core.func      # parse check
shellcheck ct/claude-code.sh install/claude-code-install.sh misc/core.func   # lint
DRYRUN=1 ./ct/claude-code.sh   # on a PVE node: prints mutating pct/pveam commands instead of running them
```

`DRYRUN` covers only commands wrapped in `run()`. Read-only queries (`pvesm status`, `pveam available`, `pct status`) still run, so a dry run still needs a real PVE node.

## Architecture: two execution contexts

1. **`ct/claude-code.sh`** runs on the host as the orchestrator. It does preflight checks, asks for CTID/resources/storage/bridge, collects bootstrap access (an SSH key or password, at least one required), fetches the template, runs `pct create`, sets `cmode shell`, starts the container, and waits for an IP. It then uses `pct push` to send a payload to `/opt/claude-setup/` inside the container and runs the installer there with `pct exec`.
2. **`install/claude-code-install.sh`** runs inside the container. It installs apt packages, Node LTS and the OpenAI Codex CLI via npm (both optional, non-fatal on failure), `gh`, the `dev` user with passwordless sudo, sshd (key-only unless a password was given), the Claude Code native installer (run as `dev`), the `claude-start`/`claude-remote` tmux helpers, the MOTD, and `/usr/bin/update`. Finally it deletes `/opt/claude-setup`, which holds the password file.
3. **`misc/core.func`** holds the shared `msg_info/ok/warn/error`, spinner, `header_info` and `catch_errors` helpers. The host script sources it, and a pushed copy is sourced inside the container.

Host→container contract: the host writes `bootstrap.env` with `DEV_USER`, `HAS_PUBKEY` and `HAS_PW`, plus optional `authorized_keys` and `bootstrap_pw` files. The installer sources these and has defaults for every value. If you add a new setting, update both sides.

`fetch_file` prefers a local checkout (`SELF_DIR`) and falls back to `curl $REPO_RAW/<path>`. When the script runs via the curl one-liner, sibling files come from GitHub `main`, not from your working tree.

Prompts are skipped when stdin isn't a TTY or `NONINTERACTIVE=1` is set. In that case answers come from env vars (`CTID`, `HOSTNAME`, `CPU`, `RAM`, `DISK`, `STORAGE`, `BRIDGE`, `SSH_PUBKEY`, `ROOT_PASSWORD`).

## Hard-won constraints (see git history)

- **Spinners vs. streamed output:** don't put a host-side `msg_info` spinner around `pveam download` or `pct exec ... install.sh`. Both stream their own progress, and the spinner's `\r` redraws clobber it (that caused the flicker bug). The in-container installer sets `NO_SPINNER=1` for the same reason; `pct exec` passes a TTY, so `-t 2` alone isn't enough.
- **Locale:** the installer forces `LANG=LC_ALL=C.UTF-8` during install, because the host locale leaks in through `pct exec` before `en_US.UTF-8` exists.
- **Proxmox Console access** comes from `pct set <ctid> -cmode shell` on the host. Don't add getty/autologin config inside the container: in unprivileged LXCs, `console-getty` fails with `243/CREDENTIALS`.
- Every script runs under `set -Eeuo pipefail` (the installer adds an ERR trap via `catch_errors`). Optional steps must be wrapped in `if ...; then` or `|| true` so they don't abort provisioning.
- Don't introduce `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` anywhere: they override the subscription login and disable Remote Control. Auth is meant to be an interactive first-run `claude` login as `dev`.
- `.gitattributes` forces LF endings on `*.sh`, `*.func` and `*.md`.
