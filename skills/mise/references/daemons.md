## Project Daemons (`[daemons]`)

**Contents**

- [Declaring Daemons](#declaring-daemons)
- [Tasks That Require Daemons](#tasks-that-require-daemons)
- [Groups, Namespaces, and Cross-Project Daemons](#groups-namespaces-and-cross-project-daemons)
- [Shared Server Providers ([daemon_providers])](#shared-server-providers-daemon_providers)
- [Service Presets](#service-presets)
- [Container Daemons](#container-daemons)
- [Ports and Worktrees](#ports-and-worktrees)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

**Experimental** (requires `experimental = true`) and requires **pitchfork** for process supervision. Added in **2026.9.6** and substantially expanded in **2026.9.12**.

Daemons are processes that keep running *between* task invocations — a database, message broker, or dev server. mise supplies the project configuration and tool environment; pitchfork owns the processes and readiness checks.

```toml
[settings]
experimental = true

[tools]
node = "24"

[daemons]
postgres = "18"

[tasks.dev]
daemons = "postgres"
run = "npm run dev"
```

`mise run dev` installs missing tools, starts PostgreSQL, waits for readiness, then runs the task. The preset exports `DATABASE_URL` and friends and keeps data between runs. The daemon stays up after the task exits.

### Declaring Daemons

Exactly one declaration style per entry:

| Declaration | Meaning |
|-------------|---------|
| `postgres = "18"` | Service preset whose **name matches the key** |
| `preset = "postgres"` + `version = "18"` | Named instance of a preset, with overrides |
| `run = "exec npm run dev"` | Custom shell command |
| `task = "dev:core"` | An existing mise task (optionally with `args`) |
| `project = "../other"` (+ `name`) | A daemon declared by **another project** |
| `provider = "local-postgres"` (+ `resource`) | A database/account on a **shared server** from global `[daemon_providers]` (2026.9.13) — see [Shared Server Providers](#shared-server-providers-daemon_providers) |
| `image = "postgres:18"` | A container run with Docker or Podman (2026.10.24) — see [Container Daemons](#container-daemons) |

```toml
[daemons.api]
run = "exec npm run dev"
ready_port = 3000

[daemons.analytics]
preset = "postgres"
version = "18"
port = 5433
```

| Field | Type | Notes |
|-------|------|-------|
| `run` | string | Long-running command. Use `exec` so it receives stop signals directly. Mutually exclusive with `task`/`preset`. Since **2026.10.1** it may use `{{ env.X }}` / `{{ vars.X }}` (rendered by `mise x` at launch; `[env]` values are never written into the generated pitchfork file; needs pitchfork > 2.29.0). Pitchfork's own `{{ port }}`/`{{ url }}` still work. |
| `task` | string | mise task to supervise. `args` requires it. mise is already the entry point, so it is not wrapped again unless `init` is also set. |
| `args` | string[] | Passed to the task after `--`. Requires `task`. |
| `preset` | enum | `cockroachdb` \| `nats` \| `postgres` \| `redis` \| `spicedb`. Requires `version`. |
| `version` | string | Version request for the preset's tool. |
| `init` | string \| string[] | Idempotent setup run **before** the long-running process, on every start and restart. Shares a shell with the main command. |
| `mise` | bool | `false` disables the mise tool-environment wrapper for both setup and the main command. |
| `port` | int \| `"auto"` \| table | Fixed port, or per-worktree allocation. `run`/`task` daemons also accept pitchfork's structured port table (container daemons reject it). |
| `ports` | table | A preset's additional named ports (integer, or `{ auto = true, base, stride }` per worktree since 2026.10.24), or a container's extra published ports. |
| `options` | table | Preset-specific options. |
| `data_dir` | string | Persistent preset data directory (or the host side of a container's `data_mount`), relative to the declaring project root or absolute. Defaults to mise state storage. |
| `proxy` | string \| bool | Reverse-proxy hostname label; `false` opts out, `true` uses the daemon's name. Only a daemon with a port is routed. |
| `proxy_tls` | enum | `terminate` \| `passthrough`. |
| `proxy_idle_timeout` | duration \| `false` | **2026.9.18** (pitchfork ≥ 2.27.0). Stop a **proxy-started** daemon after this long without traffic (`"30m"`, `"1h"`). Explicit starts (`mise daemons start`, shell hook, `boot_start`) and their deps are exempt; dependencies without their own value inherit the requester's; `false` (or `"0"`) exempts even from a global default (`PITCHFORK_PROXY_IDLE_TIMEOUT`). Open connections count as activity; direct-port traffic does not. |

Readiness is configured with pitchfork's own fields (`ready_port`, `ready_cmd`, `health_cmd`, `auto`, …). Since 2026.10.5 `ready_cmd`/`health_cmd` run **through `mise x`** (so project tools are on PATH) and may be argument arrays; `depends` references are rewritten to full daemon IDs. `auto = ["start", "stop"]` starts/stops daemons with the shell (needs `experimental` in **global** config). A preset daemon also accepts a `tool` key. Readiness checks apply **after** `init`, so anything waiting on the daemon also waits for setup.

> **Minimum pitchfork versions:** 2.25.0 base · 2.26.0 proxy · 2.27.0 `proxy_idle_timeout` · 2.28.0 task daemons · 2.30.0 `{{ env.X }}`/`{{ vars.X }}` in `run`. Docs split into `/daemons/{data,presets,sharing,worktrees}.html`.

### Tasks That Require Daemons

```toml
[daemons]
postgres = "18"
redis = "8"

[tasks.test]
daemons = ["postgres", "redis"]
run = "npm test"
```

Use a string for one, a list for several, or `true` for every daemon in the task's project configuration. Already-running daemons are reused. This replaces prerequisite tasks that launch background processes and poll for readiness.

- Startup is part of **dependency handling**: `--skip-deps` and `task.skip_depends` skip it.
- `--dry-run` validates names and the experimental setting, and reports what would start without starting anything.
- **Safe mode blocks task daemon startup.**
- In a monorepo each task resolves daemon names in its own project's hierarchy, including inherited declarations. `true` covers only the project's own daemons — it never reaches into a referenced project.
- Inspect with `mise tasks info <task>`.

> ⚠️ **Subtasks do not start daemons.** The requirement is honored for tasks a run resolves up front (including their `depends`). A subtask reached through a `run = [{ task = "…" }]` entry is resolved once the run is already executing, so its own `daemons` are **not** started. Declare the requirement on the task you invoke.

> A daemon's `task` is invoked via `mise run`, so that task's `depends` run first — but its own `daemons` are skipped, since starting those would restart this daemon.

### Groups, Namespaces, and Cross-Project Daemons

```toml
[daemon_groups]
default = ["postgres", "nats", "core"]
two-cluster = ["default", "core2"]        # groups expand in place

# equivalent table form
[daemon_groups.two-cluster]
daemons = ["default", "core2"]
```

```bash
mise daemons start two-cluster
mise daemons stop --group two-cluster
```

Members must be daemons or groups **in the same project** — a group can never select outside it. A group name may not repeat a daemon name in the same project, and needs at least one member. `mise daemons start` with no names starts the `default` group when one exists, otherwise every project daemon; `stop` without names always covers every project daemon. A bare positional name selects that project's group first, then a daemon of that name; `--group` only ever selects a group.

```toml
[daemons_settings]
namespace = "services"            # fixed pitchfork namespace, else derived from dir name + path hash
namespace_per_worktree = true     # default; appends a 16-hex suffix in a linked git worktree
```

A daemon's full ID is `<namespace>/<name>`. Namespace resolution order: `[daemons_settings].namespace` → the project's `pitchfork.toml` → the generated default. Stop the project's daemons before changing its namespace. `[daemons_settings]` **merges per key** across config files (unlike `[daemons.<name>]`, where a higher-precedence declaration replaces the whole definition). **Global and system config cannot set `[daemons_settings]`** — mise ignores those tables with a warning.

```toml
# run a daemon defined in another checkout
[daemons.pipeline]
project = "../mirror-pipeline"
name = "worker"

[daemons.api]
run = "npm run dev"
depends = ["pipeline"]
```

The referenced checkout must already be **trusted** — `mise run` and `mise daemons start` never trust it for you. A reference table accepts only `project` and `name`, and cannot point at another reference. The imported daemon runs in *its own* project's environment, so a preset's `DATABASE_URL` does **not** join your application's environment. A missing or untrusted checkout drops the import rather than breaking the rest of your config; naming it explicitly fails with the reason.

### Shared Server Providers (`[daemon_providers]`)

**Experimental, 2026.9.13.** Instead of one PostgreSQL per checkout, run **one shared server per machine** and give each project its own database (or NATS account) on it. Providers are declared **only in global config**:

```toml
# ~/.config/mise/config.toml
[daemon_providers.local-postgres]
preset = "postgres"        # required: postgres | cockroachdb | nats  (no redis / spicedb)
version = "18"             # required
port = "auto"              # the default; also ports, options, data_dir, tool
```

Projects consume a provider — a provider reference accepts **only** `provider` and `resource`:

```toml
# project mise.toml
[daemons.db]
provider = "local-postgres"   # gets a checkout-specific database (derived from checkout path + daemon name)
# resource = "shared_app"     # ^[a-z][a-z0-9_]{0,62}$ — same name in several consumers = one shared DB
```

- Starting a consumer (`mise daemons start`, a task with `daemons = ["db"]`, or `depends = ["db"]`) waits for the server **and** the database to be provisioned. mise never runs migrations or deletes databases. The usual preset vars (`DATABASE_URL`, …) point at the selected database; explicit `[env]` wins.
- Providers run with their own tools and a **minimal env** (no project env/tools), are never idle-stopped, and aren't part of project groups. Data lives in `$MISE_STATE_DIR/daemon-providers/<name>/data`; pruning projects never deletes it. Change settings in global config, then `mise daemons providers restart <name>`.
- **NATS** providers give each resource its own **account** (separate subjects and JetStream). `NATS_URL` then embeds that account's username/password — treat it as a credential. Custom NATS config files and TLS are rejected for providers.
- A missing provider is an error (never auto-selected). Local superuser auth — **not a security boundary**. Shared server providers are **not supported on Windows**.

### Service Presets

Presets supply the command, required tool, readiness check, data directory, and connection variables. Preset tools install on the first `mise daemons start` (2026.9.18). NATS and SpiceDB also need `curl` on PATH for their HTTP readiness checks.

> **Windows (2026.10.2):** every preset **except `redis`** (no Windows build) now runs on Windows under pitchfork's default `cmd /C` (a custom `windows_shell` is unsupported). PostgreSQL stops cleanly only with pitchfork ≥ 2.29.0 and refuses to run elevated. Hyper-V/WSL reserved port ranges (`netsh interface ipv4 show excludedportrange protocol=tcp`) may force an explicit `port`. Task daemons start **without a shell** since 2026.10.1 (pitchfork ≥ 2.28.0) so args arrive exactly — except a task daemon with `init`, which shares one shell (`cmd /C` on Windows).

> **PostgreSQL refuses to run as root (2026.9.13)** — starting a postgres daemon or provider as root fails before installing anything. In containers, run mise as a regular `USER`.

| Preset | Tool | Default port | Additional listeners | Exports |
|--------|------|--------------|----------------------|---------|
| `postgres` | `postgres` | 5432 | — | `PGHOST`, `PGPORT`, `PGUSER`, `PGDATABASE`, `DATABASE_URL` |
| `redis` | `redis` | 6379 | — | `REDIS_URL` |
| `cockroachdb` | `cockroach` | 26257 | `http_port` 8080 | `COCKROACH_HOST`, `COCKROACH_URL`, `DATABASE_URL` |
| `nats` | `nats-server` | 4222 | `monitor_port` 8222 | `NATS_URL`, `NATS_MONITORING_URL` |
| `spicedb` | `spicedb` | 50051 | `http_port` 8443, `metrics_port` 9090 | `SPICEDB_ENDPOINT`, `SPICEDB_PRESHARED_KEY` |

**Preset options** (`options.*`; file paths resolve relative to the project root, `~/` expands):

- **postgres** — `database` (created on first init; defaults to `postgres`). Uses the `postgres` user with local trust auth.
- **redis** — append-only persistence; no options.
- **cockroachdb** — `database` (`"defaultdb"`, names the connection default, does **not** create), `databases` (`[]`, created on first init; `name=region` sets a primary region), `settings` (`[]`, each an assignment like `"sql.defaults.vectorize = 'off'"`, prefixed with `SET CLUSTER SETTING`), `locality`, `max_offset`.
- **nats** — `jetstream` (`true`), `config` (a config file takes over JetStream/storage and voids `jetstream`), `tls_cert`/`tls_key` (both required to enable TLS; `NATS_URL` then uses `tls://`), `tls_ca` (requires the other two).
- **spicedb** — `datastore_engine` (`"memory"`), `datastore_uri`, `datastore_daemon` (startup order only — it does **not** make `datastore_uri` follow an automatic port), `preshared_key` (`"mise-dev-key"`). Setting only one of engine/uri is an error. `spicedb migrate head` runs before every start on a persistent datastore.

> 🔴 **Local development only.** Preset defaults use loopback addresses with no authentication or a fixed development key. For shared or untrusted environments, write a custom daemon with real auth and network restrictions.

Options used during **first initialization** (database names, cluster settings) do not modify existing data when changed. An explicit `[tools]` version must satisfy the preset's request (`postgres = "18.1"` satisfies a preset asking for `"18"`); multiple instances sharing a tool must use the same request. Explicit `[env]` values override preset defaults, and when several instances export the same variable the **last declaration wins**.

### Container Daemons

**2026.10.24**, experimental like all daemons (same `experimental = true` + pitchfork requirement). A daemon with `image` runs as `<runtime> run --rm` in the foreground, so its output is `mise daemons logs`. Docs: https://mise.jdx.dev/daemons/containers.html

```toml
[daemons.postgres]
image = "postgres:18"
port = { auto = true, base = 5432 }
env = { POSTGRES_PASSWORD = "dev" }
data_mount = "/var/lib/postgresql"
ready_cmd = "{{ runtime }} exec {{ container }} pg_isready -U postgres"
proxy = false                                   # not HTTP

[daemons.cockroach]
image = "cockroachdb/cockroach:v25.1.0"
command = ["start-single-node", "--insecure"]
port = { auto = true, base = 26257 }
ports.http = { container = 8080, auto = true, base = 27000 }   # exports COCKROACH_HTTP_PORT
data_mount = "/cockroach/cockroach-data"
ready_cmd = "{{ runtime }} exec {{ container }} cockroach sql --insecure -e 'select 1'"
proxy = false
```

| Key | Type | Notes |
|-----|------|-------|
| `image` | string | Makes the daemon a container. Can't combine with `run`, `task`, `preset`, `project`, `args`, or `provider`. mise doesn't pull or build images (`docker run` pulls on first use) and doesn't start Docker Desktop / `podman machine`. |
| `command` | string \| string[] | Replaces the image's default command. A string is shell text (like `run`); an array is passed as-is. |
| `port` | int \| `{ auto = true, base, stride }` | Host port for the main container port. `port = "auto"` alone is **rejected** (needs `base`). |
| `container_port` | int | Port the container listens on for `port`. Defaults to `base` (auto) or the port itself. Requires `port`. |
| `ports.<name>` | int \| `{ container, host?, auto?, base?, stride? }` | Extra published ports. An integer uses the same number on host and container. In a table, `container` is required; `host` is a fixed host port (defaults to `container`); `auto = true` allocates per worktree (`base` defaults to `container`). `host` and `auto` are mutually exclusive. Names use letters, digits, `_` (no leading digit) and can't be `port`, `container`, or `runtime`. Exported as `<NAME>_<PORTNAME>_PORT`. |
| `env` | table | Passed into the container **by name** (`-e VAR`), so values stay off the command line. Project `[env]` is **not** passed in. |
| `volumes` | string[] | `source:target[:options]`. A source starting with `.`, `/`, or `~` is a host path (relative to the project root, created if missing); anything else is a named volume. |
| `data_mount` | string | Absolute path inside the container kept on the host. |
| `data_dir` | string | Host side of `data_mount` (relative to project root or absolute). Default: mise state dir per project root, so each worktree has its own; `mise daemons prune` removes it. Requires `data_mount`. |
| `runtime` | `"auto"` \| `"docker"` \| `"podman"` | `auto` (default) uses docker if on PATH, else podman. `MISE_CONTAINER_RUNTIME` (any docker-compatible CLI, e.g. `nerdctl`) overrides it. Looked up inside the project's tool env, so a `[tools]`-installed runtime works. |
| `runtime_args` | string[] | Extra `<runtime> run` args placed before the image, e.g. `["--shm-size=1g"]`. |

- `command`, `container_port`, `data_mount`, `runtime`, `runtime_args`, and `volumes` on a non-`image` daemon are an error (`[daemons.X].<key> requires image`).
- `init`, `depends`, `ready_port`, `ready_cmd`, `health_cmd`, `proxy`, `proxy_tls`, `proxy_idle_timeout`, `mise`, and `stop_signal` work as on a custom daemon; `init` runs on the **host** before the container starts.
- `ready_cmd`/`health_cmd` placeholders: `{{ container }}`, `{{ runtime }}`, `{{ port }}` (host port of `port`), `{{ NAME }}` (host port of named port `NAME`). `{{ runtime }}` is shell text — use it in a string command or inside `["sh", "-c", "…"]`, not as an argv element. Prefer `ready_cmd` to `ready_port`: the host port opens before the server answers.
- Ports are published on **127.0.0.1 only**, TCP only, and every published host port must differ. Like any custom daemon with a port, a container gets a proxy hostname — set `proxy = false` for databases and brokers.
- Stop runs `<runtime> stop` (10 s grace); mise sets pitchfork's stop timeout to 15 s unless you set `stop_signal` (keep yours above 10 s). A stale container from a killed run is removed before the next start.
- **Not supported on Windows** (the config loads; start fails with a message). Containers can't be shared-server providers.

### Ports and Worktrees

```toml
[daemons.postgres]
preset = "postgres"
version = "18"
port = "auto"                                  # primary checkout keeps 5432

[daemons.api]
run = "exec npm run dev -- --port $API_PORT"
port = { auto = true, base = 3000 }            # base is REQUIRED for custom daemons
```

| Option | Meaning | Default |
|--------|---------|---------|
| `auto` | Enables automatic allocation. Must be `true`. | Required in table form |
| `base` | Port used by the primary checkout | The preset's default; **required for custom daemons** |
| `stride` | Spacing between allocation slots | `1` |

There are **511 worktree offsets**. With defaults, PostgreSQL uses 5433–5943 in linked worktrees and Redis 6380–6890. A preset's named ports move with its primary port; a named port you set to an **integer** is used exactly as written, while `ports.<name> = { auto = true, base = 27000 }` allocates per worktree (2026.10.24; `base` defaults to that port's preset default, and the slot follows the primary port when it is auto). All configured ports must be free — **mise never searches for a free port** — and two daemons in one project cannot share a port (reported at config load, including auto named ports and container ports; collisions across worktrees or projects aren't checked, so keep bases at least 512 apart). Resolved ports are saved in the project's generated `state.json` and reused; changing `base`/`stride` re-resolves, so restart the daemon.

Custom daemons with a port export `<NAME>_PORT` (uppercased, punctuation → underscores): `[daemons.api]` → `API_PORT`, `[daemons.web-ui]` → `WEB_UI_PORT`. Names that would collide (`web-ui` vs `web_ui`, or any two non-preset daemons that would export the same `*_PORT`, such as `api` with `ports.http` and a daemon named `api-http`) or start with a digit get no variable and a warning — the daemon still runs, and pitchfork still injects `$PORT`. Independent clones and non-Git projects keep the base port; worktrees of a bare repo all get offsets.

**Preset named ports (2026.9.18)** export as `<NAME>_<PORT_NAME>`: a `cockroachdb` daemon named `crdb` exports `CRDB_HTTP_PORT`; a `spicedb` named `authz` exports `AUTHZ_HTTP_PORT` and `AUTHZ_METRICS_PORT` — the port actually used (default, worktree offset, or `ports.<name>`).

**Stable URLs per worktree:** every proxied daemon (has a `port`, not `proxy = false`) also exports **`<NAME>_URL`** — its hostname URL, which never changes when a port moves — so services can point at each other without port arithmetic:

```toml
[daemons.api]
run = "exec npm run dev -- --port $API_PORT"
port = { auto = true, base = 3000 }

[env]
APP_BASE_URL = "{{ env.API_URL }}"
```

Set `proxy = false` on any custom daemon that doesn't speak HTTP (**every preset** opts out already and keeps exporting its connection vars such as `PGPORT`/`DATABASE_URL`/`REDIS_URL`). When a port is taken, mise's warning names the daemon, the port, and whether it is the base or a worktree offset.

```bash
mise daemons                 # list configured + previously managed daemons
mise daemons ls --json       # includes resolved `port` and `port_auto`
mise daemons start [NAME…]   # installs missing tools
mise daemons start --all     # 2026.9.18: every project daemon (incl. ones a `default` group omits);
                             # never reaches other projects; not combinable with names/--group
mise daemons stop | restart | status | logs | tui
mise daemons providers ls [--json] | start | stop | restart [NAMES]…   # shared servers (2026.9.13)
mise daemons register        # prepare for on-demand startup without starting
mise daemons urls            # each daemon's port and proxy hostname URL
mise daemons prune [-n]      # drop state from deleted project directories
```

> `mise daemons start|restart|register` implicitly trust the project config (like `mise run`); safe mode refuses `mise daemons` and ignores daemon declarations.

Hostnames are `<proxy>.<project>.<tld>` in the primary checkout and `<proxy>.<worktree>.<project>.<tld>` in a linked worktree.
