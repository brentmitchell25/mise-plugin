## Running Tasks (CLI)

**Contents**

- [Commands](#commands)
- [Execution Flags](#execution-flags)
- [Output Modes](#output-modes)
- [mise watch Flags](#mise-watch-flags)
- [Parallel Tasks and Wildcards](#parallel-tasks-and-wildcards)
- [Default Task](#default-task)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### Commands

| Command | Description |
|---------|-------------|
| `mise run <task>` / `mise r <task>` | Execute task |
| `mise <task>` | Shorthand (discouraged in scripts — future mise versions may add conflicting commands) |
| `mise run` (no args) | Runs the `default` task; with no `default`, opens the interactive task selector (TTY only) |
| `mise tasks` / `mise tasks ls` / `mise tasks list` | List all tasks (`-J/--json`, `-x/--extended`, `--no-header`); `list` alias since 2026.10.18. `mise tasks run` still works but is hidden from help (2026.10.5) |
| `mise tasks --hidden` | Include hidden tasks |
| `mise tasks --global` / `--no-global` | Filter by config scope (`--no-global` 2026.10.22; `-l/--local` is a deprecated alias — warns from 2027.1.0, removed 2027.7.0) |
| `mise tasks -a` / `--all` | Entire monorepo, including siblings (`-a` since 2026.10.15) |
| `mise tasks --name-only` | One name per line (fzf-friendly) |
| `mise tasks --sort <name\|alias\|description\|source>` | Sort order (`--sort-order asc\|desc`) |
| `mise tasks deps [--hidden]` | Show dependency tree (`depends_post` entries shown as `<name> (post)`) |
| `mise tasks deps --compact` | Expand each shared subtree once, mark repeats `(already shown)` (2026.7.18+) |
| `mise tasks deps --dot` | DOT format for Graphviz |
| `mise tasks graph [--explain] [--json]` | Inspect the workspace project graph (Cargo/uv/Go/Gradle/Node) — **not experimental since 2026.10.6**; needs `monorepo_root = true` |
| `mise tasks info <task> [-J]` | Show task details (JSON output includes `config_sources`) |
| `mise tasks add <name> -- <cmd>` | Create task via CLI: `-a/--alias`, `-d/--depends`, `--depends-post`, `-w/--wait-for`, `-D/--dir`, `-f/--file` (file task), `-H/--hide`, `-q/--quiet`, `--silent`, `-r/--raw`, `-s/--sources`, `--outputs`, `--description`, `--run-windows` (ignored with `--file`), `--shell`; since **2026.10.25** also `--env-var KEY=VALUE` and `--tools TOOL@VER` (repeatable; no version ⇒ `latest`), `--usage <SPEC>` (with `--file`, written as one `#USAGE` line per spec line), `--confirm <MSG>`, `-F/--force`, `-n/--dry-run`, `-J/--json`, and target flags `-e/--env <ENV>` (`--env local` → `mise.local.toml`; `-e ci` writes `mise.ci.toml`, loaded only with `MISE_ENV=ci`), `-g/--global`, `-p/--path` (`--file` can't combine with them). Args after `--` are shell-quoted into one `run` string, so pass the command as separate words — `-- 'echo hi'` is stored as the single quoted word `'echo hi'` and fails with `command not found`. Re-adding an identical task is a no-op; replacing a **different** task without `--force` is deprecated (warns from 2027.5.0, refused from 2028.5.0). Trusts the written file's config root (not in safe mode) |
| `mise tasks set <task> [flags] [-- <cmd>]` | **2026.10.25.** Change only the given fields of a TOML task, keeping the rest of the file (comments too). Takes the `tasks add` field flags; list flags (`--depends`, `--sources`, …) replace the whole list; `--env-var`/`--tools` add or replace single entries; `-u/--unset <FIELD>` (repeatable) removes a field; a command after `--` replaces `run`. Without `-e`/`-g`/`-p` it edits the one loaded file that defines the task and fails when several do. File tasks are refused (use `mise tasks edit`). `-n/--dry-run`, `-J/--json` |
| `mise tasks rm <task>…` / `mise tasks remove` | **2026.10.25.** Remove a TOML task's table or delete a file task's script. Asks first — pass `--yes` without a terminal (a piped `y` isn't accepted). Same `-e`/`-g`/`-p`, `-n`, `-J` flags (`--json` prints an array). Refuses tasks from `task_config.includes` files or package.json; dependents are left as is (`mise tasks validate` reports them). A task named `set`, `rm`, or `remove` is now reachable only through `mise run <name>` |
| `mise tasks edit <task> [-p]` | Edit/create task in `$EDITOR` (respects configured `includes`; `-p` prints the path) |
| `mise tasks validate [--errors-only] [--json]` | Validate task definitions. Since **2026.9.15** an unparseable `usage` spec is an **error** (`usage-parse-error`, exit 1) — CI that ran it may start failing. Since 2026.10.4 it also errors on `secrets` with no `[secrets.fnox]`, secrets conflicting with the sandbox or `[env]`, and unknown `daemons` |
| `mise watch <task>` / `mise w <task>` | Watch and re-run on file changes (requires watchexec); `--once` waits for one change, runs once, and exits with the tasks' status (2026.10.25) |
| `mise run a ::: b ::: c` | Run multiple tasks in parallel (with separate args) |

### Execution Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--jobs <N>` | `-j` | Parallel job limit (default 8; values below 1 treated as 1) |
| `--force` | `-f` | Ignore source/output freshness |
| `--dry-run` | `-n` | Preview without executing |
| `--output MODE` | `-o` | prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `--raw` | `-r` | Direct stdin/stdout/stderr (forces `--jobs=1`, bypasses redactions) |
| `--quiet` | `-q` | Suppress mise's extra output |
| `--silent` | `-S` | Hide all output except errors |
| `--continue-on-error` | `-c` | Continue running tasks even if one fails |
| `--cd <DIR>` | `-C` | Change working directory before execution |
| `--shell SHELL` | `-s` | Shell spec for TOML tasks |
| `--tool TOOL@VERSION` | `-t` | Additional tools beyond mise.toml |
| `--no-timings` | — | Hide per-task elapsed time; overrides `MISE_TASK_TIMINGS=1` (`--timings` still works but is hidden from help) |
| `--timeout DURATION` | — | Task timeout (e.g., `30s`, `5m`) |
| `--fresh-env` | — | Bypass environment cache |
| `--skip-deps` | — | Run only specified tasks, skip dependencies |
| `--no-deps` | — | Skip automatic dependency preparation |
| `--skip-tools` | — | Skip installing tools before running tasks |
| `--no-cache` | — | Skip cache on remote tasks |

**Task output cache flags** (experimental): `--task-cache <read-write|read-only|write-only|off|local-only>` (default `read-write`, env `MISE_TASK_CACHE`), `--task-cache-explain`, `--task-cache-explain-json` (**requires `--dry-run`**), `--task-cache-stats` (conflicts with `--dry-run`).

**Affected-set flags** (**not experimental since 2026.10.6**): `--affected`, `--affected-base <REV>`, `--affected-head <REV>`, `--affected-explain`, `--affected-json`, plus source selectors `--affected-committed` / `--affected-uncommitted` / `--affected-untracked` (env `MISE_AFFECTED_COMMITTED`/`_UNCOMMITTED`/`_UNTRACKED`, 2026.10.6) — see [Affected Tasks](monorepo.md#affected-tasks).

**Secrets flags** (experimental, 2026.10.4; must precede the task name; refused in safe mode; also on `mise x`): `--secrets <KEY>[,<KEY>]` grants fnox keys for this invocation; `--secrets-all` grants every fnox secret with `env = true`/`"exec"` (never `env = false`).

> **stdin** reaches a task only with `interleave` output (a single task, a sequential chain, or `-j1`) — not under `prefix`, and not when output must be redacted (`secrets`, `redact = true`, `redactions`). `--continue-on-error` still starts dependents of the failed task and lists all failures at the end. With no `default` task in a non-interactive shell, `mise run` fails with `no task default found`; `alias = "default"` on any task makes it the default.

**Sandbox / permission flags** (stable since v2026.6.6; also available on `mise exec`):

| Flag | Description |
|------|-------------|
| `--allow-env <VAR>` | Allow specific env var through (supports wildcards; implies deny for everything else) |
| `--allow-net <HOST>` | Allow network access to specific host |
| `--allow-read <PATH>` | Allow filesystem reads from specific path |
| `--allow-write <PATH>` | Allow filesystem writes to specific path |
| `--deny-all` | Block reads, writes, network, and env vars |
| `--deny-env` | Block env var inheritance (keeps PATH/HOME/USER/SHELL/TERM/LANG/COLORTERM) |
| `--deny-net` | Block all network access |
| `--deny-read` | Block filesystem reads |
| `--deny-write` | Block filesystem writes (except implicitly writable system paths such as the temp dir) |

> `--allow-net <HOST>` per-host filtering is unsupported on **Linux and macOS** (both error; macOS since 2026.10.5); on Windows sandboxing is unavailable (warns and runs unfiltered).

> **mise's own flags must precede the task name:** `mise run --silent build`, not `mise run build --silent`. Extra args after the task go to the **last** command.

> **Breaking change (2026.7.6):** `--quiet` / `quiet = true` / `MISE_QUIET=1` **no longer collapse task output** to un-prefixed interleave — they preserve the resolved style. Use `--output quiet` or `-o interleave` for the old behavior.

> **Ctrl-C is an interruption, not a failure.** Since **2026.10.1** a single Ctrl-C stops new work, **waits for running tasks to clean up** (no duplicate SIGINT, no SIGTERM to siblings), then exits **130**; a second Ctrl-C force-quits. Since 2026.10.0 every mise command (`install`, `exec`, …) exits 130 on Ctrl-C, not 1.

> **Timeouts really stop tasks (2026.10.1+).** The whole-run `--timeout` / `task.timeout` sends SIGTERM then SIGKILL after 5s (Windows: immediate `taskkill /F /T`); it does not stop `raw = true` tasks. A timed-out task is reported failed even if it exits 0.

### Output Modes

- `prefix` — Default when `jobs > 1`; each line prefixed with task name
- `interleave` — Default when `jobs == 1`; print as output arrives
- `keep-order` — Stream one task's output live; buffer others; print in definition order
- `replacing` — Replace stdout on each new line (similar to `mise install`)
- `timed` — Show only stdout lines that took >1s
- `quiet` — Print only task stdout/stderr, nothing from mise itself. **Deprecated (warns since 2026.9.3), removed 2027.9.3** — use `output = "interleave"` plus `task.quiet = true` (`MISE_TASK_QUIET`) or a per-task `quiet = true`, or `--output interleave --quiet`.
- `silent` — Print nothing from tasks or mise

Set via `--output`, `task.output`, `MISE_TASK_OUTPUT`, or the per-task `output` field. Style is **orthogonal** to verbosity — `MISE_TASK_OUTPUT=prefix --quiet` keeps prefixes while silencing mise's own messages.

### `mise watch` Flags

Wrapper around `watchexec` (install separately: `mise use -g watchexec@latest`). By default watches the `sources` of the task **and its dependencies**; a task with no sources watches the cwd; `mise watch` with no task runs `default`.

> 🔴 **2026.10.5:** watchexec flags were previously parsed but **silently dropped** — they are now forwarded (`--shell`, `--notify`, `--color`, `--timings`, `--bell`, `--fs-events`, `--filter-file`, `--emit-events-to`, `--manual`, `--only-emit-events`, `--project-origin`, `--map-signal`, `--stdin-quit`, `--no-meta`, `--ignore-file`, `--no-{project,global,default,discover}-ignore`, `--ignore-nothing`). watchexec's own `-E/--env` and `-q/--quiet` are renamed **`--watchexec-env KEY=VALUE`** and **`--watchexec-quiet`** (so `-E`/`-q` stay mise's `--env`/`--quiet`). `--workdir` is hidden and has no effect.

| Flag | Short | Description |
|------|-------|-------------|
| `--glob <GLOB>` | `-g` | **2026.10.5.** Watch these globs instead of `sources` |
| `--watch <PATH>` | `-w` | Watch path recursively (repeatable; `/dev/null` disables path watching) |
| `--watch-non-recursive <PATH>` | `-W` | Non-recursive watch |
| `--watch-file <PATH>` | `-F` | File listing paths to watch (`-` = stdin) |
| `--exts <EXTS>` | `-e` | Comma-separated extension filter |
| `--filter <PATTERN>` | `-f` | Include glob |
| `--filter-prog <EXPR>` | `-J` | **Experimental** jaq-based filter |
| `--ignore <PATTERN>` | `-i` | Exclude glob |
| `--no-vcs-ignore` | — | Include `.gitignore`d files |
| `--poll <INTERVAL>` | — | Polling instead of native fs events (default `30s`; also `--force-poll`) |
| `--clear <MODE>` | `-c` | `clear` or `reset` between runs |
| `--restart` | `-r` | Shorthand for `--on-busy-update=restart` |
| `--on-busy-update <MODE>` | `-o` | `queue`, `do-nothing` (default), `restart`, `signal` |
| `--signal <SIGNAL>` | `-s` | Signal to running process |
| `--stop-signal <SIGNAL>` | — | Default SIGTERM (Unix) |
| `--stop-timeout <DUR>` | — | Grace period before kill (default `10s`) |
| `--debounce <DUR>` | `-d` | Debounce window (default `50ms`) |
| `--postpone` | `-p` | Wait for first change before initial run |
| `--once` | — | **2026.10.25.** Wait for a change in the watched sources, run the tasks once through `mise run`, and exit with their status (implies `--postpone`; exit 1 if watchexec stops before a change, 130 on Ctrl-C). Refused with watchexec run-control flags (`--restart`, `--clear`, `--on-busy-update`, `--signal`, `--stop-signal`, `--stop-timeout`, `--delay-run`, `--shell`, `--notify`, `--bell`, `--timings`, `--wrap-process`, …). Use it in scripts and agents instead of a sleep/poll loop: `mise watch test --once` |
| `--exit-on-error` | — | **2026.10.25.** Stop watching at the first failed run and exit with its status (needs watchexec ≥ 2.5.0; no effect with `--once`) |
| `--delay-run <DUR>` | — | Sleep before each execution |
| `--notify` | `-N` | Desktop notifications |
| `--bell` | — | Terminal bell on completion |
| `--print-events` | — | Human-readable event output |
| `--fs-events <KINDS>` | — | Default `create,remove,rename,modify,metadata` |
| `--wrap-process <MODE>` | — | `group` (default), `session`, `none` — honored on macOS since 2026.7.18 |
| `--skip-deps` | — | Run only specified tasks; skip deps |

### Parallel Tasks and Wildcards

```bash
mise run test:*         # All test:* tasks
mise run lint:**        # All nested lint tasks
mise run {build,test}   # Multiple specific tasks
mise run lint ::: test ::: check  # Parallel task groups with :::
mise run cmd1 arg1 ::: cmd2 arg2  # Parallel with separate args
```

Glob patterns accepted in task selectors:

| Pattern | Matches |
|---------|---------|
| `?` | Single character |
| `*` | Zero or more characters (within one namespace segment) |
| `**` | Zero or more nested namespace segments |
| `{a,b,c}` | Comma-separated alternatives |
| `[abc]` | Character set/range |
| `[!abc]` | Negated character set |

`mise run 'test:*:local'` matches only `test:units:local`; `mise run 'test:**:local'` also matches `test:e2e:happy:local`.

### Default Task

```toml
[tasks.default]
depends = ["build", "test"]
run = "echo 'Ready!'"
```
