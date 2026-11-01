## Task Definition Methods

**Contents**

- [TOML-Based Tasks (in mise.toml)](#toml-based-tasks-in-misetoml)
- [File-Based Tasks (executable scripts)](#file-based-tasks-executable-scripts)
- [Configuring File Tasks from TOML](#configuring-file-tasks-from-toml)
- [Task Grouping (Namespaces)](#task-grouping-namespaces)
- [Remote Tasks](#remote-tasks)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### TOML-Based Tasks (in mise.toml)

#### Simple Tasks

```toml
[tasks]
build = "cargo build"
test = "cargo test"
lint = "cargo clippy"
```

#### Detailed Tasks

```toml
[tasks.build]
description = "Build the CLI"
run = "cargo build"

[tasks.test]
description = "Run tests"
depends = ["build"]
run = "cargo test"
```

#### Multiline Scripts

```toml
[tasks.build]
run = '''
#!/usr/bin/env bash
set -euo pipefail
cargo clippy
cargo build --release
'''
```

#### Shebang Support in TOML

Execute scripts in multiple languages via shebang:

```toml
[tasks.script]
run = '''
#!/usr/bin/env python
for i in range(10):
    print(i)
'''
```

Supports: Python, Node, Bun, Deno, Ruby, Bash, PowerShell. Use `-S` for multiple interpreter arguments (`#!/usr/bin/env -S deno run --allow-env`).

> PowerShell tasks run with `-NoProfile` by default since **2026.7.13** (`windows_powershell_no_profile = true`), so a profile that mutates PATH can no longer shadow the task's own tools. Opt out with `MISE_WINDOWS_POWERSHELL_NO_PROFILE=false`.

### File-Based Tasks (executable scripts)

Place in task directories. Files **must** be executable (`chmod +x`).

```bash
#!/usr/bin/env bash
#MISE description="Build the CLI"
#MISE depends=["lint"]
cargo build
```

Supported directories (searched by default, in this order — a **later directory wins** on a name collision):
- `mise-tasks/`
- `.mise-tasks/`
- `.mise/tasks/`
- `.config/mise/tasks/`
- `mise/tasks/`

If `task_config.includes` is set, it **replaces** these defaults — list them explicitly to keep them. On name collision across includes, the **last entry wins**.

**Discovery rules (task-discovery page, 2026.10.x):** task dirs are searched recursively (following symlinks); files and dirs starting with `.` are skipped; discovery **errors after 100,000 entries** in one task dir (2026.10.19); `task_config.excludes` also filters paths selected by `includes`. A nearer directory beats a parent and a project task beats a global one; a user-global task **replaces** a system task entirely (no metadata inheritance).

**Global file tasks (2026.10.5):** `$MISE_CONFIG_DIR/tasks` (default `~/.config/mise/tasks`) is global whether or not a global config file exists, and honors a custom `MISE_CONFIG_DIR`.

**Task names from file names (2026.10.5):** extension stripping applies **only to file tasks** and only for known script extensions (`bash bat cjs cmd com exe fish js ksh lua mjs mts nu php pl ps1 py rb sh ts vbs zsh` plus `windows_executable_extensions`). So `mise-tasks/gen.proto` stays `gen.proto`, TOML tasks `[tasks."test.unit"]`/`[tasks."test.e2e"]` keep their full names, `MISE_TASK_NAME` is the listed name (`build`, not `build.sh`), and both `db:migrate` and `db:migrate.sh` run the task.

**`#MISE` headers accept every task property except `run`, `run_windows`, `file`, and `usage`** (2026.10.14 — verified on 2026.10.25: `#MISE timeout="1s"` kills the script, `#MISE vars=…`, `skip_if`, `confirm`, and `allow_env` all work). Those four warn ``mise WARN `run` has no effect in a task file header, ignoring``. A value of the wrong type warns and skips **only that key**; a bad `confirm` or `deny_*` value is a **hard error** (fails closed) — and it breaks the whole project, not just that task: every `mise run`, `mise tasks ls`, and `mise tasks info` there fails until it is fixed. A header `silent` counts as explicitly set, so an `extends` template no longer overrides it. Prefix a line with `# NOMISE` to disable it; `# MISE` (with a space) also works.
**Supported `#USAGE` directives:** `arg`, `flag`, `choices`, `complete`, `env`, and root-level `mount`.

> Before **2026.10.14**, `timeout`, `vars`, and the `deny_*`/`allow_*` sandbox fields were rejected in headers (`WARN unknown field(s) … ignoring`). If a project must support older mise, keep them in a metadata-only TOML block (see [Configuring File Tasks from TOML](#configuring-file-tasks-from-toml)).

> **Trust:** in an untrusted directory, a task file whose headers contain Tera, or that lists `secrets`, must be trusted before it loads. Remote (HTTP / `git::`) task files whose `#MISE` headers contain template syntax need the naming config to be trusted (2026.10.11). Under paranoid mode every task file outside global/system config needs `mise trust <file>`.

**Each `#MISE` line is TOML.** Arrays and inline tables may span lines as long as every line keeps the prefix, and dotted keys build tables without braces:

```bash
#MISE depends=[
#MISE   "lint",
#MISE   "test",
#MISE ]
#MISE tools.node="20"
```

**Alternative header syntax** (if formatters modify `#MISE`):
```bash
# [MISE] description="Build"
# [MISE] depends=["lint"]
```

**Accepted comment prefixes** for the usage block — `#`, `//`, and `::` — each in three forms (`#USAGE`, `# [USAGE]`, `#[USAGE]`); whitespace between the marker and keyword is allowed (`# USAGE`, `// USAGE`):

```javascript
#!/usr/bin/env node
//USAGE flag "-v --verbose" help="Enable verbose output"
//MISE description="Run node script"
```

```python
#!/usr/bin/env python
#MISE description="Hello from Python"
```

```powershell
#!/usr/bin/env pwsh
#MISE description="Hello from PowerShell"
```

> Header lines are recognized by the regex `^(?:#|//|::)\s*(?:(USAGE|MISE)|\[(USAGE|MISE)\])(.*)$`. An executable file **without a shebang** still has its `#MISE`/`#USAGE` lines parsed (a UTF-8 BOM is stripped first). The `#USAGE` block starts at the **first `#USAGE` line, wherever it is** (code before it doesn't end anything); blank comment lines continue it, and the first non-comment line **after it starts** ends it — later `#USAGE` lines are ignored. `#MISE` lines are read from anywhere in the file. File-task `#USAGE` lines are **not** Tera-rendered (TOML `usage` strings are — see [Tera in TOML usage](usage-spec.md#tera-rendering-of-toml-usage-strings)).

**Root-level `mount`** (mise 2026.7.0+) hoists a spec produced by another command onto the task:
```bash
#!/usr/bin/env bash
#MISE description="Wrapper around an external CLI"
#USAGE mount "mise run run-release -- --usage-spec"
```

> **File tasks only.** mise hoists root-level `mount` nodes into a synthetic command, and that hoisting runs only for file-task headers. A root-level `mount` inside a TOML `usage = '''…'''` field loads and passes `mise tasks validate`, but is **ignored at run time** (its flags become positionals) — put it inside a `cmd` block there. Mounts resolve at **completion** time, not on `--help`.

**Direct script execution** bypasses discovery (path must start with `/`, `./`, `../`, `C:\`, `.\`, or `..\`):
```bash
mise run ./path/to/script.sh
```

Since **2026.10.5** a relative path resolves from the **current directory** (not the project root); the script still runs in the config root. In a terminal mise offers to `chmod +x` a non-executable file; otherwise it errors with `` `<resolved path>/build.sh` is not executable. Run: chmod +x <resolved path>/build.sh ``.

> **Windows file tasks:** a `#!/usr/bin/env pwsh` file without `.ps1` runs from a temporary `.ps1` copy, so `$PSScriptRoot` points at the copy — use `$env:MISE_TASK_DIR`. With both `build.ps1` and `build.cmd`, all are kept as tasks; run one by its full name.

> By default mise executes these files directly. Setting `use_file_shell_for_executable_tasks = true` (default `false`) routes them through `unix_default_file_shell_args` (`sh`) / `windows_default_file_shell_args` (`cmd /c`) instead.

### Configuring File Tasks from TOML

🔴 **Breaking change in 2026.9.13.** A `[tasks.<name>]` block whose name matches a file task now either *configures* or *replaces* that script:

| TOML block | Effect |
|------------|--------|
| **No** `run` / `run_windows` / `file` | **Metadata overlay.** Description, env, depends, timeout, sandbox fields, etc. apply to `mise-tasks/<name>.*`; the script stays the command. |
| **Has** `run` / `run_windows` / `file` | **Replaces** the script. `[tasks.hello] run = "…"` makes `mise-tasks/hello.sh` disappear as a separate task. |
| `[tasks."hello.sh"]` | Targets only that one script (when `hello.sh` and `hello.js` share the stem `hello`). |

```toml
# mise-tasks/deploy is a file task; overlay metadata without touching the script
[tasks.deploy]
description = "Deploy (overrides the header description)"
timeout = "10m"                      # verified: the file task is killed after 10m
env = { REGION = "us-east-1" }
```

- **Metadata source:** when a script remains the command, only the **highest-precedence** TOML block supplies metadata — lower blocks are not combined with it. Each `conf.d` folder fragment is its own root.

- **Precedence:** a command replaces a script only if its block comes from the config whose `task_config.includes` selected the script dir, or a higher-precedence config. Lower-precedence `run` blocks are ignored (their metadata still applies).
- **Layered definitions:** a metadata-only `[tasks.check]` in `mise.local.toml` overlays `[tasks.check]` from `mise.toml` — so you can add `depends` or env locally without copying the command. Only blocks *above* the highest-precedence command definition contribute.
- **Windows pairs:** `build.sh` + `build.ps1` form one task `build` (Windows picks the native one); `[tasks."build.ps1"]` defines a separate task.

> Before 2026.9.13, `[tasks."hello.sh"] run = …` was ignored and `[tasks.hello] run = …` left `hello.sh` as a second task. Configs relying on that now behave differently.

### Task Grouping (Namespaces)

Subdirectories create namespaced tasks (colon separator):

```
mise-tasks/
├── build              → build
├── test/
│   ├── _default       → test (default task)
│   ├── unit           → test:unit
│   ├── integration    → test:integration
│   └── e2e            → test:e2e
└── lint/
    ├── eslint         → lint:eslint
    └── prettier       → lint:prettier
```

The file named `_default` executes when invoking the directory name without a subtask. In TOML, quote the key: `[tasks."test:unit"]`.

### Remote Tasks

Fetch tasks from external sources:

```toml
# HTTP
[tasks.build]
file = "https://example.com/build.sh"

# Git — SSH
[tasks.release]
file = "git::ssh://git@github.com/org/repo.git//scripts/release.sh?ref=v1.0.0"

# Git — HTTPS
[tasks.build]
file = "git::https://github.com/org/repo.git//path?ref=main"
```

Format: `git::<protocol>://<url>//<path>?ref=<ref>` — ref is optional (defaults to repo's default branch). `https`, `http`, and `ssh` protocols work; Azure DevOps `_git` and `git@ssh.dev.azure.com:v3/…` forms are accepted without `.git`. The `//path` must be relative with no `.`/`..` segments.

> 🔴 **Experimental gate added (2026.10.5, breaking).** `git::` task files (`file = "git::…"`) and `git::`/`oci::` entries in `task_config.includes` now **require `experimental = true`**. Without it, `mise run` errors with ``loading task `remote` from a `git::` file is experimental``, and includes are skipped with ``WARN skipping task include … from an `oci::` source is experimental``; inspection commands (`tasks ls|info|deps|validate`) warn and don't fetch. Plain HTTP `file = "https://…"` is **not** gated.

Remote files cached in `$MISE_CACHE_DIR` (git in `$MISE_CACHE_DIR/remote-git-tasks-cache`, OCI catalogs in `$MISE_CACHE_DIR/remote-oci-tasks-cache`). Clear with `mise cache clear`. Disable cache with `MISE_TASK_REMOTE_NO_CACHE=true` or `--no-cache` (which, since 2026.9.16, also re-clones `git::` includes and refetches remote dependency tasks).

> Since 2026.7.11, remote HTTP and Git-backed task files have their `#MISE` headers parsed (`tools`, `description`, `hide`, inline TOML overrides), and git-backed files are made executable after clone and on cache hits.

Remote git includes in `[task_config]` (**experimental**) — a directory, or a single TOML task file:
```toml
[settings]
experimental = true

[task_config]
includes = [
  "git::https://github.com/myorg/shared-tasks.git//tasks?ref=main",
  "git::https://github.com/org/repo.git//tasks/release.toml?ref=v1.0.0",   # one TOML file
]
```

**OCI task catalogs (2026.9.18; experimental-gated since 2026.10.5)** — publish a task directory as an OCI artifact and include it by `docker pull`-style reference:
```toml
[task_config]
includes = [
  "oci::ghcr.io/myorg/shared-tasks:1.0.0",
  "oci::registry.example.com/platform/tasks@sha256:0f1e2d3c...",   # pin a digest for immutability
]
```
```bash
oras push ghcr.io/myorg/shared-tasks:1.0.0 build.toml scripts/deploy   # publish
```
- The artifact unpacks as a task directory (executable file tasks + `.toml` task files). Files starting with `#!` are made executable. Titled layers become files at that path; untitled tar/tar+gzip/tar+zstd layers extract at the root. Symlinks and device files are rejected.
- Every blob is digest-verified, but there is **no signature verification** — pin `@sha256:` for anything you don't control.
- Credentials come from `docker login` / `podman login`, else anonymous. Loopback registries use plain HTTP; others need `oci.insecure_registries`.
- Cached in `$MISE_CACHE_DIR/remote-oci-tasks-cache` **per reference — a moved tag is not re-pulled**. Refresh with a new tag/digest, by deleting the dir, or with `MISE_TASK_REMOTE_NO_CACHE=true`.
- Not supported: single-file `oci::` includes and a `//subpath` selector (the artifact is always a directory).

> `includes` entries are Tera-rendered (`config_root`, `env`, `vars`). The top-level [`include`](settings.md#remote-config-includes-include) key is a different feature and **cannot carry tasks** — share tasks through `task_config.includes`.
