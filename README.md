# caged-yolo

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with `--dangerously-skip-permissions` inside a network-sandboxed devcontainer. Full autonomy for Claude, zero risk to your host or the wider internet.

## Why

Claude Code's `--dangerously-skip-permissions` flag lets Claude run shell commands, edit files, and install packages without asking. That's incredibly productive — and terrifying on bare metal. This devcontainer gives you:

- **Network firewall** — iptables + ipset allowlist. Claude can reach GitHub and the Anthropic API, nothing else.
- **Filesystem isolation** — your workspace is bind-mounted, but the rest of your host is untouchable.
- **Shared config** — `~/.claude` is mounted in, so auth, settings, and memory persist across containers.
- **One command** — `caged-yolo` handles container lifecycle, Claude launch, and config.

## Prerequisites

- Docker Desktop (or a Docker-compatible runtime)
- [Dev Container CLI](https://github.com/devcontainers/cli): `npm install -g @devcontainers/cli`
- An Anthropic API key or Claude login configured in `~/.claude`

## Installation

```bash
# Clone this repo inside your workspace root
git clone <this-repo> ~/Workspace/caged-yolo

# Symlink into ~/.local/bin (single-user, no sudo required)
# Make sure ~/.local/bin is on your PATH (add to ~/.zshrc or ~/.bashrc if not):
#   export PATH="$HOME/.local/bin:$PATH"
mkdir -p ~/.local/bin
ln -s "$(pwd)/caged-yolo" ~/.local/bin/caged-yolo
```

The devcontainer config is resolved from the script's real location (symlinks are followed). The script itself can live anywhere — just symlink it onto your PATH.

Each invocation creates a container scoped to the directory you run it from. Only that directory is mounted, keeping projects isolated:

```bash
cd ~/Workspace/project-a
caged-yolo                # mounts project-a at /workspaces/project-a

cd ~/Workspace/project-b
caged-yolo                # separate container, mounts project-b at /workspaces/project-b

# Need both projects in one container?
cd ~/Workspace/project-a
caged-yolo --mount ~/Workspace/project-b   # both available under /workspaces/
```

## Usage

```bash
# Start (or resume) the container and launch Claude in yolo mode
caged-yolo

# Resume the last Claude session
caged-yolo -r

# Mount additional folders
caged-yolo -m ~/other-project -m ~/shared-lib

# List running containers
caged-yolo list

# Manage Claude config
caged-yolo config
caged-yolo config set theme dark

# Stop the container
caged-yolo stop

# Rebuild container (reuses cached image)
caged-yolo rebuild

# Rebuild with latest Claude (busts Docker cache)
caged-yolo rebuild --fresh
```

### Default (no subcommand)

Starts the devcontainer (if not already running), then launches `claude --dangerously-skip-permissions`. The current directory is mounted at `/workspaces/<dir-name>`.

Any extra flags are passed through to Claude: `caged-yolo -r`, `caged-yolo --help`, etc.

### `rebuild`

Destroys and recreates the container. Your `~/.claude` config and shell history are preserved (they live in bind mounts / Docker volumes).

- `caged-yolo rebuild` — fast, reuses the cached Docker image
- `caged-yolo rebuild --fresh` — rebuilds the image with the latest Claude Code (takes ~1 min)

## Network Security Model

The container starts with `NET_ADMIN` and `NET_RAW` capabilities so it can configure its own firewall. On every start, `init-firewall.sh` runs and:

1. Flushes all iptables rules and ipsets
2. Restores Docker's internal DNS resolution (127.0.0.11)
3. Allows DNS (UDP 53), SSH (TCP 22), and localhost
4. Creates an `allowed-domains` ipset (hash:net)
5. Populates it with resolved IPs for the allowlist
6. Allows traffic to the host Docker network
7. Sets default policy to **DROP** for INPUT, OUTPUT, and FORWARD
8. Adds a final **REJECT** rule for clear error messages
9. Verifies the firewall works (blocks `example.com`, allows `api.github.com`)

### Allowed Domains

| Domain | Purpose |
|--------|---------|
| GitHub IPs (via `/meta` API) | Git push/pull/clone, GitHub API, Actions |
| `raw.githubusercontent.com` | Raw file access from GitHub |
| `objects.githubusercontent.com` | GitHub release assets |
| `claude.com` | Claude web |
| `registry.npmjs.org` | npm package installs |
| `api.anthropic.com` | Claude API |
| `sentry.io` | Error reporting |
| `statsig.anthropic.com` | Anthropic feature flags |
| `statsig.com` | Feature flag service |
| `marketplace.visualstudio.com` | VS Code extensions |
| `vscode.blob.core.windows.net` | VS Code extension downloads |
| `update.code.visualstudio.com` | VS Code updates |

### Notably absent

| Domain | Why blocked |
|--------|-------------|
| `storage.googleapis.com` | Serves **all** GCS buckets — too broad. |
| Everything else | Default deny. Claude cannot exfiltrate data, download arbitrary packages, or reach unexpected services. |

## Architecture

```
Host                              Container (node:20)
─────────────────────────────     ──────────────────────────────────
caged-yolo                   ──▶  claude --dangerously-skip-permissions
  ├── devcontainer up              ├── zsh + powerlevel10k
  ├── devcontainer exec            ├── git, gh, fzf, delta, jq
  └── docker stop                  ├── iptables firewall (init-firewall.sh)
                                   └── Claude Code (native install)

Mounts:
  ~/Workspace        ──▶  /workspaces        (all projects)
  ~/.claude          ──▶  /home/node/.claude  (auth, config, memory)
  docker volume      ──▶  /commandhistory     (shell history)
```

## Container Environment

| Variable | Value | Purpose |
|----------|-------|---------|
| `DEVCONTAINER` | `true` | Signals we're in a devcontainer |
| `NODE_OPTIONS` | `--max-old-space-size=4096` | Prevents Node OOM for large projects |
| `CLAUDE_CONFIG_DIR` | `/home/node/.claude` | Points Claude to mounted config |
| `SHELL` | `/bin/zsh` | Default shell |
| `EDITOR` / `VISUAL` | `nano` | Default editor |

## Customization

### Adding allowed domains

Edit the domain list in `.devcontainer/init-firewall.sh` and run `caged-yolo rebuild`.

### Changing the base image

Edit the `FROM` line in `.devcontainer/Dockerfile`. The firewall and Claude install work on any Debian-based image.

## Roadmap: Multi-technology support

Currently the Dockerfile uses `node:24` as the base image. This works for Node projects but doesn't scale to other stacks. The plan is to migrate to a generic base image with devcontainer features for language runtimes.

### Phase 1: Generic base image

Switch from `node:24` to `mcr.microsoft.com/devcontainers/base:debian`. Add Node as a devcontainer feature (Claude Code needs it regardless of project language). This decouples "Claude's runtime" from "the project's runtime".

```jsonc
// devcontainer.json
"features": {
  "ghcr.io/devcontainers/features/node:1": {},  // always — Claude needs Node
  "ghcr.io/devcontainers/features/python:1": {} // per-project
}
```

### Phase 2: Per-project feature selection

Support a `.caged-yolo.json` config file in each project directory that declares which features it needs:

```jsonc
// ~/Workspace/my-python-api/.caged-yolo.json
{
  "features": {
    "ghcr.io/devcontainers/features/python:1": { "version": "3.12" },
    "ghcr.io/devcontainers/features/node:1": { "installPnpm": true }
  }
}
```

The CLI would merge these into the base devcontainer.json at container creation time.

### Phase 3: Firewall profiles per technology

Different stacks need different domain allowlists (e.g. `pypi.org` for Python, `maven.org` for Java). Firewall rules could be bundled with feature selection so adding a language automatically allows its package registry.

### VS Code extensions

Add extension IDs to the `extensions` array in `.devcontainer/devcontainer.json`.

### Timezone

Set the `TZ` environment variable on your host, or change the default in `.devcontainer/devcontainer.json`:

```json
"TZ": "${localEnv:TZ:Europe/Madrid}"
```

## Troubleshooting

### `ECONNREFUSED` when running commands

The firewall is blocking the connection. Check if the domain needs to be added to the allowlist in `.devcontainer/init-firewall.sh`.

### Firewall fails to initialize

Check that the container has `NET_ADMIN` and `NET_RAW` capabilities (`runArgs` in `.devcontainer/devcontainer.json`). Also verify DNS is working inside the container: `dig api.github.com`.

### Container won't start

```bash
caged-yolo rebuild   # clean slate
```

### Claude auth issues

Your `~/.claude` directory is bind-mounted. Run `claude login` on your host or inside the container to re-authenticate.
