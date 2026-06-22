# Security finding: claude-jail docker-socket sandbox escape

**Status:** Fix implemented (Option 2), staged for host apply — see `socket-broker/`.
Original assessment below was: Open (assessment only — not exploited).
**Discovered:** 2026-06-11, during a session with the in-repo AI agent (claude-jail sandbox)
**Severity:** Critical — the sandbox does not contain a determined agent; host root is reachable.
**Scope note:** This file documents the host/sandbox configuration, not the Rails app. It
lives in the project root by explicit request; the standard place for project docs is
`.autodocs/`. Remediation must be applied on the **host**, not from inside the jail.

## Summary

The `claude-jail` sandbox confines the agent to this compose project via a filtering
docker shim at `/usr/local/bin/docker`. That shim is a **soft fence, not a security
boundary** — and its single bypass is wide open in the current configuration: the jail
user can talk to the real Docker daemon socket directly, which is equivalent to host root.

The shim's own header says so:

> IMPORTANT: this shim is a soft fence, not a sandbox primitive. Anything that can talk
> to `/var/run/docker.sock` directly (curl, a statically linked docker binary, an MCP
> server, etc.) bypasses it. Treat it as a guard rail for an obedient agent, not a
> security boundary against an adversary.

## Evidence observed

- Jail user: `node` (uid 1000), and **a member of group 999**:
  `uid=1000(node) gid=1000(node) groups=1000(node),999`
- Docker socket is group-writable by that group:
  `srw-rw---- 1 root 999 /var/run/docker.sock`
- The real docker binary is **not** setuid (`-rwxr-xr-x root:root`
  `/usr/local/libexec/docker-real`). The shim reaches the daemon purely through `node`'s
  membership in group 999 — there is no privilege separation between the shim and the
  agent.
- Direct daemon access confirmed, bypassing the shim entirely:
  `curl --unix-socket /var/run/docker.sock http://localhost/version` returned the full
  Docker Engine 29.5.3 version payload. This request never passed through
  `/usr/local/bin/docker`, so none of the shim's allow/deny logic applied.

The assessment stopped at the read-only `version` call. The breakout itself was **not**
performed.

## Why this is host root

A writable docker socket is equivalent to root on the host. The standard escalation is a
single daemon API call: create a container with `HostConfig.Binds: ["/:/host"]` and
`Privileged: true` (or `PidMode: host`), then read/write the host filesystem as root from
inside it — host SSH keys, `/etc/shadow`, cron, and the other compose projects the jail
is meant to fence off. The shim blocks `docker run` at the CLI, but the socket does not
care which client issues the request.

## The core tension

The socket is **simultaneously the escape hatch and the legitimate work channel.** Because
the shim runs `docker-real` as `node`, every intended capability (running specs via
`docker compose exec rails ...`, `parallel:spec`, `docker exec`, `docker logs`,
`bin/psql-local`'s `docker inspect`) flows through the same group-999 → socket path that
enables the escape. You cannot simply revoke socket access without also disabling the
agent's ability to run and test code in the container.

## Remediation options (apply on the host)

### Option 1 — Remove `node` from group 999
Closes the hole completely. **But** it also disables ALL docker functionality from the
jail: the shim's own `docker ps` returns empty → it denies everything, and specs,
`docker exec`, `docker logs`, and `bin/psql-local` all break. Net effect: the agent
becomes read/edit/review-only, with in-container execution gone (lean on CI for tests).
Cheapest and cleanest **if** that capability reduction is acceptable.

### Option 2 — Socket proxy (preserves function, removes host root) — recommended if the agent must run code
Put a docker-socket-proxy (e.g. Tecnativa) between the jail and the real socket; deny
`POST /containers/create`, `/exec` with binds/privileged, etc. Caveat: the proxy filters
at the **endpoint** level, not per-container — allowing `/exec` allows exec into any
container the daemon sees, which loses the shim's project-scoping. To keep both function
and scoping, run the existing shim on the **broker side** of the proxy rather than just
dropping the proxy in. More moving parts to build and maintain; no loss of legitimate
capability.

### Option 3 — userns-remap
Run the jail under a user namespace so root-via-socket maps to an unprivileged host uid;
add seccomp/AppArmor to block raw socket syscalls. Keeps everything working and neuters
the breakout, but expect bind-mount ownership friction (shifted uids on the
bind-mounted source tree) and some images that assume real root misbehaving.

## Recommendation

Decide what the agent is allowed to do first:
- **Run specs/migrations in the container** → Option 2 (proxy + shim on the broker side).
- **Read/edit/review only** → Option 1; accept the loss of in-container execution and use CI.

The shim is fine as a guardrail to keep an *obedient* agent scoped to this project — which
is what it claims to be. It is not containment, and should not be relied on as such if the
threat model includes the model itself misbehaving.

## Implementation (Option 2)

The socket-proxy + broker-side-shim fix is staged in `socket-broker/`:

- `docker-compose.broker.yml` — Tecnativa socket-proxy (sole holder of the real
  socket, on an internal network) + a broker container that runs the scoping
  shim behind an SSH forced command.
- `broker-shim` — the project-scoping shim, now run on the broker; additionally
  blocks container-creating `compose` subcommands (the agent controls
  `docker-compose.yml`).
- `jail-docker` — replaces `/usr/local/bin/docker` in the jail; forwards argv
  over SSH to the broker. The jail no longer mounts the socket or joins the
  docker group.
- `setup.sh` — host-side: generates SSH keys and brings the broker stack up.
- The `claude-jail` launcher is updated to drop the socket mount/`--group-add`
  and instead attach the jail to the broker network and mount the SSH client key.

**Must be applied and verified on the host** (the host owns the real socket and
re-runs the launcher). It deliberately was *not* deployed from inside the jail —
that would require performing the very breakout this documents. Review the
launcher diff before re-running, since it rebuilds the agent's own sandbox. Full
runbook, verification steps, and residual risks: `socket-broker/README.md`.
