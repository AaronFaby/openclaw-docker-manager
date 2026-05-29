# ocm - OpenClaw Docker Manager

A lightweight CLI for managing multiple [OpenClaw](https://github.com/AaronFaby/openclaw-docker-image) Docker containers on a single host.

## Features

- Rapid container deployment with a single command
- Persistent data across upgrades via Docker volumes
- Per-container environment variable management (API keys, config)
- Seamless image upgrades without data loss
- Multi-container support with automatic port assignment
- Helper commands for logs, shell access, status monitoring

## Quick start

```bash
# Clone and make executable (or symlink ocm into your PATH)
git clone <repo-url> && cd openclaw-docker-manager
chmod +x ocm

# Create and configure a container
./ocm create main
./ocm env main          # Set your API keys (ANTHROPIC_API_KEY, etc.)
./ocm setup main        # Interactive OpenClaw setup (first time only)
./ocm start main

# Check status
./ocm list
./ocm status main
./ocm logs main -f
```

## Commands

### Container lifecycle

```bash
ocm create <name> [--tag TAG] [--port PORT]   # Create a container config
ocm setup <name>                               # Interactive OpenClaw setup
ocm start <name>                               # Start a container
ocm stop <name>                                # Stop a container
ocm restart <name>                             # Restart a container
ocm rm <name> [--volumes] [--force]            # Remove a container
```

### Management

```bash
ocm list                                       # List all containers
ocm status [name]                              # Show detailed container status
ocm logs <name> [-f] [-n LINES]                # View container logs
ocm cli <name> [cmd...]                        # Shell or run openclaw commands
ocm env <name>                                 # Edit environment variables
```

### Image management

```bash
ocm pull [tag]                                 # Pull latest image
ocm upgrade <name> [--tag TAG]                 # Upgrade a single container
ocm upgrade --all [--tag TAG]                  # Upgrade all containers
```

## Configuration

### Global config (`ocm.conf`)

Auto-created on first run. Configures the image source and defaults:

```bash
OCM_IMAGE=ghcr.io/aaronfaby/openclaw-custom
OCM_DEFAULT_TAG=latest
OCM_PORT_START=18789
OCM_CONTAINER_PREFIX=ocm
```

### Per-container config (`containers/<name>.env`)

Each container has an env file editable via `ocm env <name>`:

```bash
# --- OCM internal (do not remove) ---
OCM_PORT=18789
OCM_TAG=latest

# --- User environment variables ---
ANTHROPIC_API_KEY=sk-ant-...
XAI_API_KEY=xai-...
OPENAI_API_KEY=sk-...
```

`OCM_PORT` and `OCM_TAG` are used by `ocm` internally. All other variables are passed to the container.

## Data persistence

Each container gets four Docker volumes that persist across restarts and upgrades:

| Volume | Container path | Contents |
|---|---|---|
| `ocm-<name>-home` | `/home/node` | Home directory — config for supporting utilities (everything not covered by a more specific volume below) |
| `ocm-<name>-config` | `/home/node/.openclaw` | Configuration and auth profiles |
| `ocm-<name>-workspace` | `/home/node/.openclaw/workspace` | Session and workspace data |
| `ocm-<name>-auth` | `/home/node/.config/openclaw` | OAuth token encryption keys |

The mounts are nested (`home` ⊃ `config` ⊃ `workspace`, and `home` ⊃ `auth`); Docker overlays the more specific volume on top of the more general one. A fresh `home` volume is auto-populated from the image's baked-in `/home/node` contents on first start.

Volumes are **not** removed when you `ocm rm` a container unless you pass `--volumes`.

## Upgrading containers

Upgrades pull the latest image, stop the container, remove it, and recreate it with the same configuration and volumes — your data is preserved:

```bash
ocm upgrade main           # Upgrade a single container
ocm upgrade --all          # Upgrade all containers
ocm upgrade main --tag oc-2026.5.22   # Pin to a specific version
```

## Accessing the container

```bash
ocm cli main                  # Interactive bash shell
ocm cli main configure        # Run 'openclaw configure'
ocm cli main doctor --fix     # Run 'openclaw doctor --fix'
ocm cli main --raw env        # Run an arbitrary command
```

## Multiple containers

Containers are automatically assigned incrementing ports starting from `OCM_PORT_START` (default 18789):

```bash
ocm create production              # Port 18789
ocm create staging                 # Port 18790
ocm create dev --port 19000        # Explicit port
ocm create pinned --tag oc-2026.5.22  # Pinned image version
```

## Requirements

- Docker
- Bash 4+
