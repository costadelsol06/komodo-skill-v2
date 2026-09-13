---
name: komodo
description: Use when deploying, configuring, or troubleshooting Komodo (komo.do) — a Docker/Compose orchestration platform with Core/Periphery architecture. Covers Stack and Build resources, auto-update, custom image rebuilds via Procedures, the Komodo API/CLI, and specific failure modes like "pull access denied", "Resource is busy", stale cache after Delete, Periphery network/build or "network is unreachable" IPv6 issues, and a scheduled RunBuild/Procedure that intermittently fails with a DNS or registry-auth-looking error (e.g. "server misbehaving", "failed to fetch anonymous token") at the same time every day.
---

# Komodo

## Overview

Komodo (komo.do) is a self-hosted control plane for Docker: a web dashboard plus
**Core** (API/UI server) + **Periphery** (lightweight agent per managed host, talks to
the Docker socket). It deploys `docker compose` projects as **Stacks**, builds images
from Dockerfiles as **Builds**, and orchestrates both via **Procedures** on a
**Schedule**. It is explicitly *not* a container runtime — it drives the same
`docker`/`docker compose` CLI you'd run by hand, over SSH-like agent calls.

This skill documents behavior that is easy to get wrong even after reading the official
docs shallowly — all points below were confirmed either by reading Komodo's Rust source
directly or by reproducing the failure on a live instance.

## When to Use

- Standing up a new Komodo instance (Core + Periphery + Mongo/FerretDB) via Docker Compose
- Adopting an **already-running** `docker compose` project into a Komodo Stack
- A Stack shows a stale/wrong state (phantom services, "unhealthy" that doesn't match
  reality) after editing the underlying compose file
- A custom-built image (no registry, `build:` + local `image:` tag) won't deploy via
  Komodo — errors like `pull access denied`, `repository does not exist`
- A Procedure with parallel executions fails with `"Resource is busy"`
- Automating "rebuild a locally-built image, then redeploy just that service"
- Calling the Komodo HTTP API or `km` CLI programmatically
- Periphery's Docker builds fail with DNS/network errors despite the host having internet
- The "Global Auto Update" Procedure fails nightly with `network is unreachable` on an
  IPv6 address, even though the host itself has working IPv6
- A scheduled `RunBuild`/Procedure fails intermittently or every day at the same clock
  time with a DNS/registry error that looks like a Docker Hub auth problem, but network
  connectivity tests fine outside that window

## Quick Reference

| Symptom / Goal | Cause | Fix |
|---|---|---|
| `pull access denied`, `repository does not exist` on deploy | Stack `auto_pull` (default true) forces `docker compose pull <service>` before every deploy; fails for locally-built-only images | Add `pull_policy: build` to that **service** in the compose file (not the Stack setting) |
| `"Resource is busy"` executing two deploys in a Procedure stage | Two `DeployStack` calls targeting the **same Stack** ran in parallel — Komodo locks per-Stack | Never parallelize two deploys on the same Stack in one Procedure stage; different Stacks in parallel is fine |
| Stack shows wrong/stale service list ("unhealthy", phantom service) after editing compose | `info.latest_services` is a cache, only recomputed on Stack create/save via API/UI — not on restart, not on poll | Use `RefreshStackCache` (or delete+recreate as last resort — see Danger below) |
| Need to build a Docker image Komodo doesn't need to push anywhere | Registry is optional | Leave `image_registry` empty on the Build — image is built and tagged locally only |
| Need a periodic "rebuild custom image + redeploy" | `RunBuild` never triggers a Stack redeploy by itself | Procedure: Stage 1 `RunBuild` (parallel OK), Stage 2+ `DeployStack{services:[...]}` (one Stack per parallel group) |
| Periphery `docker build` fails resolving DNS for the base image | Periphery's own container network has no internet egress | Give Periphery's container a network with real internet egress — **not** the same bridge as public-facing containers (see Networking) |
| "Global Auto Update" fails nightly, `dial tcp [<ipv6>]:443: connect: network is unreachable` | Periphery's egress network has `EnableIPv6=false`; Docker's embedded DNS still returns AAAA records for the registry host, and there's no IPv6 route to try them on | Enable IPv6 (`enable_ipv6: true` + a ULA subnet) on that network — isolation is unaffected, see IPv6 Egress section |
| Need to script Komodo config instead of clicking through the UI | — | Generate an API key via `km create api-key`, call the HTTP API directly (see API section) |
| Scheduled `RunBuild` fails daily at the same instant with a DNS/token error (`server misbehaving`, `failed to fetch anonymous token`) that reads like Docker Hub auth, but works fine when run manually | A Procedure's `"Every day at HH:MM"` schedule runs in **Core container's own `TZ`** (e.g. `Europe/Paris`), not UTC — it collides on the real clock with another daily job (backup, cron) that saturates CPU/I-O at that instant | Check Core's `TZ` (`docker exec <core> date`), convert the schedule to the same clock as the colliding job, and stagger one of them — see Schedule Collisions section |

## Architecture

- **Core**: web UI + REST/WS API. Stores all config in MongoDB (or FerretDB).
- **Periphery**: stateless agent, one per managed host, mounts the Docker socket, runs
  the actual `docker`/`docker compose` commands. Authenticates to Core via an Ed25519
  keypair exchanged automatically on first boot if both share a Docker volume — no
  manual key copying needed for a single-host bundled deployment.
- Resources: `Server`, `Stack` (compose), `Deployment` (single `docker run` container),
  `Build` (image from Dockerfile), `Builder` (where a Build runs), `Procedure`
  (multi-stage automation), `ResourceSync` (declarative TOML-in-git for everything).

## Deploying an Existing Compose Project (adopt, don't migrate)

To attach Komodo to a compose project that's **already running**, without touching it:

1. Use Stack mode **"Files on Server"**, pointing `run_directory` + `file_paths` at the
   real, already-running directory on the host — not a fresh git clone. A git clone puts
   the files at a different path, breaking any relative bind mounts in the compose file.
2. Periphery only sees paths under its `root_directory` (default `/etc/komodo`) by
   default. For "Files on Server" pointing elsewhere, bind-mount that host directory
   into the Periphery container at the **same path** (Komodo requires host path ==
   container path for this).
3. **Set `project_name` explicitly**, matching the real Docker Compose project name
   exactly (case-sensitive). Verify with `docker compose ls` on the host first. A
   mismatch doesn't error — Komodo just can't find the "expected" containers and shows
   the Stack as down/unhealthy, since Docker Compose project identity is a label match,
   not a name-guessing heuristic.
4. Leave the Stack's `environment` field empty — Compose auto-loads the existing
   `.env` from `run_directory`. Don't duplicate secrets into Komodo's UI/DB.

## The `auto_pull` Trap (locally-built images)

Komodo's Stack config has `auto_pull` (default `true`). Every `DeployStack` execution
runs, in order:

```
docker compose pull <services>   # only if auto_pull=true; abort on failure
docker compose up -d <services>
```

For a service that's `build:`-only with **no real registry image** (e.g.
`image: my-custom-app:latest` where `my-custom-app` was never pushed anywhere), the
`pull` step always fails: `pull access denied for my-custom-app, repository does not
exist`. Komodo aborts the whole deploy at that point — **the running container is never
touched**, but the fresh image you just built is also never applied.

**Do not disable `auto_pull` at the Stack level** to fix this — it also gates the pull
step for every *other* service in the Stack, including registry-backed ones your
auto-update relies on.

**Correct fix**: set Docker Compose's own per-service `pull_policy: build` on just the
locally-built service(s):

```yaml
services:
  my-custom-app:
    build: ./my-custom-app
    image: my-custom-app:latest
    pull_policy: build   # tells Compose to never attempt a registry pull for this service
```

Verify directly before trusting it: `docker compose pull my-custom-app` should print
`Image my-custom-app:latest Skipped` instead of erroring.

A service with **only** `build:` and no `image:` field at all is naturally immune (there's
nothing for Compose to try pulling), but any service that also sets an explicit `image:`
tag needs `pull_policy: build` explicitly.

## Same-Stack Deploy Lock

Komodo serializes deploy-type actions **per Stack**. Two `DeployStack` executions
targeting the same Stack, dispatched in parallel (e.g. two executions in one Procedure
stage), race for a lock — the loser fails with `"Resource is busy"`. The one that lost
the race does not touch its target container; nothing is corrupted, but nothing is
applied either.

Executions against **different** Stacks run fine in parallel. Design Procedures
accordingly:

```toml
[[procedure.config.stage]]
name = "Deploy (different stacks, safe in parallel)"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["svc1"] } },
  { execution.type = "DeployStack", execution.params = { stack = "stack-b", services = ["svc2"] } },
]

[[procedure.config.stage]]
name = "Deploy (same stack as above — must be sequential)"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["svc3"] } },
]
```

**Two or more locally-built services in the same Stack** (both the `auto_pull` trap and
this lock apply together): give every one of them `pull_policy: build` (see previous
section), then prefer **one `DeployStack` call listing all of them** in `services` over
several separate calls — `services` accepts a list, and one call against a Stack never
races itself:

```toml
[[procedure.config.stage]]
name = "Deploy both locally-built services in one call — no lock risk"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["svc1", "svc2"] } },
]
```

Only fall back to sequential stages (one execution per stage) if you specifically need
them deployed one at a time rather than together.

## Stale Cache vs. Destructive Delete — read this before deleting anything

Komodo caches a Stack's expected container list in `info.latest_services`. This cache is
only recomputed when the Stack is **created or its config saved** through the normal
API/UI path — **not** by a Core restart, not by the periodic monitor, not by editing the
compose file on disk. If you remove a service from the compose file, the Stack keeps
believing it should exist, and shows the Stack as unhealthy/wrong indefinitely.

There is a dedicated, safe fix: the **`RefreshStackCache`** action (internally used by
Komodo's own update-check flow, `check_stack_for_update_inner`) recomputes this cache
**without deploying or touching any container**. Prefer this over anything destructive.
Same param shape as every other Stack action — one field, the Stack id or name:

```bash
# Reads credentials from env vars — never hardcode or print a key/secret (see API section)
curl -X POST https://<your-komodo-host>/write \
  -H "X-Api-Key: $KOMODO_API_KEY" -H "X-Api-Secret: $KOMODO_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"type":"RefreshStackCache","params":{"stack":"<stack-id-or-name>"}}'
```

**⚠️ `Delete` on a Stack is NOT purely a metadata operation.** Deleting a Stack (even in
"Files on Server" mode, where Komodo never wrote the compose file itself) can trigger a
real `docker compose down` on the underlying project — every container in that Stack
stops and is removed. Named volumes are preserved (`down` without `-v`), so data
survives, but **all containers in the Stack go offline simultaneously** until you run
`docker compose up -d` again from the untouched compose file. This is confirmed
behavior, not a bug — Komodo's own container-lifecycle docs describe "Remove" as
destroying the container, and `Delete` on a Stack follows the same philosophy. It is
**not reliably safe** to assume otherwise just because it worked without incident once.

**Rule of thumb:**
- Stale display / wrong cached service list → `RefreshStackCache`.
- Genuinely need to reconstruct a Stack resource → `Create` a fresh one (adopting
  already-running containers via "Files on Server" is safe and does not touch them).
- Avoid `Delete` on a Stack entirely unless you are prepared for every container in it
  to go down, and have a plan (and the compose file) to bring them back up immediately.

## Building Custom Images Locally (no registry needed)

A Build resource doesn't require a registry. Leave `image_registry` empty (`[]`) and
Komodo runs `docker build` on the configured Builder, tagging the result **locally only**
— nothing is pushed anywhere. This is the right setup when the Builder is the same host
that will also run the resulting container (no need to round-trip through a registry).

Three source modes exist: write the Dockerfile in the UI, clone a git repo, or
**"Files on host"** — build from a Dockerfile + context already present on the Builder,
same pattern as adopting an existing Stack. `files_on_host` avoids needing git
credentials in Komodo at all for images that are only ever built where they run.

`RunBuild` **never triggers a Stack redeploy by itself** — building and deploying are
separate concerns in Komodo (the one exception: a `Deployment`, singular-container
resource, explicitly configured with "Redeploy on Build"). To actually run the fresh
image, follow the build with an explicit `DeployStack{services:[...]}` targeting just
that service (see the deploy-lock section above for how to sequence multiple such calls
safely in a Procedure).

## Automating Rebuild + Redeploy on a Schedule

Combine a `Procedure` (multi-stage automation) with a `Schedule`:

```toml
[[procedure]]
name = "Rebuild custom images"
[procedure.config]
schedule_format = "English"        # or "Cron" — see below
schedule = "Every day at 04:00"
schedule_enabled = true
schedule_alert = false             # true = alert on every run, even success
failure_alert = true               # alert only when the procedure fails

[[procedure.config.stage]]
name = "Build"
executions = [
  { execution.type = "RunBuild", execution.params.build = "app-a" },
  { execution.type = "RunBuild", execution.params.build = "app-b" },
]

[[procedure.config.stage]]
name = "Deploy"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["app-a"] } },
  { execution.type = "DeployStack", execution.params = { stack = "stack-b", services = ["app-b"] } },
]
```

Notes:
- `schedule_format = "English"` accepts phrases like `"Every day at 04:00"`,
  `"Every 5 minutes"`, `"At midnight on the 1st and 15th of the month"` — there is no
  native "every N weeks" expression in either English or Cron format; approximate it
  with two fixed days per month if needed.
- A daily cadence for image rebuilds is normally cheap even when nothing changed:
  Docker's build cache makes an unchanged base a near-instant no-op, and
  `docker compose up` is itself a no-op when the resulting image is byte-identical, so
  there's rarely a good reason to default to a sparser schedule "to be safe" — that
  instinct is usually solving a cost problem that doesn't actually exist here.
- A separate built-in Procedure, **"Global Auto Update"** (created automatically on
  install, daily), handles registry-backed images independently via `auto_update` /
  `poll_for_updates` on Stacks/Deployments — that mechanism compares digests **under the
  same tag already in the compose file** (never jumps to a different tag/version), and
  does not go through the same `auto_pull`-gated deploy path described above, so a
  Stack's `auto_pull` setting does not need to be touched to keep that mechanism working.
- Exclude locally-built services from a Stack's `auto_update_skip_services` list — they
  have no registry to poll, so leaving them out avoids meaningless check attempts (though
  it isn't unsafe if you don't).
- The schedule string is evaluated in **Core's container `TZ`**, not UTC — see Schedule
  Collisions below before assuming two differently-worded daily schedules don't overlap.

## Schedule Collisions Across Containers (the timezone trap)

A Procedure's `schedule = "Every day at HH:MM"` (or a Cron schedule) is evaluated in
**the Core container's own `TZ`**, not necessarily UTC and not necessarily the host's
timezone. If Core was started with `TZ=Europe/Paris` (or any non-UTC zone) while some
other daily job on the same host — a backup container's `BACKUP_CRON_EXPRESSION`, a
systemd timer, another Procedure — is defined in **UTC**, "different-looking" schedules
can silently land on the exact same real-world instant every day.

**Symptom:** a scheduled `RunBuild` (or any Procedure) fails intermittently or every
single day at the same clock time, with an error that looks like a network/registry
problem — `dial tcp: lookup auth.docker.io on 127.0.0.11:53: server misbehaving`,
`failed to fetch anonymous token`, timeouts — but:
- the same build succeeds when triggered manually outside that time window,
- a throwaway container on the same network resolves and connects to the registry fine
  right now (see the `alpine`/`curl` test in the IPv6 section above),
- nothing about the network config itself changed.

This is the signature of **resource contention**, not a broken network: two unrelated
daily jobs are firing at the same wall-clock instant, and whichever one needs a fresh
DNS lookup or TCP handshake at that exact moment loses the race under CPU/I-O pressure
from the other (e.g. a `tar`+`gzip` volume backup pegging a small VPS's vCPUs for a
minute or two). The DNS/auth-shaped error is a side effect, not the root cause — do not
"fix" it by touching network config (IPv6, egress rules, etc.) if this is the actual
cause; that just treats the symptom in one incident and leaves the collision to
resurface differently later.

**Diagnosis:**
1. Pull every failing execution's timestamp via `ListUpdates`/`GetUpdate` (see API
   section) and confirm it's the *same* time each day.
2. Check Core's actual timezone: `docker exec <core-container> date` — compare against
   what the schedule string implies in UTC.
3. Check every other daily job on the host for the same real UTC instant: container cron
   env vars (`docker inspect <container> --format '{{range .Config.Env}}{{println .}}
   {{end}}'` grep `CRON`), `systemctl list-timers`, other Procedure schedules
   (`ListProcedures` + `GetProcedure`).
4. Confirm connectivity is fine *outside* the collision window (the `docker run --rm
   --network <egress-network> alpine curl ...` test from the IPv6 section) — this rules
   out a persistent network/config regression and points at timing instead.

**Fix:** stagger one of the two schedules so they no longer land on the same instant —
changing the Komodo side is usually simplest, via `UpdateProcedure`:

```bash
curl -X POST https://<your-komodo-host>/write \
  -H "X-Api-Key: $KOMODO_API_KEY" -H "X-Api-Secret: $KOMODO_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"type":"UpdateProcedure","params":{"id":"<procedure-id>","config":{"schedule":"Every day at 05:00"}}}'
```

A gap of even 15-20 minutes is enough for a short (~1-2 min) colliding job; pick a wider
gap if the other job's duration is unknown or variable.

## Networking Requirements for Periphery

Periphery needs the Docker socket (root-equivalent — same trust level as any tool doing
this, e.g. Watchtower/Portainer) and, if it will ever run `RunBuild`, **outbound internet
access on its own container network** — `docker build` invoked through Periphery resolves
DNS and fetches base-image layers using Periphery's own network namespace, not
transparently through the host's network the way `docker compose up` for already-cached
images does. An `internal: true` (no-egress) network on Periphery will make any build
fail with DNS errors while deploys of already-pulled images keep working fine — that
asymmetry is what makes this confusing to diagnose.

Give Periphery a **dedicated** network with real egress for this, rather than reusing a
bridge network shared with public-facing containers (reverse proxy, web apps). Periphery
holds the Docker socket; co-locating it on the same L2 segment as internet-facing
containers increases the blast radius if one of those is ever compromised, even though
Periphery authenticates inbound connections by keypair. A separate outbound-only bridge
network gives Periphery internet access with zero network-level reachability from (or to)
the public-facing containers.

## IPv6 on Periphery's Egress Network — Enabling It Without Breaking Isolation

If Periphery's dedicated egress network (previous section) has IPv6 disabled
(`EnableIPv6: false`, the Docker default for a new bridge network), a subtle failure
shows up: Docker's embedded DNS resolver proxies the host's own upstream resolvers, so
lookups for `auth.docker.io` / `registry-1.docker.io` can return AAAA records even though
the network has no IPv6 route. Periphery then dials that IPv6 address and fails with
`network is unreachable` — a routing failure, not a DNS one. It surfaces specifically in
the built-in "Global Auto Update" Procedure's nightly check (`CheckStackForUpdateInner`
in Core's logs), one failure line per affected service, every service check otherwise
unrelated to it.

**It's safe to enable IPv6 on this network without weakening isolation.** As of Docker
Engine 27.0.1+, `ip6tables`-based NAT66 is enabled **by default** on user-defined bridge
networks (`com.docker.network.bridge.gateway_mode_ipv6=nat`), giving a private ULA
(`fd00::/8`) subnet outbound-only internet access — the same non-routable-from-outside
guarantee IPv4 NAT already provides. Bridge networks stay mutually isolated regardless of
whether either side has IPv6 enabled; the only real requirement is picking a
**non-overlapping** ULA subnet from any other IPv6-enabled network on the host (e.g. a
public-facing one):

```yaml
networks:
  egress:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: fd00:c0de:cafe:2::/64   # distinct from any other IPv6-enabled network's ULA
```

Avoid the alternative `gateway_mode_ipv6=routed` mode — that assigns containers
globally-routable addresses directly (no NAT), which genuinely would expose them if the
host routes to that prefix. Leave it unset; NAT is the default and what an egress-only
network wants.

**Applying the change:** `enable_ipv6` can't be flipped on a live network — Docker
requires recreating it: `docker compose stop periphery` → `docker network rm <network>` →
`docker compose up -d`. Watch for one more trap: if the container was only *stopped* (not
removed), it can retain a reference to the old network ID and fail to start once the
network is recreated (`network <old-id> not found`) — run `docker compose rm -f
periphery` before the final `up -d` to clear it.

**Verifying it worked:** `docker exec` into Periphery can itself fail with an AppArmor
error if it bind-mounts `/proc:ro` (common, e.g. for monitoring) — that's unrelated to
networking, don't mistake it for a connectivity problem. Test with a throwaway container
on the same network instead:

```bash
docker run --rm --network <egress-network> alpine sh -c \
  "apk add --no-cache curl >/dev/null; curl -6 -m 8 -sS -o /dev/null -w '%{http_code}\n' https://auth.docker.io/"
```

Any real HTTP response (even a 404 from the token endpoint) confirms the TCP/TLS path
works; `network is unreachable` means it's still broken.

## Using the API / CLI Instead of the UI

The `km` CLI ships inside the Core image. Run it via
`docker exec <core-container> km ...` — invoked this way it can read Core's own database
config directly, no separate credentials needed for that specific context.

To script the HTTP API (e.g. to create resources without hand-filling UI forms, which is
easy to mistype on multi-field forms like Procedure stages):

```bash
# One-time: mint a key for an existing user, straight from the DB (no browser/session needed).
# Store both fields directly into a chmod-600 file or env vars — never echo/print/log a
# key or secret value; treat them as write-only from the moment they're generated.
docker exec <core-container> km create api-key "my-automation" --for <username> \
  > /path/to/credentials-file   # chmod 600 this file immediately, before reading it
```

```bash
# Wire protocol confirmed empirically: POST to /read, /write, /execute with these headers.
# Source credentials from env vars (or the file above) — never inline a literal key/secret
# in a command, and never have the agent print/output a key or secret value verbatim.
curl -X POST https://<your-komodo-host>/write \
  -H "X-Api-Key: $KOMODO_API_KEY" \
  -H "X-Api-Secret: $KOMODO_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"type":"CreateProcedure","params":{"name":"...", "config": {...}}}'
```

Every write/execute action name (`CreateProcedure`, `RunProcedure`, `DeployStack`,
`RunBuild`, `UpdateProcedure`, ...) and its exact param shape is defined as a Rust struct
in `client/core/rs/src/api/{read,write,execute}/*.rs` in Komodo's own repo — reading the
struct definition directly is faster and more reliable than guessing from the docs site,
which lags behind and omits some fields (e.g. `EnabledExecution` wraps each execution as
`{"execution": {"type": ..., "params": {...}}, "enabled": true}` — not the flatter shape
the TOML sync examples might suggest at a glance).

Async executions (`RunProcedure`, `RunBuild`, `DeployStack`, ...) return immediately with
`status: "InProgress"` and an `_id`. Poll with `GetUpdate{id}` until `status: "Complete"`,
then check `success` — a top-level `success: true` on a multi-stage Procedure does not
guarantee every individual execution inside it succeeded silently; read the `logs[]`
array, and follow any `see update '<id>'` reference in an error trace to get the specific
failing execution's own detailed log.

## Repo Resources — Git-Triggered Automation (PullRepo, adopting an existing clone)

Komodo has a first-class `Repo` resource type (separate from `Stack`/`Build`) for
tracking a git repository on a Server, with `CloneRepo`/`PullRepo` executions. Confirmed
by reading Komodo's own Rust source (`client/core/rs/src/entities/repo.rs`,
`bin/periphery/src/api/git.rs`, `lib/git/src/pull.rs`) rather than the docs site, which
doesn't go this deep:

- **HTTPS only — Komodo does not support cloning/pulling over SSH at all**, confirmed in
  the `RepoConfig.git_account` doc comment. If you already have an SSH-based clone (e.g.
  a deploy key you set up yourself), it will not be reused — see next point.
- **`PullRepo` unconditionally runs `git remote set-url origin <url>` before every
  pull**, rebuilding the remote URL from the Repo resource's own config
  (`git_provider`/`git_account`/`repo`) every single time — this **overwrites** any
  existing remote, including one pointed at an SSH alias. Don't hand-configure a remote
  and expect Komodo to leave it alone.
- **The access token is embedded directly in the URL string** (`remote_url()` in
  `client/core/rs/src/entities/mod.rs`: `https://<user>:<token>@<provider>/<repo>`) and
  that exact URL gets written to the repo's `.git/config` on disk via `git remote
  set-url` — the token therefore sits in **plaintext on the Periphery host filesystem**
  after the first pull, not just inside Komodo's own DB. Same trust level as any other
  secret already living in a Periphery-mounted directory, but worth knowing before
  assuming the token stays encrypted-at-rest everywhere.
- **`RepoConfig.path`**: if absolute (leading `/`), used directly as the clone path — set
  this to an **already-existing** git checkout's path to have Komodo adopt and manage it
  going forward (same "adopt, don't migrate" philosophy as Stacks' "Files on Server"
  mode). `PullRepo` checks for a `.git` dir first; if present, it pulls in place instead
  of re-cloning.
- **Credentials live in `git_providers` config**, not per-Repo — configured once in
  Periphery's (or Core's) config file/env as a named account for a given provider
  domain, then referenced from a Repo resource's `git_account` field. Fine-grained,
  read-only GitHub PATs scoped to one repo are the natural fit here (no SSH option
  exists to prefer instead).

## Webhooks — URL Pattern and What Each Resource Type Supports

Confirmed from Komodo's own docs (`docsite/docs/automate/webhooks.md`) — more capable
than it first appears; don't assume a Procedure is the only resource that can receive a
webhook:

```
https://<HOST>/listener/<AUTH_TYPE>/<RESOURCE_TYPE>/<ID_OR_NAME>/<EXECUTION>
```

| Resource | Available executions |
|---|---|
| Build | `/build` |
| Repo | `/pull`, `/clone`, `/build` |
| Stack | `/deploy`, `/refresh` |
| Resource Sync | `/sync`, `/refresh` |
| Procedure / Action | branch name to listen for (e.g. `/main`, or `/master`), or `/__ANY__` for all branches |

- `AUTH_TYPE` is `github` (validates `X-Hub-Signature-256`, also covers Gitea/Forgejo) or
  `gitlab` (validates `X-Gitlab-Token`).
- **One shared secret for all webhooks on an instance**: `KOMODO_WEBHOOK_SECRET` in
  Core's config/env — set once, used to validate every incoming webhook regardless of
  resource type. Check `core_config().webhook_secret` (or the corresponding env var on
  the Core container) before assuming you need to generate a new one per resource.
- **A Repo's `/pull` and a Stack's `/deploy` are independent webhooks** — if you need
  "pull, then redeploy" in a guaranteed order from a single git push, don't wire two
  separate webhooks to a git provider and hope for correct sequencing (delivery order
  isn't guaranteed). Instead, put `PullRepo` then `DeployStack` as two sequential stages
  in **one Procedure**, and point the git provider's webhook at that Procedure's own
  `/listener/github/procedure/<id>/<branch>` URL — this is also how the branch filter
  becomes useful (only the branch that receives real merges triggers the Procedure at
  all, so PR/feature-branch pushes from a bot like Renovate never fire it).

## Common Mistakes

- **Assuming a Stack `Delete` is always safe because it was safe once.** Test in a
  low-stakes context, or avoid entirely — see the Danger section above.
- **Setting `pull_policy: build` at the Stack level.** It's a Docker Compose per-service
  field, not a Komodo Stack setting — it belongs in the compose YAML.
- **Parallelizing two deploys on the same Stack "for speed."** Costs you a confusing
  `"Resource is busy"` failure instead of a few seconds saved.
- **Trusting a compose service's apparent creation timestamp as proof a build actually
  ran.** Docker's content-addressable image store reuses the same image ID (and its
  original timestamp) when a fresh build produces byte-identical layers to a prior one —
  an old-looking timestamp after a successful `RunBuild` is normal, not evidence of
  failure. Verify content directly (e.g. run the binary and check its reported version)
  if you need certainty, not the image's `Created` field alone.
- **Reading only the top-level Procedure result.** Errors from one failed execution in a
  parallel stage can overshadow siblings that also failed; always drill into individual
  execution logs when a stage reports a failure.
- **Chasing a network/DNS fix for a scheduled build failure without checking for a
  colliding job first.** A DNS or registry-auth-looking error that only happens at one
  fixed time of day is a resource-contention symptom, not evidence the network config
  regressed — compare every daily job's *real* UTC time (accounting for each container's
  own `TZ`) before touching IPv6/egress settings again. See Schedule Collisions above.
