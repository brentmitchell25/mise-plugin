## Hooks and Watchers

**Contents**

- [Hook Types](#hook-types)
- [Syntax](#syntax)
- [Watch Files](#watch-files)
- [Hook Environment Variables](#hook-environment-variables)
- [Per-Tool postinstall (not a hook)](#per-tool-postinstall-not-a-hook)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### Hook Types

| Hook | Trigger | Requires `mise activate`? |
|------|---------|--------------------------|
| `cd` | Every directory change that **ends inside** the project (including the one that enters it; not when leaving) | Yes |
| `enter` | Enter a project (once per entry; not re-fired for subdirs). Also fires when a project config is first loaded without a `cd` (e.g. after creating/trusting `mise.toml`) and at the first prompt after activation | Yes |
| `leave` | Leave a project (once) | Yes |
| `preinstall` | Before tool installation | No |
| `postinstall` | After tool installation (**fires even on no-op installs**) | No |

> **Hooks do not override across config files — they all run.** When the same hook type is defined in several loaded configs, mise runs **every** one, from highest-precedence config to lowest. Within one file, array-form hooks run in listed order. So hooks in `conf.d/a.toml`, `b.toml`, `c.toml` execute as **c, b, a** (later alphabetical fragments have higher precedence).

> `enter` does not re-fire when you `cd` **within** the same project, and `leave` does not fire on internal `cd` either. Within one directory change the order is **`leave` → `cd` → `enter`**. Hooks in **global** config have no project, so global `cd`/`enter`/`leave` fire on every directory change, in the new directory.

**Working directory (2026.10.19, breaking):** spawned `enter`/`leave` (and `watch_files`) commands run in the **project root**, even when you enter or leave through a subdirectory; `cd` hooks run in the directory the shell moved to; current-shell `script` hooks run in the shell's cwd — refer to project files via `$MISE_PROJECT_ROOT`. Hook commands are Tera templates rendered **once, at load** (`enter = "echo {{config_root}}"`); their output goes to stderr; a failing hook warns and the remaining hooks still run.

**Disable hooks** with the root-level `--no-hooks` flag (`mise --no-hooks install`, not `mise install --no-hooks`), `MISE_NO_HOOKS=1`, `no_hooks = true`, or `MISE_SAFE=1` (none affect a tool's own `postinstall` option). Spawned hook commands, task hooks, and watch-file tasks run with `MISE_NO_HOOKS=1` so a nested `mise` doesn't re-run hooks.

### Syntax

```toml
[hooks]
cd = "echo 'changed directory'"
enter = "echo 'entered project'"
leave = "echo 'left project'"
preinstall = "echo 'about to install'"
postinstall = "echo 'installed'"

# Inline object form ({ run = "..." } is equivalent to the string shorthand)
enter = { run = "echo 'entered project'" }
enter = { run = "echo unix", run_windows = "echo win" }  # OS-specific variant
postinstall = { run = "echo installed", shell = "bash -c" }  # explicit inline shell

# Multiple hooks (each `run` spawns its own subprocess)
enter = ["echo 'first'", { run = "echo 'second'" }]

# Multiline run = ONE subprocess
[hooks.enter]
run = """
echo one
echo two
"""

# Shell hooks (execute IN the current shell; shell = selector bash|zsh|fish)
[hooks.enter]
shell = "bash"
script = "source completions.sh"   # `scripts = [...]` for multiple

# Task hooks — executed via `mise run`, so the full task system applies.
# Works with ALL hook types (cd, enter, leave, preinstall, postinstall).
[hooks]
enter = { task = "setup" }
enter = ["echo 'entering'", { task = "setup" }]  # Mixed syntax

# Array-of-tables form
[[hooks.cd]]
run = "echo 'I changed directories'"
[[hooks.cd]]
run = "echo 'I also changed directories'"
```

`run`/`run_windows` must be **strings** — arrays are unsupported. On Windows mise uses `run_windows` when set; on other platforms a hook with only `run_windows` is skipped.

> ⚠️ **A task used as a `preinstall` hook does NOT auto-install missing tools** — that would defeat the point of running ahead of the install it is preparing. Its commands must already be on PATH. Every other task-backed hook type keeps normal task tool-installation behavior.

> **Task hooks and trust:** a task hook does **not** trust the config for you — an untrusted config, or an untrusted task it runs or depends on, makes the hook fail with a warning (and since 2026.10.19 a hook's `mise run` records no implicit trust). A task hook in **global** config runs in the active project only after that project's config and task dirs are trusted; until then it runs in the global config root. A task that lists `secrets` never runs from a hook. Since 2026.10.13 task hooks run through `mise` even when a tool **shim** triggered them.

`script` takes a string or an array of lines; `scripts` takes only an array.

`shell` means different things depending on context: on a `run` hook it's an **inline shell command** (include the eval arg: `"bash -c"`, `"pwsh -Command"`); on a `script`/`scripts` hook it's a **shell-name selector** (`bash`, `zsh`, `fish`) and mise only emits the script when the active `mise activate` shell matches.

**Important:** Shell hooks don't auto-cleanup on directory exit like `[env]` does. mise executes literal shell code without tracking it, so exported vars, aliases, and sourced files persist — reverse them manually in a corresponding `leave` hook.

> **Deprecation:** the *spawned* `script`/`scripts` table form is deprecated in favor of `run` — warns **2026.9.0**, removed **2027.3.0**. Current-shell `shell` + `script`/`scripts` for `cd`/`enter`/`leave` is unaffected. For `preinstall`/`postinstall`, `script`/`scripts` are legacy aliases for `run`, and a `shell` set alongside them is **ignored with a warning**.
>
> Since 2026.7.8, non-string `postinstall` hooks and unknown table fields in hook definitions are **rejected at parse time**.

### Watch Files

```toml
[[watch_files]]
patterns = ["src/**/*.rs"]
run = "cargo fmt"

[[watch_files]]
patterns = ["*.js"]
run = "eslint --fix ."
shell = "bash -c"

# Task reference
[[watch_files]]
patterns = ["uv.lock"]
task = "sync-deps"
```

Fields: `patterns` (required glob array, relative to the project root — `*` doesn't cross `/`, use `**`), `run`, `task`, `shell` (applies to `run` only). Set **one** of `run`/`task` — if both are set mise **warns and runs the task**. The command runs in the project root. Sets `MISE_WATCH_FILES_MODIFIED` (colon-separated, literal colons backslash-escaped).

**Requires `mise activate` only** — changes are checked at each activated-shell **prompt**, not by a background watcher (no watchexec needed). Entries in **global** config are ignored. Since 2026.10.5 watch files are skipped in safe mode and with `--no-hooks`, and a change made while hooks were off does not fire later.

**`MISE_WATCH_FILES_GIT_REWRITTEN=1`** (2026.10.12) is set when git rewrote **every** changed file (checkout, pull, rebase) — not for commits, HEAD moves that keep the same commit (10.14), `stash pop`, or `reset --hard` without a commit. Use it to skip formatters on branch switches:
```toml
[[watch_files]]
patterns = ["**/*.go"]
run = '[ -n "$MISE_WATCH_FILES_GIT_REWRITTEN" ] || { IFS=:; hk fix $MISE_WATCH_FILES_MODIFIED; }'
```

### Hook Environment Variables

All hooks (and watch files) receive:
- `MISE_ORIGINAL_CWD` — user's working directory at hook fire
- `MISE_PROJECT_ROOT` — detected project root

`[hooks]` entries additionally receive:
- `MISE_CONFIG_ROOT` — root of the **config file that defines the hook** (distinct from the project root)

> For a hook in **global** config, `MISE_CONFIG_ROOT` is the global config root and `MISE_PROJECT_ROOT` is the **active project's** root (the global root only outside any project). Only global-only operations such as `mise use --global` use the global root for both and skip project hooks.

CD/enter/leave hooks additionally receive:
- `MISE_PREVIOUS_DIR` — previous directory (only when a directory change occurred)

Config-level `postinstall` receives:
- `MISE_INSTALLED_TOOLS` — JSON array of `{name, version, requested_version, backend, install_path}` (the last two since **2026.9.13**), e.g.
  `[{"name":"node","version":"20.10.0","requested_version":"lts","backend":"core:node","install_path":"/home/u/.local/share/mise/installs/node/20.10.0"}]`.
  - `requested_version` is the **canonical pre-resolution selector** (`latest`, `20`, `lts`, `ref:main`), so a hook can branch without re-reading config.
  - `backend` is the canonical backend id **with options and URL credentials stripped** (they may carry registry secrets).
  - `install_path` is the concrete version dir — never a floating link like `latest` or `20`.
  - After a **partial failure** the array still lists the tools that did install.
  - It means "installation completed", not "config now selects it" — tool-level hooks can run before floating links are refreshed. **A postinstall hook is not an application-activation notification.**

> A `mise install` that finds nothing to install **still runs `postinstall`**, with `MISE_INSTALLED_TOOLS` set to `[]`. Guard accordingly.

> `preinstall`/`postinstall` run with the **project root as CWD** even when `mise install` was invoked from a subdirectory — the invocation dir is still in `MISE_ORIGINAL_CWD`. A project's install hooks run only when the cwd is inside that project. `preinstall` does not see `[env]` entries that use `tools = true`. `mise install --dry-run` prints install hooks without running them. Since **2026.10.18** `[hooks].postinstall` runs against the **just-installed** version. Don't use `postinstall` to restart services after `mise upgrade` — it can run before the upgrade finishes.

Tool-level `postinstall` additionally receives:
- `MISE_TOOL_NAME` — tool identifier (e.g., `node`)
- `MISE_TOOL_VERSION` — installed version
- `MISE_TOOL_INSTALL_PATH` — installation directory
- `MISE_CONFIG_FILE` — the exact config file that declared the tool
- `MISE_CONFIG_ROOT` / `MISE_PROJECT_ROOT`
- plus that tool's `install_env` values

### Per-Tool postinstall (not a hook)

Runs immediately after each tool is installed, before other tools in the same session:

```toml
[tools]
node = { version = "22", postinstall = "corepack enable" }
python = { version = "3.12", postinstall = "pip install pipx" }

# 2026.9.17: table form — run on EVERY `mise install` that selects the tool, even if already installed
go = { version = "1.24", postinstall = { run = "go install golang.org/x/tools/gopls@latest", when = "always" } }
```

`when = "install"` (default, also the string form) runs only on a fresh install or repair; `when = "always"` re-runs on every explicit install (once per tool request; skipped on dry runs).

> **Hooks from a remote [`include`](settings.md#remote-config-includes-include) fragment** run **before** the including file's hooks of the same type.
