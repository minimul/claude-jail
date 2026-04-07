# claude-jail

A single-file Bash launcher that runs the [`claude`](https://github.com/anthropics/anthropic-sdk-python) CLI inside a Docker sandbox. Node.js and npm dependencies stay off your host machine, and `claude` only has access to the directory from which `claude-jail` is launched, plus any Docker containers defined within that same directory.

## Security model

The claude-jail container is **fully isolated from the host Docker daemon**. Instead of mounting the Docker socket (which grants full host-level access), claude-jail uses **Docker-in-Docker (DinD)**, a completely separate Docker daemon running inside its own container.

### Why DinD instead of docker-socket-proxy?

A Docker socket proxy (e.g. Tecnativa/docker-socket-proxy) filters by API *endpoint category*, but it cannot restrict what happens *within* allowed endpoints. Since `docker compose` requires both the Containers and POST endpoints, a proxy still allows:

- Creating privileged containers that mount the host root filesystem
- Inspecting environment variables (secrets, API keys) of all host containers
- Exec-ing into any container on the host

DinD eliminates all of these vectors by giving the claude-jail container its own isolated daemon with its own container/image/network/volume namespace.

### What DinD prevents

| Attack vector | Protected? | How |
|---|---|---|
| Mount host filesystem via `docker run -v /:/host` | ✅ | DinD's "host" is its own isolated filesystem |
| Read secrets from other containers | ✅ | Other containers don't exist in the DinD daemon |
| Exec into other containers | ✅ | Other containers don't exist in the DinD daemon |
| Create privileged containers | ✅ | Privileged inside DinD is scoped to DinD, not the real host |
| Access host Docker socket | ✅ | Socket is never mounted into claude-jail |
| Tamper with claude credentials | ⚠️ | `~/.claude` is read-write (claude needs to write sessions & locks); protect `auth.json` with `chmod 600` on the host |

### Trade-offs

- The DinD sidecar requires `--privileged` for its own container (it runs a Docker daemon). This is standard for DinD and does not grant privileges to the claude-jail container.
- Images pulled inside DinD are stored in the DinD container's filesystem, not the host's image cache. They persist as long as the `claude-jail-dind` container exists.

## How it works

On first run, `claude-jail` builds a Docker image from an embedded Dockerfile (Node.js LTS on Debian Bookworm with `claude` and common CLI tools pre-installed). It also starts a `claude-jail-dind-<dirhash>` sidecar container running an isolated Docker daemon, on a dedicated `claude-jail-net-<dirhash>` bridge network. The directory hash scopes DinD resources per launch directory so concurrent sessions in different directories are fully isolated. The sidecar and network are torn down automatically when the claude container exits. If you edit the script — for example to add packages — the image is automatically rebuilt via `md5sum` change detection.

## Prerequisites

- Docker installed and running

## Installation

Copy the script to somewhere on your `$PATH` and make it executable:

```bash
cp claude-jail ~/.local/bin/claude-jail
chmod +x ~/.local/bin/claude-jail
```

## Usage

```
claude-jail [OPTIONS] [-- ARGS...]

Options:
  -e, --env KEY=VAL   Pass an environment variable to the container (repeatable)
  -r, --rebuild       Force rebuild of the Docker image
      --no-cache      Rebuild without using Docker cache
  -s, --shell         Start a bash shell instead of claude
  -R, --resume UID    Resume a previous session by its ID
  -c, --continue      Continue the most recent session
      --safe          Do NOT pass --dangerously-skip-permissions to claude
  -h, --help          Show this help message

Arguments after -- are passed through to claude.
```

By default, `claude-jail` launches `claude` with `--dangerously-skip-permissions`. Since `claude` is running inside an isolated container with no access to the host Docker socket or filesystem outside the launch directory, the per-tool permission prompts add friction without meaningful protection. Pass `--safe` to restore the normal permission prompts.

### Examples

```bash
# Run claude interactively (builds the image on first run)
claude-jail

# Pass arguments directly to claude
claude-jail -- -p "Summarize this codebase"

# Pass environment variables into the container
claude-jail -e ANTHROPIC_API_KEY=sk-ant-... -e MY_VAR=hello

# Combine env vars with claude arguments
claude-jail -e ANTHROPIC_API_KEY=sk-ant-... -- -p "Summarize this codebase"

# Open a shell inside the container
claude-jail --shell

# Force a full image rebuild
claude-jail --rebuild
```

If `--shell` is used while a container from the same working directory is already running, the script `docker exec`s into it rather than starting a new one.

## Operational safety

The jail protects the host, not the launch directory. Keep these rules in mind:

1. **Never launch `claude-jail` from `$HOME`, `/`, or any directory containing unrelated projects.** The current working directory is mounted read-write and `claude` runs with `--dangerously-skip-permissions` by default, so anything under `$(pwd)` can be modified or deleted without confirmation. Launch from a specific project directory you are willing to let `claude` rewrite.
2. **Protect your API key.** `~/.claude-home/.claude/auth.json` is mounted read-write into the container; a compromised session can read it. At minimum:
   ```bash
   chmod 600 ~/.claude-home/.claude/auth.json
   ```
   Prefer short-lived OAuth credentials over a long-lived `ANTHROPIC_API_KEY` when possible.
3. **Egress is not restricted.** The claude container can reach the public internet. If you need tighter control, attach it to a network with an explicit allowlist.

The claude container itself runs with `--cap-drop=ALL` and `--security-opt=no-new-privileges` to limit in-container privilege escalation.

## Configuration

### API keys

Claude state is persisted on the host under `~/.claude-home` and mounted into the container as `/home/node/.claude` (plus `~/.claude-home/.claude.json` → `/home/node/.claude.json`). `HOME` inside the container is `/home/node`, so credentials written there are available on every run without setting environment variables.

For example, to configure your Anthropic API key:

```bash
mkdir -p ~/.claude-home/.claude
echo '{"anthropic":{"type":"api_key","key":"sk-ant-..."}}' > ~/.claude-home/.claude/auth.json
chmod 600 ~/.claude-home/.claude/auth.json
```

### Adding packages

To install extra packages into the image, edit the `CUSTOM_APT_PACKAGES` variable near the top of the script:

```bash
CUSTOM_APT_PACKAGES="jq git tmux sqlite3"
```

Save the file and the image will rebuild automatically on the next run.

## Volume mounts

| Host | Container | Mode | Purpose |
|---|---|---|---|
| `$(pwd)` | same path | read-write | Current working directory (path-mirrored for Docker compose compatibility) |
| `~/.claude-home/.claude` | `/home/node/.claude` | read-write | claude configuration, sessions, auth |
| `~/.claude-home/.claude.json` | `/home/node/.claude.json` | read-write | claude user settings file |

The Docker socket (`/var/run/docker.sock`) is **not** mounted into the claude-jail container. Docker access is provided via `DOCKER_HOST=tcp://<dind-ip>:2375` pointing to the isolated `claude-jail-dind` sidecar daemon.

The launch directory is mounted at its exact host path so that paths used by `docker-compose` volume mounts resolve correctly inside the DinD daemon. `HOME` is passed in explicitly as `/home/node` so `claude` can locate its config directory.

## DinD sidecar management

The DinD daemon runs as a per-directory container named `claude-jail-dind-<dirhash>` on a dedicated `claude-jail-net-<dirhash>` bridge network. Both are created on `claude-jail` startup and removed automatically when the claude container exits (via an `EXIT` trap). If `--shell` attaches to an already-running claude container, the sidecar is left alone.

```bash
# List any active claude-jail DinD sidecars
docker ps --filter name=claude-jail-dind-

# Force-remove a leftover sidecar (e.g. after an unclean exit)
docker rm -f claude-jail-dind-<dirhash>
docker network rm claude-jail-net-<dirhash>
```

## Included tools

The Docker image ships with these CLI tools alongside `claude`:

- `git` — version control
- `jq` — JSON processor
- `gh` — GitHub CLI
- `docker` CLI and `docker compose` plugin
- `curl`, `gnupg`, `build-essential`

## Testing Docker access

A `docker-compose.yml` is included to verify that `claude-jail` can reach Docker containers defined in the same directory. It runs two automated tests:

1. **Volume mount** — confirms that `docker-compose.yml` is visible inside the container via the workspace volume mount.
2. **Service networking** — confirms that the `docker-access-test` container can reach the `test-web` service over the compose network.

Open a shell inside the `claude-jail` container from the `claude-jail` directory:

```bash
claude-jail --shell
```

From inside the container, run the tests:

```bash
docker compose up --abort-on-container-exit --exit-code-from docker-access-test
```

Expected output:

```
=== DinD Docker Compose Test ===

--- Test 1: Volume mount (project files visible) ---
[PASS] docker-compose.yml found via volume mount

--- Test 2: Compose service networking ---
[PASS] Got response from test-web: OK
```

Both services exit cleanly when all tests pass. Tear down afterward:

```bash
docker compose down
```

### Verifying isolation

From inside the claude-jail container, confirm that the host is not accessible:

```bash
# No Docker socket — should fail
ls -la /var/run/docker.sock
# => ls: cannot access '/var/run/docker.sock': No such file or directory

# Only compose containers visible — no host containers
docker ps -a
# => (only your compose services)

# Cannot see host filesystem even through Docker
docker run --rm -v /:/host alpine cat /host/etc/hostname
# => (shows DinD hostname, not host machine hostname)
```

## License

MIT
