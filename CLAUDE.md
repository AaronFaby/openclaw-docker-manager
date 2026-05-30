# OpenClaw Docker Manager (ocm)

## Project overview

A single-file bash CLI (`ocm`, ~920 lines) for managing multiple OpenClaw Docker containers on a host. No external dependencies beyond Docker.

## Architecture

- `ocm` — the entire tool, a self-contained bash script
- `ocm.conf` — global config (image, default tag, port range start, container prefix). Auto-created on first run.
- `containers/<name>.env` — per-container config. Contains `OCM_PORT`, `OCM_TAG` (internal), and user environment variables (API keys etc). Lines prefixed `OCM_` are stripped before passing to Docker's `--env-file`. After `ocm env` edits the file, surrounding quotes are stripped from values (Docker's `--env-file` passes values verbatim, so quotes would leak into the variable and break API calls).

## Docker resource naming

- Containers: `ocm-<name>`
- Volumes: `ocm-<name>-home`, `ocm-<name>-config`, `ocm-<name>-workspace`, `ocm-<name>-auth`
- Image: `ghcr.io/aaronfaby/openclaw-custom` (configurable in `ocm.conf`)

## Volume mount mapping

| Volume | Container path | Purpose |
|---|---|---|
| `ocm-<name>-home` | `/home/node` | Home dir for supporting utilities' config (everything not covered by a more specific mount) |
| `ocm-<name>-config` | `/home/node/.openclaw` | Config and auth profiles |
| `ocm-<name>-workspace` | `/home/node/.openclaw/workspace` | Session and workspace data |
| `ocm-<name>-auth` | `/home/node/.config/openclaw` | Encryption keys for OAuth tokens |

The mounts are nested: `home` contains `config`/`auth`, and `config` contains `workspace`. Docker handles this correctly only if the mounts are listed parent-first in `docker run` (home → config → workspace → auth) so each more specific mount overlays the one above it. A fresh `home` volume is auto-populated from the image's baked-in `/home/node` contents on first run.

## Key design decisions

- All containers use `--restart unless-stopped` so they survive Docker daemon restarts.
- Port auto-assignment scans existing `.env` files for the highest `OCM_PORT` and increments.
- The script resolves its own location following symlinks (`_ocm_dir`), so it works when symlinked into `$PATH`.
- The "restarting" Docker state must be handled alongside "running" in all lifecycle commands (start, stop, restart, upgrade, rm, status, list, cli, env).
- `_docker_run` is the single source of truth for the `docker run` command, used by start, restart, and upgrade.
- `cmd_setup` runs a one-off interactive container (`docker run -it --rm`) with the same volumes to configure OpenClaw before first start. The gateway crashes without this initial setup.

## Testing

No automated tests. Validate changes manually:

```bash
ocm create test --port 19100
ocm setup test
ocm start test
ocm list
ocm status test
ocm logs test -n 5
ocm stop test
ocm restart test
ocm upgrade test
ocm rm test --force --volumes
```

The container will be in "restarting" state unless `ocm setup <name>` has been run first.

## Gotchas

- Ports 18789 and 18790 may conflict with existing OpenClaw containers on the host. Use `--port` or `ocm env` to assign a free port.
- `set -euo pipefail` is active. Empty arrays need special handling (see `_TMP_FILES` cleanup trap).
- macOS `sed -i` requires `sed -i ''` vs Linux `sed -i`. `cmd_pull`, `cmd_upgrade`, and `_strip_env_quotes` sidestep this entirely by writing `sed` output to a `mktemp` file and `mv`-ing it into place (tracked in `_TMP_FILES` for cleanup).
- `--env-file` passes values verbatim — Docker does *not* interpret quotes. `cmd_env` calls `_strip_env_quotes` after editing to remove a single pair of surrounding matching quotes (`"..."` or `'...'`) from each value, warning when it does. Unbalanced/lone quotes are left intact as likely-intentional content.
