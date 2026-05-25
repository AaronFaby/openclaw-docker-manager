# OpenClaw Docker Manager (ocm)

## Project overview

A single-file bash CLI (`ocm`, ~770 lines) for managing multiple OpenClaw Docker containers on a host. No external dependencies beyond Docker.

## Architecture

- `ocm` — the entire tool, a self-contained bash script
- `ocm.conf` — global config (image, default tag, port range start, container prefix). Auto-created on first run.
- `containers/<name>.env` — per-container config. Contains `OCM_PORT`, `OCM_TAG` (internal), and user environment variables (API keys etc). Lines prefixed `OCM_` are stripped before passing to Docker's `--env-file`.

## Docker resource naming

- Containers: `ocm-<name>`
- Volumes: `ocm-<name>-config`, `ocm-<name>-workspace`, `ocm-<name>-auth`
- Image: `ghcr.io/aaronfaby/openclaw-custom` (configurable in `ocm.conf`)

## Volume mount mapping

| Volume | Container path | Purpose |
|---|---|---|
| `ocm-<name>-config` | `/home/node/.openclaw` | Config and auth profiles |
| `ocm-<name>-workspace` | `/home/node/.openclaw/workspace` | Session and workspace data |
| `ocm-<name>-auth` | `/home/node/.config/openclaw` | Encryption keys for OAuth tokens |

The workspace mount is nested inside the config mount. Docker handles this correctly — list workspace after config in `docker run` so the more specific mount overlays.

## Key design decisions

- All containers use `--restart unless-stopped` so they survive Docker daemon restarts.
- Port auto-assignment scans existing `.env` files for the highest `OCM_PORT` and increments.
- The script resolves its own location following symlinks (`_ocm_dir`), so it works when symlinked into `$PATH`.
- The "restarting" Docker state must be handled alongside "running" in all lifecycle commands (start, stop, restart, upgrade, rm, status, list, cli).
- `_docker_run` is the single source of truth for the `docker run` command, used by start, restart, and upgrade.

## Testing

No automated tests. Validate changes manually:

```bash
ocm create test --port 19100
ocm start test
ocm list
ocm status test
ocm logs test -n 5
ocm stop test
ocm restart test
ocm upgrade test
ocm rm test --force --volumes
```

The container will be in "restarting" state unless configured with valid API keys via `ocm env <name>`.

## Gotchas

- Ports 18789 and 18790 may conflict with existing OpenClaw containers on the host. Use `--port` or `ocm env` to assign a free port.
- `set -euo pipefail` is active. Empty arrays need special handling (see `_TMP_FILES` cleanup trap).
- macOS `sed -i` requires `sed -i ''` vs Linux `sed -i`. The `cmd_pull` function handles this with an `$OSTYPE` check; `cmd_upgrade` uses `sed -i.bak` + `rm .bak` for portability.
