## Task Configuration Reference

**Contents**

- [All Task Fields](#all-task-fields)
- [Structured run Array](#structured-run-array)
- [Structured depends with Args/Env](#structured-depends-with-argsenv)
- [Task Templates (extends)](#task-templates-extends)
- [Global Task Configuration](#global-task-configuration)
- [Task-Scoped Secrets (Experimental)](#task-scoped-secrets-experimental)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### All Task Fields

#### Core Execution

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `run` | `string \| string[] \| ({task: string, args?: string[], env?: object} \| {tasks: string[]} \| string)[]` | — | Command(s) to execute. The only required property. Shorthand task arrays may mix entries: `tasks.ci = [{ task = "lint" }, "echo done"]` (2026.10.7). |
| `run_windows` | same as `run` | — | Windows-specific override. **`run_windows = []` skips the task on Windows** (2026.10.18). |
| `file` | `string` | — | External script path (local, HTTP, or Git URL) |
| `shell` | `string` | `task_config.shell`, else `sh -o errexit -c` (Unix) / `cmd /c` (Windows) | Interpreter. TOML-tasks only. E.g., `"bash -c"`, `"node -e"`. Since 2026.9.3, *simple* inline commands on Unix (`node build.js`) run **directly without `sh`** unless a `shell` is set explicitly, or the command uses shell syntax, builtins, or sandboxing. |
| `usage` | `string` | — | Usage spec for arguments/flags. TOML-tasks only. |

#### Metadata

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `description` | `string` | — | Task help text |
| `alias` | `string \| string[]` | — | Alternative name(s) |
| `hide` | `bool` | `false` | Exclude from listings |

#### Dependencies

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `depends` | `string \| {task, args?, env?, optional?} \| (string \| string[] \| {task, args?, env?, optional?})[]` | — | Tasks to run BEFORE. An inner `string[]` is `[task, ...args]` (non-empty). A **single table** is accepted (2026.10.7): `depends = { task = "build", args = ["x"] }`. Parallel by default. |
| `depends_post` | same as `depends` | — | Tasks to run AFTER this task. Runs if the parent **started** (even if it failed), but is **skipped when a regular dependency fails** before the parent starts. |
| `wait_for` | same as `depends` | — | Wait for tasks without adding them as deps. A `wait_for` naming a **nonexistent task errors** unless `optional = true`. |

#### Environment & Tools

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `env` | `table` | — | Task-specific env vars (**NOT passed to depends**). Supports sops/age-encrypted values and `_.file`. |
| `pass_through_env` | `string[]` | — | **Experimental.** Ambient env vars (wildcards allowed) passed through **without** affecting the cache key (tokens, CI vars). Only matters under env sandboxing (`deny_env`/`deny_all`/`allow_env`) — without it every ambient var passes anyway. |
| `tools` | `table` | — | Tools to install before running |
| `vars` | `table` | — | Task-local vars that override `[vars]` |
| `daemons` | `bool \| string \| string[]` | — | **Experimental (2026.9.12+).** Project daemons that must be running and ready before the task body runs. `true` = every daemon in this project's `[daemons]`. See [Tasks That Require Daemons](daemons.md#tasks-that-require-daemons). |
| `secrets` | `string \| string[]` | — | **Experimental (2026.10.4).** fnox keys (`^[A-Za-z_][A-Za-z0-9_]*$`) granted to **this task only** — never inherited by `depends`, `depends_post`, or `run = [{ task }]` subtasks. Needs top-level `[secrets.fnox]` and `experimental = true`. See [Task-Scoped Secrets](#task-scoped-secrets-experimental). |

#### Execution Context

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dir` | `string` | `{{config_root}}` | Working directory. Use `{{cwd}}` for user's cwd. |
| `raw` | `bool` | `false` | Direct stdin/stdout connection. Takes an **exclusive lock per command** (nothing else runs alongside each of its commands; other tasks can run between them). Does **not** set jobs=1 — that is `mise run --raw`. Bypasses redactions and the artifact cache; not stopped by the whole-run timeout. |
| `raw_args` | `bool` | `false` | Pass all args verbatim including `--help`/`-h` to underlying command |
| `interactive` | `bool` | `false` | Exclusive lock held from the task's first to its last script command — every other task waits (verified; the schema description saying others "still run in parallel" is wrong). The lock is **released while a `{ task }`/`{ tasks }` run step executes**. Bypasses redactions and the artifact cache. |
| `skip_if` | `string` | — | **2026.10.10.** Shell command run **after `depends`, before `confirm`**, in the task's dir/shell/env/sandbox. **Exit 0 ⇒ the task is skipped** (prints `[x] skip_if condition met, skipping`) and counts as success; any other exit runs the task. Output discarded, no stdin, no `secrets`; shares the task `timeout` (failure to start / timeout ⇒ task fails). Tera-rendered (`usage.*` available); works in `#MISE` headers. `--force` bypasses it; `--dry-run` prints `skip_if: … (not checked: dry run)`. A skipped task does **not** invalidate dependents. |

#### Output Control

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `output` | `"prefix" \| "interleave" \| "keep-order" \| "replacing" \| "timed" \| "quiet" \| "silent"` | inherits `task.output` | **Per-task output style** (2026.7.6+). Orthogonal to `quiet`/`silent`. |
| `quiet` | `bool` | `false` | Suppress mise's own chatter (echoed command). **No longer changes the output style.** |
| `silent` | `bool \| "stdout" \| "stderr"` | `false` | Suppress all/specific task output |

#### Freshness & Caching

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sources` | `string \| string[]` | — | Input files (globs, `{a,b}` braces, `!` exclusions, `@group:name` refs) |
| `outputs` | `string \| string[] \| {auto: bool}` | `{auto: true}` (if `sources` set) | Generated files. Supports ordered `!` exclusions. `outputs = []` enables result-only caching. `{ auto = false }` is accepted and means the same as unset (2026.10.8). |
| `cache` | `{enabled, audit, env, command_inputs}` | `{enabled=false, audit=false, env=[], command_inputs=[]}` | **Experimental.** Task output artifact cache — see [Task Output Caching](task-caching.md#task-output-caching-experimental). |
| `rust_cache` | `bool \| {enabled?, verify?}` | `false` | 🔴 **Deprecated NO-OP** (removal 2027.8.14); an enabled value prints a migration warning. Use [mbx](task-caching.md#rust-compiler-action-cache--removed-rust_cache-is-a-no-op). |
| `watch` | `{no_vcs_ignore: bool}` | — | `no_vcs_ignore = true` watches sources excluded by `.gitignore` (2026.8.0+) |

#### Safety & Sandboxing

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `confirm` | `string \| {message, default?, yes?, no?}` | — | Prompt before running. `default` (optional since 2026.10.3, defaults to yes) accepts `yes`/`no`/`y`/`n`/`true`/`false`; `yes`/`no` are display labels (2026.10.3). All support Tera with `usage.*`. **Guards only `run`, not `depends`.** `-y/--yes` skips it; with **no TTY at all** (CI, agent shells) it fails: `task requires confirmation but there was nobody to ask; pass --yes to accept`. |
| `deny_all` | `bool` | `false` | Block reads, writes, network, and env inheritance |
| `deny_read` / `deny_write` / `deny_net` / `deny_env` | `bool` | `false` | Individual sandbox blocks |
| `allow_read` / `allow_write` | `string[]` | — | Path allowlists |
| `allow_net` | `string[]` | — | Host allowlist |
| `allow_env` | `string[]` | — | Env var allowlist (wildcards: `NODE_*`) |

These are **flat top-level task fields**, not a nested `sandbox` table.

> 🔴 **`redactions` is NOT a task field.** `[tasks.x] redactions = [...]` is a hard parse error (`unknown field 'redactions'`). It is a **top-level** key only — see [Redactions](task-dependencies.md#redactions-experimental).

#### Timeout & Inheritance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `timeout` | `string` | — | Max duration **per command** (each `run` entry, and the file-task script — verified: `run = ["sleep 1.5","sleep 1.5"]` with `timeout = "2s"` succeeds); a `{ task = … }` step uses the referenced task's own timeout. The global `task.timeout` is a **whole-run** limit — when both apply the shorter wins; `--timeout` overrides. Tera supported. On expiry: SIGTERM, then SIGKILL after 5s (Windows: Ctrl+C, 5s grace, then tree kill). A timed-out task **fails even if it exits 0** (2026.10.2). Accepted in `#MISE` headers since 2026.10.14. |
| `extends` | `string` | — | Name of a `[task_templates.*]` entry to inherit from (single string; no multiple inheritance) |

### Structured `run` Array

Mix inline scripts with task references, including parallel sub-task execution:

```toml
[tasks.pipeline]
run = [
  { task = "lint" },                 # run lint (with its dependencies)
  { task = "build", args = ["--release"], env = { RUSTFLAGS = "-C opt-level=3" } },
  { tasks = ["test", "typecheck"] }, # run test and typecheck in parallel
  "echo 'All checks passed!'",       # then run a script
]
```

### Structured `depends` with Args/Env

Pass arguments and environment variables to dependencies:

```toml
[tasks.deploy]
depends = [
  { task = "build", args = ["--release"], env = { RUSTFLAGS = "-C opt-level=3" } }
]
run = "./deploy.sh"
```

Shell-style inline env form also supported:
```toml
depends = ["NODE_ENV=test setup"]
```

Forward parent args via Tera `usage.*`:
```toml
[tasks.deploy]
usage = 'arg "<app>"'
depends = [{ task = "build", args = ["{{usage.app}}"] }]
run = 'echo "deploying {{usage.app}}"'
```

> Prefer this structured form for templated args. The string form `depends = ["build {{usage.app}}"]` runs fine, but `mise tasks validate` (2026.10.3) reports it as `missing-dependency` because it checks the unrendered name — a false failure if you validate in CI.

Wildcards work too: `depends = ["lint:*"]`.

**Optional dependencies (2026.7.17+):** `optional = true` runs all matches when present and is silently omitted when nothing matches. Invalid selectors still error. Works across `depends`, `depends_post`, and `wait_for`.
```toml
depends = [{ task = "lint:*", optional = true }]
```

**Conditional dependencies (2026.7.8+):** a dependency whose **templated name renders empty is skipped**, and empty templated dependency *args* are omitted.

**`wait_for` matching:** by name alone matches regardless of args/env; with explicit args/env it must match exactly.

Env vars passed to dependencies are scoped to that dependency only.

### Task Templates (`extends`)

Reusable task definitions live in `[task_templates.*]`; tasks inherit them with `extends`. Template names may be namespaced with `:`.

```toml
[task_templates."python:build"]
description = "Build a Python project"
run = "uv build"
tools = { python = "3.12", uv = "latest" }
env = { PYTHONPATH = "src" }

[tasks.build]
extends = "python:build"

[tasks.test]
extends = "python:test"
run = "pytest --cov"       # overrides the template's run
```

**Merge rules:**

| Behavior | Fields |
|----------|--------|
| **Full override** (task wins, no merging) | `run`, `run_windows`, `depends`, `depends_post`, `wait_for`, `sources`, `outputs`, `cache`, `output`, `alias` |
| **Override if set locally** | `dir`, `description`, `shell`, `timeout`, `silent`, `confirm`, `watch`, `daemons`, `file` (`run` is ignored when either side sets `file`) |
| **Deep merge** (per key; task wins) | `tools`, `env`, `vars` |
| **Concatenated** | `usage` — **template's flags first, then the task's** (2026.9.11+) |
| **Compose / combine** | sandbox `deny_*` compose; `allow_*` and `pass_through_env` combine |
| **Not accepted on templates** | `hide`, `interactive`, `quiet`, `raw`, `raw_args` (warn as unknown — set per task); `extends` (no chaining); **`secrets` is a config error** |

Templates from every loaded config (global and parent included) form **one namespace** — the highest-precedence definition of a name wins.

> `depends` **replaces** rather than merges: template `["lint","typecheck"]` + task `["build"]` = `["build"]`.

> ⚠️ **An empty local list does NOT cancel an inherited one.** `depends = []` (likewise `run`, `run_windows`, `depends_post`, `wait_for`, `sources`) on the task still inherits the template's value. To genuinely clear it, extend a different template that has none. Exceptions: `outputs = []` *is* an explicit declaration (result-only caching), and `cache = { enabled = false }` explicitly disables inherited caching.

**`usage` now merges (changed 2026.9.11).** A task that `extends` a template *and* declares its own `usage` gets the template's flags **too**, listed first in `--help`. Previously the task's spec replaced the template's entirely, so shared flags had to be copied into every task.

```toml
[task_templates.deploy]
usage = 'flag "--env <env>" help="Target environment"'

[tasks."deploy:api"]
extends = "deploy"
usage = 'flag "--canary" help="Canary rollout"'   # --env AND --canary both available
run = "./deploy.sh"
```

> A flag declared in **both** places is listed **twice** — declare each flag in exactly one of the two. Workspace-root `[monorepo.task_defaults]` still only fills in `usage` when the task has none.

**File tasks can extend too (2026.9.11+):**
```bash
#!/usr/bin/env bash
#MISE extends="python:build"
```
For a **file task the template's `run` is ignored** (the script itself is the command), but `extends` still inherits tools, env, vars, args, and the rest. A template's `vars` can also read values the extending task supplies. (The template's `usage` flags only reach file tasks since **2026.10.19** — earlier versions silently dropped them.)

Templates are Tera-rendered in the **consuming** project's context (`{{config_root}}`, `{{env.VAR}}`, `{{cwd}}`, `{{vars.*}}`). Tasks loaded via `task_config.includes` now receive the collected `task_templates` map.

### Global Task Configuration

#### [task_config] Section

```toml
[task_config]
dir = "{{cwd}}"      # Default working directory for all tasks in this file
shell = "bash -c"    # Project-scoped default shell for tasks (2026.7.15+)
cascade = false      # Cascade dir/shell/cache/inputs/includes to descendant config roots
includes = [
  "tasks/*.toml",                              # Local task files
  ".mise/tasks/",                              # Task directory
  "git::https://github.com/org/tasks?ref=v1",  # Remote git tasks (experimental)
  "oci::ghcr.io/org/tasks:1.0.0",              # OCI task catalog (2026.9.18)
]
```

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `dir` | string | — | Default dir for all tasks in scope |
| `shell` | string | — | Project-scoped default shell; task-level `shell` wins |
| `cascade` | bool | `false` | Cascade `dir`, `shell`, `cache`, `global_inputs`, `global_env`, `global_pass_through_env`, `input_groups`, `includes`, `excludes`, and `rust_cache` to descendant config roots. A descendant's non-empty `global_inputs` replaces; `input_groups` merge by name (nearest wins); a descendant `cascade = false` stops inheriting. |
| `includes` | string[] | the five default task dirs | Local paths, `git::` (experimental), or `oci::` refs; Tera-rendered. **Last entry wins** on name collision |
| `excludes` | string[] | `[]` | Config-root-relative paths or globs excluded from **file-task discovery**. Task dirs are searched recursively and every non-mise `.toml` in them loads as a task file — exclude `pyproject.toml`/`Cargo.toml` copies this way. The closest config defining `excludes` replaces inherited ones; `excludes = []` clears them. |
| `cache` | table | — | **Experimental**; inherits only to tasks with sources+outputs |
| `rust_cache` | bool \| table | — | 🔴 **Deprecated NO-OP** (removal 2027.8.14). Does nothing. |
| `global_env` | string[] | — | **Experimental**; env names folded into every task's cache key |
| `global_pass_through_env` | string[] | — | **Experimental**; passed through without affecting cache keys |
| `global_inputs` | string[] | — | **Experimental**; config-rooted patterns applied to every task; may use `@group:` refs |
| `input_groups` | `{name: string[]}` | — | **Experimental**; referenced as `@group:name`, nestable |

> 🔴 **Breaking change (2026.7.14):** `unix_default_inline_shell_args`, `unix_default_file_shell_args`, `windows_default_inline_shell_args`, and `windows_default_file_shell_args` are now **global-config-only**. Project-local values are ignored, because local config is loaded before trust evaluation and an untrusted repo could otherwise influence how trusted commands execute. **Migration:** set `shell` per task, or use `task_config.shell` (added in 2026.7.15 precisely as the sanctioned replacement). Global config and `MISE_*` env vars still apply.

Setting `includes` **replaces** default search paths — list defaults explicitly to keep them:
```toml
includes = [
  "mise-tasks", ".mise-tasks", ".mise/tasks",
  ".config/mise/tasks", "mise/tasks",
  "mytasks", "tasks.toml",
]
```

Included task-file format (short form, not full mise.toml):
```toml
task1 = "echo task1"
task2 = "echo task2"

[task4]
run = "echo task4"
vars = { target = "linux" }
```

#### [vars] Section

Shared variables between tasks (NOT environment variables — never exported to processes):

```toml
[vars]
project_name = "myapp"
version = "1.0.0"
e2e_args = { default = "--headless" }                     # only if unset
api_token = { required = "Set api_token in mise.local.toml" }
secret_arg = { value = "--token=abc123", redact = true }
_.file = "vars.toml"    # load vars from an external file (dotenv/json/yaml/toml)

[tasks.build]
run = "echo Building {{vars.project_name}} v{{vars.version}}"

[tasks.test]
vars = { e2e_args = "--headed" }   # task-local override
run = './scripts/test-e2e.sh {{vars.e2e_args}}'
```

Value forms: plain scalar; `{ value, redact?, tools? }`; `{ default, redact?, tools?, prompt? }`; `{ required = true|"msg", redact?, tools?, prompt? }`; `{ age = ... }` (experimental). Plus the `_` module: `vars._.file` and `vars._.source` — strings, arrays, or (since 2026.10.9) **tables** like `_.file = { path = ".env.secret", redact = true }`.

Vars accessed via `{{vars.key_name}}` Tera templates. **Scope precedence:** global (`~/.config/mise/config.toml`) < project `mise.toml` < `mise.local.toml` < task-local `vars`. For a `{ default = … }` / `{ required = … }` var the full order is (highest first): a **process env var of the same name** (set and non-empty; `[env]` values are not consulted) → a plain value in any config → the machine's **saved prompt answer** → `default`. Top-level vars resolve once at load and are stored as strings, so a task-local override does **not** recompute vars derived from it (`args = "--mode={{vars.mode}}"` keeps the top-level value).

**Per-machine prompts (2026.10.6)** — values each developer supplies once, stored outside config (docs: `https://mise.jdx.dev/configuration/vars.html#prompt`):

```toml
[vars.git_name]
default = "Ada Lovelace"
prompt  = "Git author name"

[vars.git_email]
required = "Run mise vars prompt to set git_email"
prompt   = "Git email"
```

```bash
mise vars                       # list vars and where each value comes from ([redacted] masked)
mise vars prompt [NAME[=VALUE]]…  # ask for every unanswered prompt var (or just those named)
mise vars git_name=Ada          # save without asking;  `mise vars git_name` prints it
mise vars ls [--no-header]      # alias: list
mise vars --json                # 2026.10.22: {"name": "value"} (redacted stay [redacted]); --json-extended adds source; not with NAME
mise vars unset git_name        # forget a saved answer (aliases rm/remove; edits only the state file)
mise bootstrap --prompt-vars    # ask during a bootstrap run (e.g. with --adopt)
```

Answers live in `$MISE_STATE_DIR/vars.toml`, keyed by **name only** (shared by every project on the machine). Only `mise vars prompt` and `bootstrap --prompt-vars` ever ask — everything else uses the saved answer, then `default`, or fails for a `required` var. `prompt` is valid only in `[vars]`, never `[env]`. `[bootstrap.repos]` keys can be templated with these vars (2026.10.9).

> The `vars.mise` namespace is **rejected at parse time** (no deprecation period). Use `vars._`.

### Task-Scoped Secrets (Experimental)

Added **2026.10.4** (docs: `https://mise.jdx.dev/environments/secrets/fnox.html`). mise asks [fnox](https://github.com/jdx/fnox) (≥ 1.39.0, from the project's tools or PATH; a running fnox daemon is used only from fnox 1.43.0 — older daemons count as not running) for the named keys and gives them **only to the task that lists them** — and programs it starts.

```toml
min_version = "2026.10.4"   # older mise rejects `secrets` on a task

[settings]
experimental = true

[secrets.fnox]              # new top-level key; project config only
profile = "prod"            # optional — passes `-P prod` to fnox

[tasks.deploy]
depends = ["build"]                        # build receives NOTHING
secrets = ["DEPLOY_KEY", "DATABASE_URL"]   # string or array; names ^[A-Za-z_][A-Za-z0-9_]*$
run = "./deploy.sh"

[tasks.migrate]
env.PGURL = "postgres://app:{{ secrets.DB_PASSWORD }}@db/app"   # the reference IS the grant
run = "./migrate.sh"
```

```bash
mise run --secrets STRIPE_KEY deploy        # one-off grant (flags BEFORE the task name)
mise run --secrets-all deploy               # every fnox secret with env = true / "exec"
mise x --secrets GH_TOKEN -- gh release list
mise secrets ls [-J] [--no-header]          # KEY, ENV, FILE, SCOPES, TASKS, DESCRIPTION — never values
```

- **Not inherited:** dependencies, `depends_post` tasks, `run = [{ task }]` children, hooks, `skip_if`, `mise env`, the activated shell, shims, and daemons never see them. For `mise run --secrets`, only the tasks named on the command line get them.
- **`{{ secrets.NAME }}`** is allowed **only** in a task's own `env` (incl. a file task's `#MISE env=` header) — rejected in `run`, `[env]`, `[vars]`, and elsewhere. No filters/other variables; with `env_shell_expand` on, a `$` in the literal text is an error.
- **Restrictions:** tasks that list secrets always need **trust**; can't come from remote, global, or system config; are refused when launched from hooks, `watch_files`, daemons, or bootstrap; **skip the artifact cache**; always have **redacted output** (`--raw` ignored unless the task sets `raw`/`interactive`). `secrets` is not allowed in `[task_templates]` or `[monorepo.task_defaults]`. **Personal keys are refused (2026.10.23):** a key from fnox's global config (`~/.config/fnox/config.toml`, where `fnox set --global` writes) or a `fnox.toml` in/above `$HOME` — or a project secret whose value interpolates one — belongs to you, so naming it in a task's `secrets` or `{{ secrets.NAME }}` is an error (`X comes from your own fnox config`); `--secrets-all` omits these keys, while an explicit `mise run --secrets NAME` / `mise x --secrets NAME` still passes them. `mise secrets ls` and `mise tasks validate` report them as kind `personal` (needs an fnox that reports key origin). A key mise also sets for the task (via `[env]`, task `env`, a tool, or a setting) is an error; a key the task's sandbox would drop is an error (add it to `allow_env`). File tasks use `#MISE secrets=[…]`. Refused in safe mode.
- `[secrets.fnox]` merges per field (nearest project file wins, so `mise.<env>.toml` can switch `profile`); it is **ignored** in global/system config and files in or above `$HOME` (`mise doctor` reports it). In CI or without a TTY mise runs `fnox --non-interactive --no-daemon`; it reads a running fnox daemon's cache but never starts one. `mise x` can't grant file (`as_file`) secrets and its output is never redacted.
- **Migration:** the `mise-env-fnox` plugin and `_.fnox-env` directive are deprecated (`mise doctor` flags them) — move to `[secrets.fnox]` + per-task `secrets`. Check config with `mise tasks validate`.
- 🔴 **`mise secrets` is now a built-in command** — a task named `secrets` must be run as `mise run secrets`. Likewise `installs` (10.4), `vars` (10.6), and `lsp` (10.18).
- Variables a parent mise marks in `__MISE_SECRET_KEYS` are redacted and hidden from templates, `get_env()`, and `exec()` in a nested mise.

#### Global Task Settings

| Setting | Type | Default | Env Var | Description |
|---------|------|---------|---------|-------------|
| `task.output` | string | unset (→ `prefix` if jobs>1, else `interleave`) | `MISE_TASK_OUTPUT` | prefix, interleave, keep-order, replacing, timed, quiet (deprecated), silent |
| `task.quiet` | bool | `false` | `MISE_TASK_QUIET` | Suppress mise's own task messages/prefix headers without hiding task output (replaces `output = "quiet"`) |
| `raw` | bool | unset | `MISE_RAW` | Global `--raw` (forces jobs=1, bypasses redactions) |
| `task.timeout` | duration | unset | `MISE_TASK_TIMEOUT` | Time limit for a **whole `mise run` invocation**; when a per-task (per-command) timeout also applies, the shorter wins |
| `task.timings` | bool | unset | `MISE_TASK_TIMINGS` | Show elapsed time per task (shown by default with `prefix` output) |
| `task.skip` | string[] | `[]` | `MISE_TASK_SKIP` | Tasks to skip by default |
| `task.skip_depends` | bool | unset | `MISE_TASK_SKIP_DEPENDS` | Skip dependencies |
| `task.run_auto_install` | bool | `true` | `MISE_TASK_RUN_AUTO_INSTALL` | Auto-install missing tools |
| `task.show_full_cmd` | bool | unset | `MISE_TASK_SHOW_FULL_CMD` | Disable command truncation in output |
| `task.disable_paths` | string[] | `[]` | `MISE_TASK_DISABLE_PATHS` | Paths to exclude from task discovery |
| `task.remote_no_cache` | bool | unset | `MISE_TASK_REMOTE_NO_CACHE` | Always fetch latest remote tasks |
| `task.source_freshness_hash_contents` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_HASH_CONTENTS` | Use blake3 content hashing instead of mtime |
| `task.source_freshness_equal_mtime_is_fresh` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_EQUAL_MTIME_IS_FRESH` | Equal mtime = fresh |
| `task.disable_spec_from_run_scripts` | bool | `false` | `MISE_TASK_DISABLE_SPEC_FROM_RUN_SCRIPTS` | Derive the usage spec only from the `usage` field (early opt-out before Tera arg functions are removed) |
| `task.auto_infer` | string[] | `[]` | `MISE_TASK_AUTO_INFER` | Ecosystems whose scripts become tasks (e.g. `["node"]`). **Not experimental since 2026.10.6** — needs only `monorepo_root = true` |
| `task.cache_dir` | path | `$MISE_CACHE_DIR/task-artifacts` | `MISE_TASK_CACHE_DIR` | **Experimental.** Task artifact cache location |
| `task.cache_max_size` | string | unset | `MISE_TASK_CACHE_MAX_SIZE` | **Experimental.** SI/IEC units (`500MB`, `2GiB`); LRU eviction |
| `task.cache_max_age` | duration | unset | `MISE_TASK_CACHE_MAX_AGE` | **Experimental.** `0s`/unset disables |
| `task.cache.remote_*` | — | — | — | **Experimental.** See [Remote Task Cache](task-caching.md#remote-task-cache). Flat `task.cache_remote_*` is deprecated. |
| `task.cache.audit_report` | path | unset | `MISE_TASK_CACHE_AUDIT_REPORT` | **Experimental.** JSON Lines audit of undeclared reads/writes |
| `task.cache.stats_report` | path | unset | `MISE_TASK_CACHE_STATS_REPORT` | 🔴 **Deprecated no-op** (hidden; warns since 2026.10.5, removed 2027.10.5) — the built-in action cache it reported on was removed. `--task-cache-stats` still works |
| `task.monorepo_depth` | int | `5` | `MISE_TASK_MONOREPO_DEPTH` | Subdirectory depth for monorepo discovery |
| `task.monorepo_exclude_dirs` | string[] | `[]` | `MISE_TASK_MONOREPO_EXCLUDE_DIRS` | Empty ⇒ built-ins (`node_modules`, `target`, `dist`, `build`); any custom value **replaces** them |
| `task.monorepo_respect_gitignore` | bool | `true` | `MISE_TASK_MONOREPO_RESPECT_GITIGNORE` | Honor `.gitignore` in monorepo discovery |
| `jobs` | int | `8` | `MISE_JOBS` | Max concurrent task execution |
| `otel.enabled` | bool | `false` | `MISE_OTEL_ENABLED` | **Experimental (2026.9.13).** Export `mise run` traces — see [OpenTelemetry](task-tracing.md#task-tracing-with-opentelemetry-experimental) |
| `otel.logs` | bool | `false` | `MISE_OTEL_LOGS` | **Experimental.** Also export task stdout/stderr as OTLP log records |

> **Deprecated flat aliases** (`task_output`, `task_timeout`, `task_timings`, `task_skip`, `task_skip_depends`, `task_disable_paths`, `task_remote_no_cache`, `task_run_auto_install`, `task_show_full_cmd`) **began warning in 2026.8.0** and are **removed in 2027.2.0**. Use the dotted `task.*` forms.
>
> **`jobs` default is 8.** The old 8-vs-4 discrepancy is gone: on 2026.8.12 both `mise settings get jobs` and the global `-j/--jobs` help report **8**, and `mise run --help` no longer documents a separate default. Values below 1 are treated as 1.
