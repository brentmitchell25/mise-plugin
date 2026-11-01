## Configuration and Settings

**Contents**

- [File Hierarchy](#file-hierarchy)
- [Remote Config Includes (include)](#remote-config-includes-include)
- [Idiomatic Version Files](#idiomatic-version-files)
- [Key Settings Reference](#key-settings-reference)
- [Minimum Version](#minimum-version)
- [Automatic Environment Variables](#automatic-environment-variables)
- [CI/CD Integration](#cicd-integration)
- [IDE Integration](#ide-integration)
- [MCP Server](#mcp-server)
- [Key Environment Variables](#key-environment-variables)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### File Hierarchy

Config files in per-directory precedence order (highest first):

1. `mise.local.toml` (gitignored)
2. `mise.toml`
3. `mise/config.toml`
4. `mise/conf.d/*/mise.toml` (folder fragments)
5. `mise/conf.d/*.toml` (visible form, 2026.8.13+)
6. `.mise/config.toml`
7. `.mise/conf.d/*/mise.toml`
8. `.mise/conf.d/*.toml`
9. `.config/mise.toml`
10. `.config/mise/mise.toml`
11. `.config/mise/config.toml`
12. `.config/mise/conf.d/*/mise.toml`
13. `.config/mise/conf.d/*.toml` (alphabetical)
14. `.tool-versions`

Any can also appear as dotfiles (`.mise.toml` beats `mise.toml`), and every `config.toml` can have a `config.local.toml` beside it. `config_root` is the project directory **except** for `.config/mise/mise.toml` / `.config/mise/mise.local.toml`, whose root is `.config/mise/`.

**Global override tier (2026.10.14)** — `~/.config/mise/config.override.toml` (and `config.<env>.override.toml`) outranks **all** project config, including `mise.local.toml` and `mise.<env>.local.toml`, for every section (docs: `https://mise.jdx.dev/configuration.html#override-file`):

```toml
# ~/.config/mise/config.override.toml
[env]
AWS_PROFILE = "personal"
[settings]
jobs = 8
[tools]
node = "24"
```

It is trusted by path, never a default write target (use `--override` on `use`/`unuse`/`set`/`unset`/`settings set|add|unset`/`config get|set|explain`; conflicts with `--global`/`--path`), and **not loaded when `MISE_GLOBAL_CONFIG_FILE` is set**. Tools declared there resolve from the **global** `mise.lock` and are exempt from the project's `--locked` check. `mise config ls` shows a `Tier` column (`--json` adds `"override": true`); `mise doctor` labels these files. Unrelated to `MISE_OVERRIDE_CONFIG_FILENAMES`.

**`conf.d` fragments (2026.8.7+)** work in project, global (`~/.config/mise/conf.d`), and system (`/etc/mise/conf.d`) directories. With `env_conf_d = true`, fragments named `*.<env>.toml` (plus `.local` variants) load only when that configuration environment is active.

> ⚠️ **Dotted fragment names (`node.tools.toml`) — reverted breaking change.** 2026.8.9 briefly read the extra dot as an environment selector; **2026.8.11 reverted it**. Today dotted names still load **unconditionally**, but warn as **deprecated** (since 2026.8.10): in **2027.8.10** the suffix becomes an environment selector by default. Opt in now with `env_conf_d = true` in a **miserc** file (or `MISE_ENV_CONF_D`); `env_conf_d = false` keeps legacy behavior and silences the warning. Either way, **use hyphens** — `node-tools.toml`.

**`conf.d` folder fragments (2026.9.14)** — a *folder* inside any `conf.d` directory is its own config root:

```text
~/.config/mise/conf.d/
├── git.toml                  # single-file fragment
└── git-tools/                # folder fragment (may be a symlink to a dotfiles checkout)
    ├── mise.toml             # always loaded
    ├── mise.local.toml       # always loaded, usually gitignored
    ├── mise.linux.toml       # loaded when the `linux` environment is active
    ├── mise.linux.local.toml
    └── gitconfig             # a dotfile source resolved relative to the folder
```

- Only `mise.toml`, `mise.local.toml`, `mise.<env>.toml`, `mise.<env>.local.toml` are read; **not recursive**; folders starting with `.` are ignored. `mise.<env>.toml` is unaffected by the `env_conf_d` migration.
- The folder is the **config root**: relative paths resolve inside it, `{{ config_root }}` is the folder, and **tasks run in the folder**. Its `[task_config]` applies only to its tasks, and its `includes` resolve inside it.
- Order: folder fragments load **after** single-file fragments in the same `conf.d` (alphabetically by folder) and **before** that directory's regular config. Verified on 2026.10.3: in a project, `.mise/conf.d/<folder>/mise.toml` ranks below the project `mise.toml`, but `.mise/conf.d/<folder>/mise.dev.toml` (with `dev` active) ranks **above** it.
- This is the migration target for the deprecated `[bootstrap].config_roots`.

**Full stack, lowest → highest:**
```
/etc/mise/conf.d/*.toml, /etc/mise/config.toml, /etc/mise/config.<env>.toml
~/.config/mise/conf.d/*.toml, config.toml, config.<env>.toml, config.local.toml, config.<env>.local.toml
<ancestor dirs>/mise.toml …
<project>/mise.toml, mise.<env>.toml, mise.local.toml, mise.<env>.local.toml
<project>/<subdir>/mise.toml
~/.config/mise/config.override.toml, config.<env>.override.toml   ← highest (2026.10.14)
```

**Legacy:** `.tool-versions` (asdf-compatible)

**Schema validation:**
- `https://mise.jdx.dev/schema/mise.json`
- `https://mise.jdx.dev/schema/mise-task.json`

mise searches upward from cwd to root (stops at `MISE_CEILING_PATHS`). Merge behavior by section (`https://mise.jdx.dev/configuration.html#merge-behavior-by-section`):
- **`[tools]`:** per tool — the nearest entry replaces the **whole** entry, including options
- **`[env]` / `[vars]`:** per variable
- **`[settings]`:** per setting (an **untrusted** project's `[settings]` is ignored entirely, 2026.10.13)
- **`[tasks]`:** per name — a higher-precedence definition **with a command** (`run`/`run_windows`/`file`) replaces the task; a **metadata-only** block overlays it (see [Configuring File Tasks from TOML](task-definition.md#configuring-file-tasks-from-toml))
- **`[tool_config]`:** config-root-scoped, not merged invocation-wide
- **`[secrets.*]`:** per field, project config only
- **`[daemons]`:** per daemon · **`[daemons_settings]`:** per key (ignored in global/system config)

**Write targeting:** `mise use`, `mise set`, `mise unset` write to the lowest-precedence file in the highest-precedence directory — with both present, writes go to `mise.toml`, not `mise.local.toml` (`mise use --env local node@20` targets `mise.local.toml`). `mise unuse` targets the **first loaded config that declares the tool** (`--path` to choose); `mise config get/set` default to the **highest-precedence loaded** file (`--file` to choose).

**Useful commands:**
```bash
mise cfg / mise config     # Show loaded files in precedence order (Tier column)
mise config get|ls|set     # Read/write individual config keys (set --type infer|string|integer|float|bool|list|set)
mise config explain [KEY] [-f FILE | -g | --system | --override] [-J]   # 2026.10.14: each key with its
                           # line + schema description; flags unknown ("did you mean") and deprecated
                           # keys; reads plain TOML, so it works on configs mise refuses to load
mise config set --append  redactions '*_TOKEN'   # 2026.8.15+ list-key append
mise config set --remove  redactions '*_TOKEN'   # 2026.8.15+ list-key remove
mise config set --global|--system <key> <value>  # 2026.8.15+ explicit scope targeting
mise ls --current          # Active versions with overrides
mise doctor                # Diagnose setup issues
mise doctor project        # Run this project's [doctor.checks]
mise fmt                   # Format mise.toml — 2026.9.6+ also SORTS certain list values
                           # (redactions, task sources/outputs, task_config.global_inputs,
                           #  input_groups) by defined rules
```

**`.tool-versions` format:**
```text
node        20.0.0       # comments are allowed
ruby        3            # fuzzy version
erlang      ref:master   # compile from vcs ref
go          prefix:1.19  # latest 1.19.x
shfmt       path:./shfmt # custom runtime
node        sub-2:lts    # numeric subtraction from lts
python      sub-0.1:latest
```

### Remote Config Includes (`include`)

Added **2026.9.18** (not experimental; docs `https://mise.jdx.dev/configuration.html#include`). Pull shared config fragments from a git repo, an OCI artifact, or (2026.10.6) a **local** TOML file:

```toml
include = [
  "git::https://github.com/myorg/platform.git//mise.toml?ref=main",
  "oci::ghcr.io/myorg/platform-config@sha256:0f1e2d3c...",   # artifact with mise.toml at its root
  "./config/tools.toml",                                      # 2026.10.6: local, relative to this file
  "/etc/mise/team.toml",                                      #            or absolute
]

[tools]
node = "22"   # this file's own entries override the included ones
```

- **Forms:** `git::<url>//<path>.toml?ref=<ref>`, `oci::<registry>/<repo>[:tag|@sha256:<digest>]`, or a local `.toml` path (read on **every** load, never cached; an **error under paranoid mode**; skipped in safe mode); a string or an array. Since 2026.10.19 an include pinned to a commit SHA must check out **exactly** that commit (replace an annotated-tag object ID with the commit SHA).
- **Precedence:** each fragment is merged **into** the including file (never a config file of its own) and ranks **just below it**; later entries override earlier ones. Hooks of the same type from both run (shared first); aliases merge per name; the including file's PATH entries come first.
- **Allowed in a fragment:** `[tools]`, `[tool_alias]`, `[env]`, `[vars]`, `[hooks]`, `[alias]`, `[shell_alias]`, `[plugins]`, `[wrappers]`, `[_]`, **`[bootstrap]`** (2026.10.5 — merged key by key, the including file wins), `min_version` (enforced).
- **Errors (not silent):** a nested `include`; `[settings]` and monorepo keys; `[tasks]`, `task_config`, `task_templates` (share tasks via [`task_config.includes`](task-definition.md#remote-tasks) instead); per-file sections like `[dotfiles]`, `[daemons]`, `redactions`.
- Relative paths (`_.file`, …) and `{{ config_root }}` resolve against the **including** file; tools lock into the including file's lockfile.
- **Trust:** inherits the including file's trust (an untrusted repo can't trigger a fetch). Safe mode never fetches for project config. **Paranoid mode requires a full commit SHA (40 or 64 hex, `?ref=<sha>`) or `@sha256:` digest** — a branch/tag is an error.
- **Cache:** `$MISE_CACHE_DIR/config-includes`. SHA/digest refs are never refetched. Branch/tag refs refresh when older than `fetch_remote_versions_cache` (1h), **only** from commands that look at remote versions (`install`, `up`, `use`, `config`); hook-env, `mise ls`, `mise exec`, shims, and offline mode use the cache. A failed refresh keeps the cached copy with a warning; a cold cache with no network is an error. `mise cache clear` forces a refetch.

### Idiomatic Version Files

Disabled by default. Enable per-tool:
```bash
mise settings add idiomatic_version_file_enable_tools python
mise settings add idiomatic_version_file_enable_tools go        # go.mod support
mise settings add idiomatic_version_file_enable_tools dagger task lefthook
```

Supported files include `.nvmrc`, `.node-version`, `package.json`, `.python-version`, `.python-versions`, `.ruby-version`, `Gemfile`, `.go-version`, `go.mod`, `rust-toolchain.toml`, `.java-version`, `.sdkmanrc`, `global.json`, `.terraform-version`, `.bun-version`, `.deno-version`.

- **`go.mod` (2026.7.13+):** a `toolchain goX.Y.Z` directive is an exact pin. Since **2026.10.9** an exact `go X.Y.Z` line with no `toolchain` line is also used as the version (no warning). Reading a patch-less `go X.Y` minimum (and `CMakeLists.txt` `cmake_minimum_required`) as a version floor **warns and is removed in 2026.11.0** — use `toolchain goX.Y.Z`, `go X.Y.Z`, `.go-version`, or `mise.toml`. `idiomatic_version_file_ignore_minimum_versions` goes away at the same time.
- **`.nim-version`** is an idiomatic file for `nim` (2026.9.13).
- **`go.work` (2026.9.12+):** with `go` enabled, an active `go.work`'s `toolchain` line selects the Go version and — as with the `go` command — member `go.mod` files are **ignored in workspace mode**. `GOWORK` (`auto`, `off`, or an absolute path) is honored.
- **`.bazelversion` (2026.9.12+):** an idiomatic version file for `bazel` when enabled. Only **concrete releases** are read — `latest`, `last_green`, `8.x`, and commit hashes select nothing rather than failing.
- **`idiomatic_version_file_ignore_minimum_versions`** (bool, default `false`) ignores idiomatic fields that declare only a *minimum* compatible version.
- **Structured parsers (2026.7.15+):** registry `idiomatic_files` entries support `version_regex`, `version_json_path`, and `version_expr`, giving **in-process parsing with no plugin or shell execution** for 11 tools — Dagger, Task, chezmoi, CMake, Earthly, golangci-lint, GoReleaser, Lefthook, Pixi, pre-commit, Ruff (`dagger.json`, `Taskfile.yml`, `.chezmoiversion`, …).
- **Disable individual files per tool (2026.7.17+)** with `tool:filename` pairs — e.g. keep `.nvmrc` selecting Node while stopping node from reading `devEngines.runtime` in `package.json`, with pnpm still reading it:
  ```bash
  mise settings add idiomatic_version_file_disable_files node:package.json
  ```
- Since 2026.7.18, `idiomatic_version_file_enable_tools` set in a config root's own `[settings]` is honored by monorepo-wide commands (`mise ls --monorepo`, `mise install --monorepo`).

### Key Settings Reference

mise ships **351 settings** (leaf count from `https://mise.jdx.dev/schema/mise.json` on 2026.10.19: 177 top-level keys, 40 of which are nested namespaces — note `mise settings --all` prints only the ~213 that resolve to a value, omitting unset optional ones). **Changed defaults/types:** `lockfile_mode` `merge` → **`generate`** (2026.10.19); `java.shorthand_vendor` `openjdk` → **`temurin`** (2026.10.5); `paranoid` bool → **bool or list** (2026.10.14); `env_file` path → **list** (2026.10.14); `fetch_remote_versions_cache` `"1h"` → **unset** (2026.10.11, see below). This is a representative subset — run `mise settings --all` (or `mise settings ls`) for the live list, and `mise settings set <key> <value>` / `mise settings get <key>` to manage them.

```toml
[settings]
# Execution
jobs = 8                    # Concurrent jobs (MISE_JOBS)
experimental = false        # Enable experimental features
yes = false                 # Auto-answer prompts (MISE_YES) — global only
safe = false                # Inert config-reader mode — global only

# Task defaults (example values — task.output/timeout/timings are UNSET by default)
task.output = "prefix"      # prefix|interleave|keep-order|replacing|timed|silent (unset → prefix if jobs>1)
task.timeout = "10m"        # Default task timeout
task.timings = true         # Show elapsed time
task.quiet = false          # Suppress mise's own task chatter (replaces output = "quiet")
task.skip = ["slow-task"]   # Tasks to skip
task.skip_depends = false   # Skip dependencies
task.source_freshness_hash_contents = false  # blake3 content check
task.auto_infer = []        # e.g. ["node"] (stable since 2026.10.6)
task.cache_max_size = "2GiB"  # experimental
use_file_shell_for_executable_tasks = false  # Run file tasks through a shell

# Shells — ALL FOUR ARE GLOBAL-CONFIG-ONLY since 2026.7.14
unix_default_inline_shell_args = "sh -o errexit -c"
unix_default_file_shell_args = "sh"
windows_default_inline_shell_args = "cmd /c"
windows_default_file_shell_args = "cmd /c"
windows_powershell_no_profile = true   # -NoProfile for pwsh tasks (default true)

# Environment
env_shell_expand = true     # Shell-style expansion — DEFAULT TRUE
env_cache = false           # experimental — cache computed environment (never caches secret-bearing envs, 2026.10.3)
env_cache_ttl = "1h"        # Cache TTL
env_file = [".env", ".env.local"]  # MISE_ENV_FILE (list since 2026.10.14; ':'-separated env var)
auto_env = false            # Auto-load platform config files (default-on in 2027.6.0)

# Tool management
auto_install = true         # Auto-install missing tools
exec_auto_install = true    # Auto-install on mise x/run
not_found_auto_install = true
not_found_auto_install_registry = false  # 2026.9.17: install UNCONFIGURED tools into global config
auto_install_disable_tools = []
disable_backends = ["asdf"] # Disable backends (new installs only)
disable_default_registry = false  # Only affects vfox and asdf shorthands
enable_tools = []           # Allowlist (unset = all; empty = none)
disable_tools = []
pin = false                 # Default --pin for mise use
lockfile = true             # Create + update lockfiles (unset = update existing ones only)
lockfile_platforms = []     # Extra platforms to resolve in the lockfile
locked = false              # Fail if no pre-resolved URLs
prereleases = false         # Allow pre-release versions for fuzzy requests
registry_floating = false   # Fetch current registry data instead of release-pinned snapshots
registry_cache_ttl = "1h"
shared_install_dirs = []    # Read-only dirs searched for installed versions
system_deps = "prompt"      # prompt|auto|warn|ignore — vfox systemDependencies handling

# Security
paranoid = false            # Extra-secure behavior — global only; bool or a list of
                            # "config" | "env_files" | "plugins" | "provenance" (2026.10.14)
gpg_verify = true           # Built-in OpenPGP verification (no external gpg binary)
slsa = true                 # SLSA provenance verification
github_attestations = true  # GitHub Artifact Attestations
provenance_api_failures_fatal = true  # Treat provenance API failures as install errors
netrc = true                # Honor ~/.netrc for HTTP auth (netrc_file overrides path)
minimum_release_age = "7d"  # built-in default is 24h when unset (every backend since
                            # 2026.10.15); "0s" disables
minimum_release_age_excludes = []  # Tools exempt from the release-age delay
locked_verify_provenance = false   # Re-verify at install (auto-on with paranoid)
not_found_system_fallback = true   # false = shim for a missing tool fails loudly
                                   # instead of falling back to a PATH binary
all_compile = false         # Never use precompiled binaries for any tool
url_replacements = {}       # Map of URL patterns → replacements for all requests
use_versions_host_track = true     # Anonymous download statistics
gix = true                  # Use gix for git operations (false = shell out to git)
libgit2 = true              # Use libgit2 for git operations
self_update.auto = false    # 2026.10.4 (was auto_update): opt-in mise self-update; true or an
                            # interval like "1d" — global only
self_update.check_duration = "7d"   # (was auto_update_check_duration) — global only
self_update.minimum_release_age = "24h"  # 2026.9.17; unset → minimum_release_age → 24h; "0s" = immediate
tool_update.global_auto = false     # 2026.10.4: auto-update GLOBAL tools (bool | interval) — global only
tool_update.check_duration = "24h"  # interval for per-tool auto_update = true (min 1h) — global only
download_cache = true               # 2026.10.7: verified download cache ($MISE_CACHE_DIR/downloads-cas)
download_cache_max_size = "2GiB"    # 0 = unlimited (LRU)
extract_max_ratio = 100             # 2026.10.19: decompression-bomb guard (0 = off) — global only
lockfile_mode = "generate"          # 2026.10.19 default (was merge); "merge" = incremental
lockfile_auto_prune = true          # 2026.10.4: full `mise lock` drops tools absent from config
install_layout = "legacy"           # experimental: "identity" (Windows + experimental ⇒ identity)
prune.exclude = []                  # tools never pruned
upgrade.auto_prune = true   # 2026.8.15+: removal is now DEFERRED, not immediate —
                            # the replaced version is kept for upgrade.prune_after
upgrade.prune_after = "24h" # grace period before a replaced version is auto-pruned

# Sandbox (deny-by-default policy)
[settings.sandbox]
deny_all = false
deny_read = false
deny_write = false
deny_net = false
deny_env = false

# Performance / network
[settings]
# fetch_remote_versions_cache — UNSET by default since 2026.10.11: outdated/upgrade/ls-remote/lock/latest
#   refresh after 1h; install/use/run reuse a list for minimum_release_age − 1h (23h by default,
#   clamped 1–24h). Setting it applies one value to every command.
fetch_remote_versions_timeout = "20s"
http_timeout = "30s"                # Connect / between-reads timeout
http_download_timeout = "30m"       # Total download wall-clock including retries
http_retries = 3                    # HTTP retries with exponential backoff (0 = none)
cache_prune_age = "30d"             # Age before cached downloads are pruned
use_versions_host = true            # Use mise-versions shared version cache
offline = false                     # Block all HTTP requests
prefer_offline = false              # Prefer cached data

# UI / shell
color = true                # Colorized output
color_theme = "default"     # auto|default|charm|base16|catppuccin|dracula
terminal_progress = true    # OSC 9;4 terminal progress indicators
verbose = false             # Verbose install output
activate_aggressive = false # Push tool bin-paths to the front of PATH

# Windows
windows_shim_mode = "exe"   # exe|file|hardlink|symlink
windows_executable_extensions = ["exe", "bat", "cmd", "com", "ps1", "vbs"]

# Status line
status.missing_tools = "if_other_versions_installed"
status.show_env = false
status.show_tools = false
status.show_deps_stale = true
status.truncate = true

# Node-specific
[settings.node]
corepack = false            # Enable corepack
compile = false             # Compile from source
verify = true
npm_shim = true             # bash wrapper at bin/npm that reshims after `npm install -g`
flavor = ""                 # Alternate distribution flavor

# NPM backend
[settings.npm]
package_manager = "auto"    # auto|npm|aube|aube_cli|bun|pnpm
shell_out = false           # Route metadata + installs through the npm CLI

# Python-specific
[settings.python]
uv_venv_auto = false        # false | "source" | "create|source" | true (legacy form deprecated)
uv_venv_create_args = []
venv_create_args = []
compile = false             # Compile from source
venv_stdlib = false         # Prefer stdlib venv module
precompiled_flavor = "install_only_stripped"

# Ruby-specific
[settings.ruby]
compile = false             # PRECOMPILED IS NOW THE DEFAULT (2026.8.0) — set true to force source
ruby_install = false        # Use ruby-install instead of ruby-build
precompiled_url = "jdx/ruby"

# Aqua security
[settings.aqua]
cosign = true
slsa = true
github_attestations = true
minisign = true
baked_registry = true
registry_cache_ttl = "1w"
# registries = ["myorg/aqua-registry"]  # replaces deprecated registry_url

# Cargo
[settings.cargo]
binstall = true             # Use precompiled binaries
binstall_only = false
binstall_quickinstall = false
# binstall_native = true    # graduated from experimental in 2026.7.16

# PyPI (pipx.* are hidden older names)
[settings.pypi]
uvx = true                  # Use uv instead of pipx
# keyring_provider = "subprocess"   # 2026.10.16 (default disabled)

# Conda
[settings.conda]
channel = "conda-forge"

# Age encryption (experimental)
[settings.age]
key_file = "~/.config/mise/age.txt"
strict = true

# Sops encryption
[settings.sops]
rops = true                 # Use native Rust implementation (required for TOML)
strict = true               # Fail on decryption errors
age_key_file = "~/.config/mise/age.txt"

# Hook environment
[settings.hook_env]
cache_ttl = "0s"            # Cache hook-env dir checks (useful on NFS)
chpwd_only = false          # Only run on directory change, not every prompt

# OCI images
[settings.oci]
default_from = "debian:bookworm-slim"
default_mount_point = "/mise"
insecure_registries = []

# Dotfiles
[settings.dotfiles]
default_mode = "symlink"    # symlink|symlink-each|copy|template
root = "~/.dotfiles"
relative_symlinks = false   # 2026.9.14: Stow-style relative links (per-entry `relative` overrides)
windows_symlinks = false    # 2026.10.15: real symlinks for symlink-each on Windows

# OpenTelemetry (experimental, 2026.9.13)
[settings.otel]
enabled = false             # export mise run traces (needs OTEL_EXPORTER_OTLP_*ENDPOINT)
logs = false                # also export task output lines

# System packages
[settings.system_packages]
sudo = true                 # set managers = [...] to pick package managers
# elevate_with = "run0"     # 2026.10.9: sudo → run0 → doas fallback; pin one here
ostree = "error"            # 2026.10.9: rpm-ostree hosts — "error" | "layer"

# Java (core tool)
[settings.java]
shorthand_vendor = "temurin"   # 2026.10.5 default (was openjdk)

# Write targets for `mise use -g`, `bootstrap packages use --global`, `mise dot add` (global only)
[settings.write_targets]
# tools = "{{ config_source | dirname }}/10-tools.toml"
```

**Nested namespaces (40):** `age`, `aqua`, `cargo`, `conda`, `dotfiles`, `dotnet`, `erlang`, `forgejo`, `github`, `github_relay`, `gitlab`, `go`, `history`, `hook_env`, `java`, `node`, `npm`, `oci`, `otel`, `packslip`, `pipx`, **`prune`**, `pypi`, `python`, `ruby`, `rust`, `sandbox`, `self_update`, `shims`, `skills`, `sops`, `spm`, `status`, `swift`, `system_packages`, `task`, **`tool_update`**, `upgrade`, **`write_targets`**, `zig`. (`task.cache` and `history.watch` nest one level deeper.) **Bold = new since 2026.10.3.**

**Notable new top-level settings:**

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `activate_shims` | bool | `true` | `false` keeps tool shim dirs off PATH during activation and hooks (2026.9.2+), so external [command wrappers](wrappers.md#command-wrappers-wrappers) keep working |
| `shims_dir` / `system_shims_dir` / `system_installs_dir` | path | — | Shim/install locations for system-scoped and collocated layouts (2026.9.0+) |
| `shims.exclude` | string[] | `[]` | Command names mise should **never** create shims for, leaving them to the system (2026.9.10+) |
| `lockfile_mode` | enum | **`generate`** | `generate` (default since 2026.10.19, no longer experimental — rebuild from current requests, reuse unchanged entries, lock other platforms) or `merge` (incremental). `provenance_verified` is never a trust signal. |
| `locked_scopes` | string[] | `["global","project","system"]` | Config scopes where invocation-wide `locked` mode is enforced |
| `no_env` | bool | unset | Do not load env vars from config files (matches `--no-env`) |
| `no_hooks` | bool | unset | Do not execute hooks from config files (matches `--no-hooks`) |
| `task.quiet` | bool | `false` | Suppress mise's own output while executing tasks |
| `upgrade.prune_after` | duration | `"24h"` | Grace period before versions replaced by `mise upgrade` are auto-pruned |
| `idiomatic_version_file_ignore_minimum_versions` | bool | `false` | Ignore idiomatic version-file fields that only declare a **minimum** compatible version (now hidden; removed with the floors in 2026.11.0) |
| `not_found_auto_install_registry` | bool | `false` | **2026.9.17.** Command-not-found installs an *unconfigured* registry tool when exactly one provides the command, adding it to global config |
| `self_update.minimum_release_age` | string | unset (→ `minimum_release_age` → `24h`) | **2026.9.17.** Release-age delay for self-update, auto-update, and update notices. **Not** global-only |
| `dotfiles.relative_symlinks` | bool | `false` | **2026.9.14.** Relative symlink targets for `symlink`/`symlink-each` |
| `truncate` | bool | `true` | Terminal-width truncation; `--truncate`/`--no-truncate` override. **Auto-disabled when a coding agent is detected** (2026.9.6+) |
| `disable_update_warning` | bool | `false` | Suppress the "a newer mise is available" warning (2026.9.5+) |
| `disable_hints` | string[] | `[]` | Silence specific hint messages by name |
| `self_update.repository` / `self_update.api_url` | string | `jdx/mise` / `https://api.github.com` | Source used by `mise self-update` |
| `github_relay.*` | — | — | `concurrency` (`8`, 1–32, excess fails closed), `request_timeout` (`5m`), `max_duration` (`0s` = session lifetime), `log_requests` (`false`), `log_format` (`text`\|`jsonl`) — for [`mise ssh`](dev-tools.md#cli-commands-for-tools) borrowed GitHub access |
| `packslip.exec` / `packslip.stampers` | bool / array | `false` / unset | See [Packslip Backend](dev-tools.md#packslip-backend-preferred-where-available) |
| `skills.*` | — | — | See [Agent Skills](dev-tools.md#packslip-man-pages-and-agent-skills) |
| `history.*` | — | — | See [Dotfiles History](bootstrap.md#dotfiles-history-mise-dot--history) |

**Global-config-only settings (45 in `settings.toml`, 40 excluding deprecated aliases)** — ignored when set from project config: `auto_update`/`auto_update_check_duration` (deprecated → `self_update.auto`/`self_update.check_duration`), `ci`, `extract_max_ratio`, `forgejo.credential_command`, `github.credential_command`, `gitlab.credential_command`, `github.enterprise_hosts`, `github.oauth_cache_get`, `github.oauth_cache_set`, `github.oauth_cache_secure_enclave`, `github_relay.{concurrency,log_format,log_requests,max_duration,request_timeout}`, `history.allow_plaintext_history`, **`history.describe_command`** (global-only since 2026.9.7, a security fix), `locked_scopes`, `paranoid`, `safe`, `self_update.api_url`, `self_update.auto`, `self_update.check_duration`, `self_update.repository`, `shims_dir`, `system_installs_dir`, `system_shims_dir`, `task.cache.remote_oidc_audience`, `task.cache.remote_token`, `task.cache.remote_token_file` (plus their deprecated flat `task.cache_remote_*` aliases), `tool_update.global_auto`, `tool_update.check_duration`, `trusted_config_paths`, `unix_default_file_shell_args`, `unix_default_inline_shell_args`, `windows_default_file_shell_args`, `windows_default_inline_shell_args`, `write_targets.{tools,packages,dotfiles}`, `yes`. Beyond this list, an **untrusted** project's entire `[settings]` table is ignored (2026.10.13). `[daemons_settings]` is the inverse — **ignored from global/system config** — and `[daemon_providers]` is allowed **only** in global config.

> **`gpg_verify` behavior (2026.7.12+):** GPG verification **always runs** when enabled, with Node/Swift signatures verified in-process via rPGP (no external `gpg` binary). Previously a missing `gpg` silently skipped verification — you must now set `gpg_verify = false` explicitly to opt out.

**Settings** generally map to `MISE_` + the upper-snake path (`MISE_JOBS=4`, `MISE_TASK_OUTPUT=interleave`), but ~20 don't follow the rule: `status.*` → `MISE_STATUS_MESSAGE_*`; `python.venv_stdlib` → `MISE_VENV_STDLIB`; `python.pyenv_repo` → `MISE_PYENV_REPO`; `ruby.ruby_build_*`/`ruby.ruby_install*` → `MISE_RUBY_BUILD_*`/`MISE_RUBY_INSTALL*`; `rust.cargo_home` → `MISE_CARGO_HOME`; `rust.rustup_home` → `MISE_RUSTUP_HOME`; `dotnet.dotnet_root` → `MISE_DOTNET_ROOT`; `node.nvm_dir` → `NVM_DIR`; `node.nodenv_root` → `NODENV_ROOT`; `ci` → `CI`. About 25 have no env var (legacy/deprecated settings plus `write_targets.*`). Five settings are **env-only** (must be env vars): `default_config_filename`, `default_tool_versions_filename`, `global_config_file`, `global_config_root`, `system_config_file`.

### Minimum Version

```toml
min_version = '2024.11.1'                               # Hard (errors)
min_version = { soft = '2024.11.1' }                    # Soft (warns)
min_version = { hard = '2024.11.1', soft = '2024.9.0' } # Both
```

### Automatic Environment Variables

Tasks automatically receive:

| Variable | Description |
|----------|-------------|
| `MISE_ORIGINAL_CWD` | Original working directory |
| `MISE_CONFIG_ROOT` | Directory containing mise.toml |
| `MISE_PROJECT_ROOT` | Project root directory (subproject dir in monorepos; stable regardless of cwd) |
| `MISE_MONOREPO_ROOT` | Monorepo root (set inside a monorepo with `monorepo_root = true`) |
| `MISE_TASK_NAME` | Current task name |
| `MISE_TASK_DIR` | Task script directory |
| `MISE_TASK_FILE` | Full path to task script (for an inline TOML task: the config file that defines it) |
| `MISE_ENV` | The active configuration environments |
| `MISE_TASK_COLOR` | ANSI start sequence of the task's prefix colour (empty when colours are off or there's no prefix) |
| `TRACEPARENT` / `TRACESTATE` | With [OpenTelemetry](task-tracing.md#task-tracing-with-opentelemetry-experimental) enabled — nested `mise run` and OTel-aware tools join the trace |

### CI/CD Integration

**GitHub Actions (`jdx/mise-action@v5`):**

```yaml
- uses: actions/checkout@v7
- id: mise
  uses: jdx/mise-action@v5      # latest v5.2.3 (2026-10-24); v5 is the current major
  with:
    version: 2026.10.19   # pin mise (default: newest release ≥ minimum_release_age old)
    minimum_release_age: 24h   # v5 DEFAULT; "0s" restores v4's "newest release" behavior
    # sha256: "…"         # verify the mise binary
    install: true         # run `mise install`
    install_args: "bun"   # extra args to `mise install`
    plugins: |            # v5.1: newline list of `name` or `name url`
      my-plugin https://github.com/me/vfox-my-plugin
    bootstrap: false      # run `mise bootstrap` instead of `mise install`
    bootstrap_skip: "tools,task"
    bootstrap_args: "--yes"
    cache: true           # cache via GitHub cache
    cache_save: true
    cache_save_post: false   # v5.1: save the cache in the post step (catches later-installed tools)
    cache_max_age_days: 30   # v5.2: default key gains -p<N> / {{cache_period}} so caches rebuild periodically (0 = off)
    cache_restore_retry_budget: 0s   # v5.2: retry on HTTP 429 (also cache_save_retry_budget)
    monorepo: true        # v5.2: add --monorepo to `mise install` when [monorepo].config_roots is set
    cache_key: "{{cache_key_prefix}}-{{platform}}-{{install_args_hash}}-{{plugins_hash}}-{{file_hash}}"
    auto_update: false    # v5.1: keep the cached mise until the cache key changes
    experimental: false   # enable experimental features
    log_level: info
    working_directory: .
    reshim: false         # run `mise reshim --all`
    env: true             # export mise environment variables
    export_path: true     # add mise PATH entries to subsequent steps
    add_shims_to_path: true
    fetch_from_github: true   # falls back between GitHub and mise.jdx.dev for the latest version
    # mise_dir: …         # also exports MISE_DATA_DIR (install_dir is deprecated)
    github_token: ${{ secrets.GITHUB_TOKEN }}
    persist_github_token: false   # v5.1.1: token is NO LONGER exported to later steps by default
    # tool_versions: |    # optionally inline .tool-versions content
    # mise_toml: |        # optionally inline a mise.toml
- run: echo "node is ${{ steps.mise.outputs.node }}"   # v5.1 per-tool outputs (+ `versions` JSON)
```

> 🔴 **mise-action v5 breaking changes** (verified against the GitHub API):
> - **v5.0.0:** `minimum_release_age` defaults to **`24h`** — an unpinned run installs the newest stable mise **at least a day old** (release list from a CDN index, no API quota). Set `minimum_release_age: 0s` for v4 behavior.
> - **v5.0.1:** a cached/existing mise binary is integrity-verified (signed checksums or the `sha256` input) before running; version switches do a full install.
> - **v5.1.1:** the GitHub token is **no longer exported** as `MISE_GITHUB_TOKEN` to later steps. `persist_github_token: true` restores it, or pass a different (e.g. read-only) token value.
> - New outputs: `cache-hit`, `versions` (JSON of `{version, requested_version, install_path, source}` per tool), and one output per tool (`steps.mise.outputs.node`).
> - **v5.2.x (2026-10):** the binary is verified against the release's signed `packslip.sigstore.json` — for mise ≥ 2026.9.3 a **missing packslip is a hard error** (v5.2.1); PATH entries come from `mise env --paths`; the cache key also hashes `conf.d/*.toml` (v5.2.2) and searches only the working directory and its parents (v5.2.3) — each key change causes one cache miss.

Behavior worth knowing: PATH entries are added individually via `GITHUB_PATH` from `mise env --paths` (the runner's complete PATH is not copied into `GITHUB_ENV`). When a `mise.lock` exists in the working directory or a parent, the action **automatically appends `--locked`** — unless you supply `mise_toml`/`tool_versions` inputs. `install_args` cannot be combined with `bootstrap: true`. Values flagged `redact = true` or matching `redactions` are masked automatically. Cache-key templating supports `{{version}}`, `{{cache_key_prefix}}` (default `mise-v1`), `{{platform}}`, `{{file_hash}}`, `{{mise_env}}`, `{{install_args_hash}}`, `{{bootstrap_hash}}`, `{{plugins_hash}}`, `{{env.VAR_NAME}}`, `{{default}}`, and `{{#if …}}…{{/if}}` conditionals. `wings_enabled` (experimental mise-wings asset cache via OIDC; needs `permissions: id-token: write`).

**Cache downloads, not installs (2026.10.7+).** The verified download cache is trusted with `--locked`, so caching `downloads-cas` is safer than caching installed tools (a restored *tool* cache is trusted as-is and not re-checked against `mise.lock`):

```yaml
- uses: actions/cache@v6
  with:
    path: ~/.cache/mise/downloads-cas      # macOS: ~/Library/Caches/mise/downloads-cas; Windows: ${{ runner.temp }}\mise\downloads-cas
    key: mise-downloads-${{ runner.os }}-${{ hashFiles('mise.lock') }}
- uses: jdx/mise-action@v5
  with: { cache: false }
```

Lock every CI platform first (`mise lock --platform linux-x64,linux-arm64,macos-arm64`), then `mise install --locked`. `mise generate github-action --write --task=ci` scaffolds a workflow (`-t/--task`, `--name`; uses `actions/checkout@v7` + `jdx/mise-action@v5`).

**GitLab CI** — either commit a wrapper (`mise generate install-script -l -w ./bin/mise`) or use the official image (GitLab accepts at most **two** paths in `cache:key:files` by default):

```yaml
# Committed wrapper (the localized wrapper keeps installs in .mise/ inside the project)
build-job:
  image: debian:13-slim
  cache:
    key:
      prefix: mise-debian13-amd64-w1   # distinct prefix per architecture; bump when regenerating the wrapper
      files: [mise.toml, mise.lock]
    paths: [.mise/installs/, .mise/cache/]
  before_script:
    - apt-get update && apt-get install -y --no-install-recommends curl ca-certificates tar
  script:
    - ./bin/mise install --locked
    - ./bin/mise exec -- npm run build
```

```yaml
# Official -debian image (mise, curl, git, CA certs preinstalled; no wrapper needed)
build-job:
  image: ghcr.io/jdx/mise:2026.10.19-debian
  variables:
    MISE_DATA_DIR: $CI_PROJECT_DIR/.mise
    MISE_CACHE_DIR: $CI_PROJECT_DIR/.mise/cache
  cache:
    key:
      prefix: mise-image-amd64
      files: [mise.toml, mise.lock]
    paths: [.mise/installs/, .mise/cache/]
  script:
    - mise install --locked
    - mise exec -- npm run build
```

**Official Docker images (2026.9.12+)** — `ghcr.io/jdx/mise` and `jdxcode/mise`, for `linux/amd64` and `linux/arm64`, built from the **minisign-verified release binaries**:

| Tag | Contents |
|-----|----------|
| `2026.10.19`, `2026.10`, `latest` | **scratch** image (static binary + CA certificates, **no shell**) — intended for `COPY --from=` |
| `debian`, `*-debian` | Debian slim base with `curl` and `git` |
| `dev` | Unsupported source-built image |

```Dockerfile
FROM debian:13-slim
COPY --from=ghcr.io/jdx/mise:2026.10.19 /usr/local/bin/mise /usr/local/bin/mise
```

> 🔴 **Breaking (2026.9.12):** `:latest` is now the **scratch** image, not a usable base. CI and dev-container users should switch to the `debian` tag and install tools explicitly.

**Generic CI bootstrap:**
```bash
curl https://mise.run | sh
mise install
mise x -- <cmd>

# Skip reinstall if already present (Docker layer caching)
curl https://mise.run | MISE_INSTALL_SKIP_IF_EXISTS=1 sh

# Reproducible: pin the version (bypasses the release-age delay)
curl -fsSL https://mise.run | MISE_VERSION=2026.10.19 sh

# Windows PowerShell (2026.10.15; PS 5.1+, no admin): installs ~\.local\bin\mise.exe + mise-shim.exe,
# verified against SHASUMS256.txt, and adds it to the user Path
irm https://mise.run/powershell | iex
# from MSYS2 / Git Bash:
powershell.exe -NoProfile -Command "irm https://mise.run/powershell | iex"
```

Installer env vars: `MISE_VERSION`, `MISE_INSTALL_PATH`, `MISE_INSTALL_FROM_GITHUB=1` (default source is mise.jdx.dev with GitHub fallback since 2026.10.11; checksums always come from GitHub or are embedded), `MISE_INSTALL_MUSL=1`, `MISE_INSTALL_OS`, `MISE_INSTALL_ARCH`, `MISE_INSTALL_EXT` (`tar.zst`\|`tar.gz`), `MISE_INSTALL_HELP=0`; PowerShell also `MISE_INSTALL_MODIFY_PATH=0`, `MISE_INSTALL_SHIMS_PATH=1`. Install-and-activate endpoints: `https://mise.run/{zsh,bash,fish}`.

> **The installer now waits 24h too (2026.9.17/9.18).** `curl https://mise.run | sh` picks the newest stable release **at least 24h old at run time**. Override with `MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE` (→ `MISE_MINIMUM_RELEASE_AGE` → `24h`; integer + `s|m|h|d|w`, `0s` = immediate); `MISE_VERSION` pins and bypasses it.

Or `mise generate install-script -l -w ./bin/mise` produces a self-contained `./bin/mise` you can commit, so jobs don't re-download mise (the old `mise generate bootstrap` spelling is a hidden alias until 2027.9.0). `-w` with no path writes `./bin/mise`; `--windows` also writes `<PATH>.cmd` (requires `--write`). `-l/--localize` sandboxes `MISE_DATA_DIR`/`MISE_CACHE_DIR` into a `.mise` directory in the project (`--localized-dir <DIR>` to change it); the localized wrapper trusts the project's config and ignores the user's global config, and needs `bash`, `curl`, CA certs, `tar`, and `sha256sum`/`shasum`. Without `--version`, the generated wrapper is pinned to the age-eligible release. The script honors `MISE_VERSION` and `MISE_INSTALL_PATH`.

**packslip install (CI/Docker, 2026.10.x docs):** `packslip install github.com/jdx/mise --pin ps1_nlhmwtfeufglxv5myvwvronk7a [--version 2026.10.19]` (packslip ≥ 1.5.1; Linux x64/arm64, macOS arm64, Windows) installs mise with signer verification pinned to mise's repository.

**Untrusted config in CI:**
```yaml
script: |
  MISE_SAFE=1 mise lock --bump --json
```
Recommended whenever a job resolves tool versions from configuration it doesn't control — most commonly a bot refreshing `mise.lock` on PR branches.

**Useful CI env vars:**
- `MISE_YES=1` — auto-answer prompts
- `MISE_SAFE=1` — inert config reader for untrusted/fork-PR configs
- `MISE_DATA_DIR` — install/cache root
- `MISE_EXPERIMENTAL=1` — unlock experimental features
- `MISE_OFFLINE=1` / `MISE_PREFER_OFFLINE=1` — network policy

### IDE Integration

IDEs inherit the environment from their launch shell and do not reload mise config changes. Because arbitrary `[env]` vars only load when a shim is executed, activate **shims** in your login profile so GUI-launched editors see mise tools:

```bash
# ~/.zprofile / ~/.bash_profile (login, non-interactive)
eval "$(mise activate zsh --shims)"
```

```fish
# ~/.config/fish/config.fish
if status is-interactive
  mise activate fish | source
else
  mise activate fish --shims | source
end
```

```lua
-- Neovim: prepend the shim dir to PATH
vim.env.PATH = vim.env.HOME .. "/.local/share/mise/shims:" .. vim.env.PATH
```

- **VS Code:** extension `hverlin/mise-vscode` (tools, tasks, env, `mise.toml` completion; auto-configuring other language extensions is **off by default** — `mise.configureExtensionsAutomatically`). For task/debug terminals set an automation profile: `"terminal.integrated.automationProfile.osx": { "path": "/bin/zsh", "args": ["--login"] }` (it doesn't affect the extension host or LSPs). Or use `runtimeExecutable: "mise"` with `runtimeArgs: ["exec", "--", "node"]` in launch configs. Remote contexts (SSH/WSL/devcontainer) need mise on that side. Since 2026.10.2 the JSON schema validates per-backend tool options, so typos in `[tools]` now show as editor errors.
- **JetBrains:** plugin `intellij-mise` (tools + run-configuration env), or the asdf-compat symlink `ln -s ~/.local/share/mise ~/.asdf`.
- **Fixed SDK paths (2026.10.8):** point IDE settings that need a stable path (Maven home, JDK home) at `~/.local/share/mise/installs/<tool>/global` — it follows the version **global/system** config selects (see [the `global` link](dev-tools.md#install-layout-download-cache-and-the-global-link)).
- **Xcode:** run tools with `"$HOME/.local/bin/mise" --cd "$SRCROOT" exec -- swiftlint lint`. With User Script Sandboxing, listing `$(SRCROOT)/mise.toml` as an input file is not enough for every tool (it also needs the installed executables/data dirs). Xcode Cloud: commit a wrapper and in `ci_scripts/ci_post_clone.sh` run `cd "$CI_PRIMARY_REPOSITORY_PATH"; ./bin/mise install`, then `./bin/mise exec -- swiftlint lint`.
- **Neovim:** plugin `miser.nvim`, or the shim-PATH snippet above.
- **Emacs:** package `mise.el` — `(add-hook 'after-init-hook #'global-mise-mode)`; or add the shims dir to both `PATH` and `exec-path`.
- **Vim:** `let $PATH = $HOME . '/.local/share/mise/shims:' . $PATH`.

#### Language Server (`mise lsp`, 2026.10.18)

A language server for mise config files over stdin/stdout (docs: `https://mise.jdx.dev/ide-integration.html#language-server`). It serves only files mise loads as config (`mise.toml`, `.mise.toml`, `mise.<env>.toml`, `mise.local.toml`, `mise/config.toml`, `conf.d/*.toml`) and ignores everything else (including `task_config.includes` files) — so it's safe to attach to **every** TOML file. It only **parses**: it never renders templates, runs anything, or uses the network, so opening an untrusted file is safe.

- **Diagnostics:** codes `syntax`, `unknown-key`, `deprecated-key`, `invalid-value`, `unknown-task` (warning), `task-cycle` (error); unknown `[settings]` keys are warnings, other unknown keys errors.
- **Completion/hover** of keys, enum values, registry tool names (offline — no version lookups), and task names in `depends`/`depends_post`/`wait_for`/`run = [{ task }]` and templates in `extends`; **go-to-definition, find references, rename** of tasks within one file; **quick fixes** for deprecated keys (`env_path` → `env._.path`, `task_output` → `task.output`); **formatting** identical to `mise fmt`; **semantic tokens** (2026.10.19).
- Turn features off when another TOML server (Tombi, Even Better TOML) runs alongside via `initializationOptions`: `{"formatting":false,"hover":false,"completion":false,"definition":false,"references":false,"rename":false,"semanticTokens":false}`.

```lua
-- Neovim 0.11+
vim.lsp.config("mise", { cmd = { "mise", "lsp" }, filetypes = { "toml" }, root_markers = { "mise.toml", ".mise.toml" } })
vim.lsp.enable("mise")
```
Helix and Emacs use the same `mise lsp` command. An older mise runs a **task** named `lsp` instead — detect support with `mise help lsp` printing `Usage: mise lsp`.

#### Shell Setup (`/shell-setup.html`)

- **PowerShell:** `(&mise activate pwsh) | Out-String | Invoke-Expression` — cd updates only on PowerShell 7+; 5.1 updates at the next prompt (warning hidden with `$env:MISE_PWSH_CHPWD_WARNING=0`).
- **Nushell:** in `env.nu`: `^mise activate nu | save ($nu.default-config-dir | path join mise.nu) --force`; in `config.nu`: `use ($nu.default-config-dir | path join mise.nu)`.
- **Xonsh:** `execx($(mise activate xonsh))`. **Elvish:** `var mise: = (ns [&])`, `eval (mise activate elvish | slurp) &ns=$mise: &on-end={|ns| set mise: = $ns }`, `mise:activate`.
- **Windows:** native mise can activate bash/zsh/fish inside Git Bash, MSYS2, and Cygwin; **cmd.exe can't be activated** — put `%LOCALAPPDATA%\mise\shims` on the user `Path`. Shell aliases (`[shell_alias]`) work only in bash, zsh, and fish.

> For Bash, edit the **first existing** of `~/.bash_profile`, `~/.bash_login`, `~/.profile` — creating a new `~/.bash_profile` can stop `~/.profile` from being read. On Linux the login profile is read at login, so logout/login is required after editing it. For a fixed SDK path an IDE can't re-resolve, point it at `mise which <bin>` / `mise where <tool>`. JetBrains' asdf-compat symlink works only if `~/.asdf` doesn't already exist.

### MCP Server

```bash
mise mcp    # JSON-RPC over stdin/stdout
```

```json
{"mcpServers": {"mise": {"command": "/abs/path/mise", "args": ["--cd", "/abs/project", "mcp"], "env": {}}}}
```
Use an absolute binary path and `--cd` (Claude Code: `claude mcp add mise -- mise --cd /abs/project mcp`); add `"env": {"MISE_SAFE": "1"}` to inspect an untrusted project.

**Resources** (loaded once — restart the server after config edits): `mise://tools` (`?include_inactive=true`; entries carry `version`, `requested_version`, `install_path`, `installed`, `active`, `source`), `mise://tasks` (`env` is an **array of env directive strings** since 2026.10.5), `mise://env`, `mise://config` (no settings).
**Tools:** `list_commands` (2026.7.16+ — the full command tree with each command's declared effect: `read`, `write`, `destructive`, or unclassified, plus help text and hidden status; `include_hidden` opts in), `run_task` (sets `MISE_YES=1`, no stdin, limited by `task.timeout`), `install_tool` (**implemented 2026.10.5** — `{tool, version}`; version defaults to the configured one, else latest; returns `tool`, `version`, `install_path`; refuses `tool[option=value]`).

> 🔒 `run_task` and `install_tool` **refuse untrusted projects** and never trust implicitly (file tasks in a project without config need `mise trust <dir>`). Since 2026.10.19 the server also refuses untrusted file tasks, task names that look like file paths, and inline tool options.

> **Unclassified is explicitly "unknown", not "safe."** Consumers should treat a missing effect as "ask".

### Key Environment Variables

- `MISE_DATA_DIR` (default `~/.local/share/mise`)
- `MISE_CACHE_DIR` (default `~/.cache/mise`; macOS `~/Library/Caches/mise`; Windows `%TEMP%\mise`)
- `MISE_TMP_DIR` (default `$TMPDIR/mise` or `/tmp/mise`; Windows `%TEMP%\mise-tmp`)
- `MISE_STATE_DIR`, `MISE_INSTALLS_DIR`, `MISE_PLUGINS_DIR`, `MISE_DOWNLOADS_DIR`, `MISE_INSTALL_STORE_DIR` (identity layout), `MISE_SYSTEM_DATA_DIR` (+ `MISE_SYSTEM_INSTALLS_DIR`, `MISE_SYSTEM_SHIMS_DIR`) — full list at `https://mise.jdx.dev/configuration/environment-variables.html`
- `MISE_SYSTEM_CONFIG_DIR` (default `/etc/mise`)
- `MISE_GLOBAL_CONFIG_FILE` (default `~/.config/mise/config.toml`)
- `MISE_GLOBAL_CONFIG_ROOT` (default `$HOME`; used as `{{config_root}}` for global config)
- `MISE_CONFIG_DIR` (default `~/.config/mise`; holds global `config.toml`, `conf.d/`, `miserc[.local].toml`)
- `MISE_SYSTEM_CONFIG_FILE` (default `/etc/mise/config.toml`)
- `MISE_ENV_FILE` (`:`-separated list of names, `;` on Windows — e.g. `.env:.env.local`; parsed by `mise-dotenv` since 2026.10.3)
- `MISE_${TOOL}_VERSION` (e.g., `MISE_NODE_VERSION=20`)
- `MISE_TRUSTED_CONFIG_PATHS` / `MISE_CEILING_PATHS` / `MISE_IGNORED_CONFIG_PATHS` (`:` Unix, `;` Windows)
- `MISE_OVERRIDE_CONFIG_FILENAMES` / `MISE_DEFAULT_CONFIG_FILENAME`
- `MISE_LOG_LEVEL` (trace|debug|info|warn|error), `MISE_LOG_FILE`, `MISE_LOG_FILE_LEVEL`, `MISE_LOG_HTTP`
- `MISE_LOG_VERBOSE_DEPS` — the only way to see h2/hyper/reqwest/rustls logs, even under `-vv`
- `MISE_QUIET` (sets the log level to **`error`** and hides task headers and progress)
- `MISE_NO_CONFIG=1`, `MISE_NO_ENV`, `MISE_NO_HOOKS` — skip config / env / hooks
- `MISE_TIMINGS=1|2`, `MISE_FRIENDLY_ERROR=0|1`, `MISE_LIST_ALL_VERSIONS=1`
- `MISE_CONFIG_FILE` / `MISE_SYSTEM_DIR` — older aliases of `MISE_GLOBAL_CONFIG_FILE` / `MISE_SYSTEM_CONFIG_DIR`
- `MISE_HTTP_TIMEOUT` (30s) / `MISE_HTTP_DOWNLOAD_TIMEOUT` (30m)
- `MISE_TERM_WIDTH` — terminal width override (takes precedence over `COLUMNS`; env-only, not a setting)
- `MISE_BASH_PATH` — bash used for `_.source` on Windows
- `MISE_FISH_AUTO_ACTIVATE` (default on; `0` disables)
- `MISE_RAW` (pipes directly; forces `MISE_JOBS=1`)
- `MISE_DEBUG=1` / `MISE_TRACE=1` — debug / trace logging
- `MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE` — release-age delay for self-update and the installer
- `MISE_NOT_FOUND_AUTO_INSTALL_REGISTRY`, `MISE_DOTFILES_RELATIVE_SYMLINKS`, `MISE_OTEL_ENABLED`, `MISE_OTEL_LOGS` — settings added in 2026.9.13 through 2026.9.17
- `MISE_SELF_UPDATE_AUTO`, `MISE_TOOL_UPDATE_GLOBAL_AUTO`, `MISE_DOWNLOAD_CACHE`, `MISE_INSTALL_LAYOUT`, `MISE_LOCKFILE_MODE`, `MISE_PYPI_KEYRING_PROVIDER`, `MISE_SYSTEM_PACKAGES_ELEVATE_WITH`, `MISE_GITHUB_ENTERPRISE_HOSTS`, `MISE_AFFECTED_{COMMITTED,UNCOMMITTED,UNTRACKED}` — 2026.10.4 through 2026.10.19

**Global CLI flags:** `-C/--cd <DIR>`, `-E/--env <ENV>` (repeatable or comma list), `-j/--jobs <N>`, `-q/--quiet`, `-v/--verbose`, `-y/--yes`, `--raw`, `--locked`, `--silent`, `--no-config`, `--no-env`, `--no-hooks`. (`--output` is accepted globally but hidden — it belongs to `mise run`.)
