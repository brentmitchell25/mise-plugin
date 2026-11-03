## Environment Configuration

**Contents**

- [Basic Variables](#basic-variables)
- [Special Directives (env._)](#special-directives-env_)
- [Profiles / Configuration Environments (MISE_ENV)](#profiles--configuration-environments-mise_env)
- [Required and Redacted Variables](#required-and-redacted-variables)
- [Secrets (fnox, SOPS, age)](#secrets-fnox-sops-age)
- [Templates (Tera)](#templates-tera)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### Basic Variables

```toml
[env]
NODE_ENV = 'production'
DEBUG = 'app:*'
PORT = 3000

# Default — applied only when the var is unset/empty (existing non-empty values win).
EDITOR = { default = 'vim' }

# Unset a variable
UNWANTED_VAR = false
```

Value forms (schema `oneOf`, all with `unevaluatedProperties: false`): scalar (string/int/bool; `false` unsets) · `{ value, tools?, redact?, required? }` (with `value`, **only `required = false`** is accepted — `{ value = "x", required = true }` is a parse error since 2026.10.7) · `{ default, tools?, redact? }` (**no `required`**) · `{ required, tools?, redact? }` · `{ age = … }` (experimental).

| Sub-key | Type | Meaning |
|---------|------|---------|
| `value` | string\|int\|bool | The value; `false` removes the variable |
| `default` | string\|int | Fallback used only when the var is unset **or empty** |
| `required` | bool \| string | Must be supplied by the starting environment or **any other config file** (parent, global, `mise.local.toml`…); string form is the error help text |
| `redact` | bool | Redact from logs. `redact = false` opts **out** of matching global `redactions` patterns |
| `tools` | bool | Defer resolution until after tools set their environment |
| `age` | string \| `{value, format}` | Experimental; `format` is `raw` or `zstd` |

```bash
mise set NODE_ENV=development   # Set via CLI
mise set                        # View all (key / value / source table)
mise set -E staging NODE_ENV=staging  # subcommand -E/--env writes mise.staging.toml
                                # (a GLOBAL `mise -E staging set …` only selects which files load)
mise set --override AWS_PROFILE=personal   # 2026.10.14: global override file (beats all project config)
mise set --prompt PASSWORD      # Interactive prompt — input is MASKED only with --age-encrypt
cat private.key | mise set --stdin MY_KEY  # From stdin (single key, read to EOF)
mise unset NODE_ENV             # Remove
mise env                        # Export all
mise env --json                 # Export as JSON
mise env --json-extended        # JSON with source + tool attribution
mise env --dotenv               # Export as dotenv
mise env --redacted             # Show only redacted variables
mise env --values               # Show only values
mise env -s bash                # Shell-specific output
mise env --paths                # 2026.10.11: PATH entries mise manages, in PATH order
```

> A `PATH` key in `[env]` is **ignored** by `mise exec`, `mise env`, and activated shells — use `_.path`.

Env vars resolve **before** tools, so they can configure tool-install subprocesses. Use `tools = true` on a value to defer evaluation until tool paths/versions exist.

> Variables that configure mise itself (`MISE_DATA_DIR`, `MISE_INSTALLS_DIR`, …) are read at process start and **cannot** be set from `[env]` — set them in the shell or CI environment.

### Special Directives (`env._`)

The reserved key `_` is a TOML table for configuration, since nested env vars make no sense.

> **Deprecation:** the older `env.mise.*` spelling is deprecated (removal **2026.12.0**) — use `env._.*`. Likewise, the `value`/`values` keys inside `_.file`/`_.path`/`_.source` objects are deprecated in favor of `path` (a string *or* an array); removal **2026.12.0**.

#### `_.path` — Add to PATH

Options: `path` (string or array) / `paths` (array), `tools`, `position` (`"prepend"` default \| `"append"`, 2026.10.24). **No `redact`/`expand`** (accepted but ignored).

```toml
[env]
_.path = ["tools/bin", "{{config_root}}/scripts"]
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }  # Lazy eval after tools
_.path = { path = ["~/bin", "~/.cargo/bin"], position = "append" }  # after tool paths
```

`position = "append"` puts entries **after the paths of mise-managed tools and before the inherited `PATH`**, so a personal `~/bin` can't shadow a mise tool; it orders only the paths mise adds (`tools = true` doesn't change it). Any other value is a parse error; `position` on `_.file`/`_.source` is accepted and ignored.

Relative paths resolve against `{{config_root}}`; `~` paths are accepted.

#### `_.file` — Load from .env/json/yaml/toml files

Options: `path` (string or array; `paths` array also accepted), `tools`, `expand`, `redact`, **`find_up`** (2026.10.12).

```toml
[env]
_.file = '.env'
_.file = ['.env', '.env.local', '.env.{{env.MISE_ENV}}']
_.file = { path = ".secrets.yaml", redact = true }
_.file = { path = ".env.json", expand = true }
_.file = { path = [".env", ".env.local"], find_up = true }   # search config root + parents
```

**Format is decided by the extension:** only `.json`, `.yaml`, and `.toml` are structured — **anything else (including `.yml`) is parsed as dotenv**. Globs (`.env.*`) are supported and missing files are skipped. YAML values are read as written (`1.10` stays `1.10`, `02134` keeps its zero). sops/age-encrypted variants work too.

**`find_up`** (bool, default `false`) searches the **config root** and every parent for the file name (an absolute path searches its own dir and parents). Each entry is searched on its own and later entries win; several plain relative names in one `path` list are searched together (closest directory wins; within a directory the later name wins) — names containing a template, `$VAR`, `~`, or an absolute path are searched separately. 🔴 **Changed in 2026.10.16/17:** relative `find_up` now starts at the **config root**, not the cwd — so in **global** config (root `~`) it no longer finds project files. Restore cwd-based search there with `_.file = { path = "{{cwd}}/.env", find_up = true }`.

**`env_file` setting / `MISE_ENV_FILE`** takes **several** names since 2026.10.14 (`env_file = [".env", ".env.local"]`, `MISE_ENV_FILE=.env:.env.local` — `;` on Windows); it searches **up from the cwd**, applies **after** all `[env]` (overriding it), and skips files `_.file` already loaded. `_.file` relative paths resolve against `config_root`.

> **Changed behavior (2026.7.14):** values in **structured** files (JSON/YAML/TOML) are **literal by default** again. Previously, with `env_shell_expand` on, every structured value was shell-expanded — corrupting literals like bcrypt-style hashes and potentially pulling in matching process-environment values. Opt back in per file with `expand = true`, which also lets values reference vars defined earlier in the same file, an earlier file, or an earlier `[env]` block. A global `env_shell_expand = false` overrides `expand = true`.

> 🔴 **Dotenv parsing changed (2026.10.3).** mise replaced dotenvy with its own `mise-dotenv` parser (from dotenv-ng), used by both `_.file` and the `env_file` setting / `MISE_ENV_FILE`. Same-file references always expand, and **`${VAR}` now resolves against the file's own earlier assignments first**, then values loaded so far (with `expand = true`) or the process env — previously an ambient value (e.g. one `mise activate` exported from another `.env`) won. New operators: `${VAR:-default}`, `${VAR:+alt}`, `${VAR:?message}` (aborts with `required variable 'X' is not set: <message>`); `${VAR:=x}` stays **literal**. Multiline quoted values work. `_.file` still fails on a syntax error; the `env_file` setting keeps the assignments read before the error and warns once.
>
> **Quoting restored (2026.10.7):** single-quoted text is literal (backslashes included); adjacent quoted/unquoted parts join (`A='it'\''s'` ⇒ `it's`); quotes inside unquoted values are removed, so **single-quote JSON** (`CONFIG='{"debug": true}'`); an unmatched quote stays literal (`NAME=O'Brien`) unless a partner quote appears in a trailing comment.

> **Deprecation:** the top-level `env_file`/`dotenv` and `env_path` keys are deprecated (removal **2027.4.0**). Migrate to `_.file` and `_.path`.

#### `_.source` — Source shell scripts

Options: `path` (string or array), `tools`, `redact`. The directive itself can also be an array of strings or tables (see [Multiple Identical Directives](#multiple-identical-directives)).

```toml
[env]
_.source = "./setup-env.sh"
_.source = { path = "my/env.sh", redact = true }
_.source = ["./script_1.sh", "./script_2.sh"]   # ordered
```

Scripts must be sourceable by **bash** (run with `--norc` since 2026.10.12); shebangs are ignored. On Windows this requires a real POSIX bash (Git for Windows / MSYS2) — common install locations are probed even if bash isn't on PATH, `MISE_BASH_PATH` overrides, WSL's `bash.exe` is never auto-selected, and prepended `/c/...` / `/cygdrive/c/...` entries are converted to Windows form. **Skipped** (not an error) under `safe = true`; under `paranoid = ["env_files"]` each sourced file needs content-bound trust.

**PATH changes from a sourced script are prepend-only:** `export PATH="/new/bin:$PATH"` works (the original PATH must remain an exact suffix); appending, removing, reordering, or replacing PATH entries is **silently ignored**. Relative prepended entries resolve against `config_root`; empty entries are dropped.

#### `_.python.venv` — Auto-activate Python venv

```toml
[env]
_.python.venv = ".venv"                                  # string shorthand
_.python.venv = { path = ".venv", create = true }
_.python.venv = { path = ".venv", python = "3.12" }      # specify python version
_.python.venv = { path = ".venv", create = true, uv_create_args = ["--seed"] }  # uv venv with pip
```

| Option | Type | Purpose |
|--------|------|---------|
| `path` | string | venv location (required; templates supported) |
| `create` | bool | Auto-create if missing (default `false`) |
| `python` | string | Python version used to create the venv |
| `python_create_args` | string[] | Args passed to `python -m venv` |
| `uv_create_args` | string[] | Args passed to `uv venv` |

Uses `uv venv` when `uv` is on PATH, else `python -m venv`. uv omits pip by default — add `uv_create_args = ["--seed"]`. Activation needs `mise activate`/`mise exec`; **shims alone don't add the venv `bin/` to PATH**.

> This is a **separate codepath** from the `python.uv_venv_auto` setting — `uv_create_args` here is not used by `uv_venv_auto`. For uv-managed projects (with `uv.lock`), prefer `python.uv_venv_auto` (`"source"` or `"create|source"`); the legacy `true` value is being phased out. The `[tools] python = { virtualenv = ".venv" }` tool option is **deprecated** (warned since 2026.7.13, removal ~2027.7.0) in favor of `_.python.venv`.

#### Plugin-Provided Directives

Plugins can register custom `_.<name>` directives (the TOML table lands in the plugin's `ctx.options`):

```toml
[env]
_.my-plugin = {}
_.my-plugin = { option1 = "value1", option2 = "value2" }
_.vault-secrets = { vault_url = "https://vault.example.com", secrets_path = "secret/myapp" }
```

mise loads the plugin, calls its `MiseEnv` hook for env vars and `MisePath` hook for PATH entries, and applies them on `mise env` / shell integration.

#### Multiple Identical Directives

TOML doesn't allow duplicate keys. A second `_.source` (or `_.file`/`_.path`) key in one `[env]` table is a parse error that rejects the **whole file**: `mise env` exits 1 and every shim in that tree fails. Two forms keep per-entry options (verified on 2026.10.25):

```toml
# A directive array whose items are tables: each item keeps its own options
[env]
_.source = [{ path = "./script_1.sh", redact = true }, { path = "./script_2.sh", tools = true }]
```

```toml
# Array-of-tables: blocks resolve in order
[[env]]
_.source = "./script_1.sh"
[[env]]
_.source = { path = "./script_2.sh", tools = true }
```

- A plain-string array (`_.source = ["a.sh", "b.sh"]`) runs both scripts in order but carries **no** `tools`/`redact`. A single table with `path = [...]` applies its options to every path.
- One file can't mix `[env]` and `[[env]]` (`duplicate key` at `[[env]]`). Convert the whole file.
- A directive written below a `[tasks.*]` header belongs to that task. Uncommented it's `unknown field '_'`; commented out it silently does nothing.

Worked example, including ordering rules for a script that reads a variable: [per-tree identity](per-tree-identity.md#two-_source-scripts-in-one-tree).

> Array parsing was tightened in 2026.7.12: accidental array forms for `vars` and task `env`/`vars` are rejected. Intentional `[[env]]` and directive-level arrays (`_.source`, `_.file`, `_.path`) remain supported.

#### Lazy Evaluation (`tools = true`)

```toml
[env]
NODE_VERSION = { value = "{{ tools.node.version }}", tools = true }
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }
```

Plain entries resolve in written order, so a script without `tools = true` sees only the variables above it. `tools = true` entries resolve after **every** plain entry from **every** config file. A global `_.source = { …, tools = true }` therefore sees a project's override of a variable it reads (verified on 2026.10.25). See [per-tree identity](per-tree-identity.md#global-default).

### Profiles / Configuration Environments (`MISE_ENV`)

"Profiles" were renamed **configuration environments**; the `profile` setting / `MISE_PROFILE` is deprecated in favor of `MISE_ENV`.

Three ways to set `MISE_ENV`:
1. CLI: `-E development` / `--env development`
2. Env var: `MISE_ENV=development`
3. `.miserc.toml` / `.miserc.local.toml`: `env = ["development"]`

**`MISE_ENV` cannot be set in `mise.toml`** — it must be known before mise.toml is discovered.

```bash
MISE_ENV=staging mise run deploy
```

This loads `mise.staging.toml` in addition to `mise.toml`. Config file **precedence** (highest first):

1. `mise.{MISE_ENV}.local.toml`
2. `mise.local.toml`
3. `mise.{MISE_ENV}.toml`
4. `mise.toml`

Comma-separated supports multiple environments: `MISE_ENV=ci,test` — rightmost wins **among files of the same kind**; a local file still beats every shared one (`mise.ci.local.toml` overrides `mise.test.toml`). Since 2026.10.5 an environment file in a **parent** directory no longer overrides an ordinary file in a child directory. Also recognized: `mise/config.{MISE_ENV}.toml`, `.config/mise.{MISE_ENV}.toml`. `MISE_OVERRIDE_CONFIG_FILENAMES` bypasses all of it (matched literally since 2026.10.19).

**`.miserc.toml`** is loaded very early — before config discovery and before Settings. Lookup order (highest first): **in each directory from cwd upward** — `.miserc.local.toml`, `.miserc.toml`, `.config/miserc.toml` → `~/.config/mise/miserc.local.toml` → `~/.config/mise/miserc.toml` → `/etc/mise/miserc.toml`. `--no-config` skips miserc discovery too (2026.10.2).

**Personal env selection without editing a shared file** (2026.9.13 / 2026.9.17):
```toml
# .miserc.local.toml — per checkout (add to core.excludesFile; create per worktree)
env = ["native"]
```
```toml
# ~/.config/mise/miserc.local.toml — per machine, loads regardless of cwd
env = ["work"]
```
Explicitly set fields override the shared file at the same level; omitted fields inherit. `env` **replaces** the inherited list (`env = []` clears it). CLI `-E` and `MISE_ENV` still win over both.

Only **seven** settings are `.miserc`-settable:

| Key | Env var | Default |
|-----|---------|---------|
| `env` | `MISE_ENV` | `[]` |
| `auto_env` | `MISE_AUTO_ENV` | unset |
| `ceiling_paths` | `MISE_CEILING_PATHS` | `[]` |
| `ignored_config_paths` | `MISE_IGNORED_CONFIG_PATHS` | `[]` — **2026.8.9+** supports relative entries and globs (incl. recursive `**`). Entries in `.miserc.toml` resolve against the declaring file; `MISE_IGNORED_CONFIG_PATHS` resolves against the invocation directory. |
| `override_config_filenames` | `MISE_OVERRIDE_CONFIG_FILENAMES` | `[]` |
| `override_tool_versions_filenames` | `MISE_OVERRIDE_TOOL_VERSIONS_FILENAMES` | `[]` |
| `env_conf_d` | `MISE_ENV_CONF_D` | unset — opt into environment-specific dotted `conf.d` filenames now (see [File Hierarchy](settings.md#file-hierarchy)) |

Its Tera context is limited. **Global/system miserc:** `env.*`, `config_root`, `cwd`, `xdg_*`, `mise_config_dir`, `system_brew_prefix`, all filters/tests, and all functions **except** `exec()`, `read_file()`, `read_json/yaml/toml()`; `vars`, `tools`, `mise_env`, `mise_bin`, and `mise_pid` are unavailable. 🔒 **Project `.miserc.toml` (2026.10.19 security):** `env` is empty and `get_env()` unavailable; only `if`/`elif`/`else`/`endif` statements; only the functions `arch`, `os`, `os_family`, `libc`, `avx2`, `num_cpus`, and a short filter/test whitelist; max **16 KiB**. A project file that breaks these limits (or fails to parse) is left unrendered **with a warning** (`not rendering the templates of the .miserc.toml in <dir>: …`); other render failures, such as an undefined variable, silently fall back to the raw content. Your own `~/.miserc.toml` can't ignore the global config. `.miserc.toml` never needs trust.

**Platform auto-envs** (`auto_env` setting): `unix` (**not defined on Windows**); `linux`/`macos`/`windows`; `linux-x64`/`macos-arm64`/`windows-x64`. Files like `mise.windows.toml` and `mise.macos-arm64.toml` load automatically with matching lockfiles (`mise.windows.lock`). Precedence: `unix` < `{os}` < `{os}-{arch}` < explicit `MISE_ENV` entries. Affects **config discovery and lockfile selection only** — platform envs are not added to `{{ mise_env }}` or the `MISE_ENV` passed to tasks. **Disabled by default today; warns from 2026.12.0 when a platform file would newly load; defaults to enabled in 2027.6.0.** Setting it in `mise.toml` does nothing (early-init).

### Required and Redacted Variables

```toml
[env]
# Required — error if not set
DATABASE_URL = { required = true }
DATABASE_URL = { required = "Set postgres connection string" }

# Redacted — hidden from output
API_KEY = { value = "secret_key_here", redact = true }

# Opt a value OUT of a matching redactions pattern
TEST_TOKEN = { value = "not-sensitive", redact = false }

# Pattern-based redactions (top-level, not under [env])
redactions = ["*_TOKEN", "SECRET_*", "API_*"]
```

Required vars are satisfied by a pre-existing environment value or by a config file processed **later** (e.g. `mise.local.toml`). Regular commands (`mise env`) **fail** with the help text; `mise hook-env` (shell activation) **warns and continues** so shell setup isn't broken.

Redaction requires a non-`raw` output mode — tasks with `raw = true` bypass interception. The default `prefix` (jobs > 1) and `interleave` (jobs = 1) styles already print full logs with redaction applied; only `replacing`/`timed` hide lines. A value supplied by the caller for a `{ required = true, redact = true }` var (or one overriding a redacted `default`) is still redacted.

### Secrets (fnox, SOPS, age)

mise documents three approaches:

1. **fnox (recommended)** — a separate [@jdx](https://github.com/jdx) project: a full secret manager with remote storage (1Password, AWS Secrets Manager) and remote encryption (AWS KMS). Two ways in: the native **`[secrets.fnox]` + per-task `secrets`** integration (**experimental**, 2026.10.4 — see [Task-Scoped Secrets](task-configuration.md#task-scoped-secrets-experimental)), or wrapping commands with `fnox exec -- mise ...`. The `mise-env-fnox` plugin / `_.fnox-env` directive are deprecated. Project grants can't use keys from your **personal** fnox config (2026.10.23, see the task-scoped secrets restrictions).
2. **SOPS (experimental)** — encrypt entire files, load via `env._.file`.
3. **Direct age (experimental)** — encrypt individual env vars inline in `mise.toml`.

**SOPS (age-backed):**

```bash
mise use -g sops age
age-keygen -o ~/.config/mise/age.txt
sops encrypt -i --age "<public key>" .env.json
```

```toml
[env]
_.file = ".env.json"
_.file = { path = ".env.json", redact = true }   # with redaction
```

**Key resolution:** `MISE_SOPS_AGE_KEY` → `MISE_SOPS_AGE_KEY_FILE`/`sops.age_key_file` → `SOPS_AGE_KEY_FILE` → `SOPS_AGE_KEY` → `~/.config/mise/age.txt`.

**Settings:** `sops.age_key`, `sops.age_key_file` (unset by default — `~/.config/mise/age.txt` is only the last step of the lookup chain), `sops.rops` (default `true`, native Rust impl), `sops.strict` (default `true`). `sops.age_recipients` is a **deprecated no-op** (warns 2026.10.5, removed 2027.10.5) — encrypt with `mise set --age-encrypt --age-recipient <r> KEY=value` instead.

> The external `sops` CLI has no TOML support. mise decrypts SOPS `.env.toml` only with the default `sops.rops = true` (the built-in engine, which supports **age only**). With `sops.rops = false` mise shells out to the `sops` CLI, which supports AWS KMS, GCP KMS, Azure Key Vault, Vault, and PGP — but encrypted TOML then fails. mise finds that CLI in the project's tools, then `PATH`, skipping mise shim and command-wrapper dirs (2026.10.20), so a shim alone doesn't count (`sops command not found` under `sops.strict`).

**Secret hygiene (2026.10.3, security):**
- `__MISE_DIFF` / `__MISE_SESSION` — inherited by every child of an activated shell, task, and `mise x` — now store a `blake3:<hex>` **digest** per managed value instead of plaintext (the value itself is still in the child's env; low-entropy values are brute-forceable).
- **Secret-bearing environments are not written to the env cache:** never for an `age` value, a sops-encrypted `_.file`, a plugin returning `cacheable = false`, redacted `[vars]`, or redacted `default`/`required` directives. Directives with `redact = true` (and plugins returning `redact = true`) are skipped unless `env_cache_redacted = true` (2026.10.25), which caches them encrypted with the key `mise activate` puts in the shell env; values stay masked on a cache hit. Settings-level `redactions` patterns alone do **not** mark the env uncacheable. `MISE_DEBUG=1` logs why an env wasn't cached.
- `_.my-plugin = { redact = false }` now **overrides** the plugin's redact preference (and isn't passed to the plugin); a non-boolean `redact` is a config error.
- `mise x -- fish` no longer passes env values in fish's argv (visible in `ps`).

**Direct age encryption:**

The `age` tool is **not** required — support is built into mise. Defaults to your SSH key (`~/.ssh/id_ed25519` or `~/.ssh/id_rsa`) when present.

```bash
mise settings set experimental=true
mise set --age-encrypt DB_PASSWORD=supersecret
mise set --age-encrypt --prompt DB_PASSWORD   # keeps it out of shell history
```

Flags: `--age-encrypt`, `--age-recipient <KEY>` (repeatable), `--age-ssh-recipient <PATH|KEY>` (repeatable), `--age-key-file <PATH>`, `--no-redact`.

```toml
[env]
DB_PASSWORD = { age = { value = "<base64>", format = "zstd" } }
```

`format` is `raw` or `zstd` (zstd used automatically above ~1KB).

**Decryption identity order:** `MISE_AGE_KEY` → `age.identity_files` → `age.key_file` → `~/.config/mise/age.txt` → SSH identities (`age.ssh_identity_files` plus common defaults).

**Encryption recipient defaults** (when `--age-encrypt` is used with no explicit recipients): public keys derived from identities in `~/.config/mise/age.txt`, plus keys inferred from SSH private keys with a matching `.pub`. If none are found, the command errors.

**Settings:** `age.identity_files`, `age.key_file`, `age.ssh_identity_files`, `age.strict` (default `true` — no identity, no decryptable identity, or an invalid payload is a **hard failure** rather than a partially-resolved environment).

Decrypted values are redacted by default — set `redact = false` on the entry to opt out.

### Templates (Tera)

mise.toml values support Tera templates. **mise 2026.7.1+ uses Tera v2 by default.** The TOML structure itself is not templated and must remain valid TOML.

```toml
[env]
PROJECT_DIR = "{{config_root}}"
LOG_FILE = "{{config_root}}/logs/{{now() | date(format='%Y-%m-%d')}}.log"
NODE_PATH = "{{env.npm_config_prefix}}/lib/node_modules"
PROJECT_NAME = "{{ cwd | basename }}"
```

**Available template variables:**

| Variable | Type | Description |
|----------|------|-------------|
| `env.*` | HashMap | Current environment variables |
| `config_root` | PathBuf | Project root for the config file (`~/src/foo/.config/mise/config.toml` → `~/src/foo`) |
| `cwd` | PathBuf | Current working directory |
| `config_source` | String | **2026.8.15+.** Absolute path of the config file the template is written in (vs `config_root`, its directory). Also available in `.tool-versions`, `[env]` directives, and a task's own `env` — **not** in `run`, `dir`, or file-task headers. Not symlink-resolved (use `\| canonicalize`) |
| `mise_config_dir` | String | **2026.10.19.** `MISE_CONFIG_DIR`, else `xdg_config_home/mise` |
| `system_brew_prefix` | String | **2026.10.13.** Homebrew prefix (`MISE_SYSTEM_BREW_PREFIX`) |
| `mise_bin` | String | Path to mise binary |
| `mise_pid` | String | Process ID |
| `mise_env` | Vec | Configuration environments from `MISE_ENV` (**undefined** if unset; excludes platform envs) |
| `tools` | HashMap | Installed tool info (`.version`, `.path`). Becomes an array with multiple versions (`tools.node[0].version`). Requires `tools = true` in env directives. |
| `usage` | HashMap | Task arguments/flags (task run scripts only). Hyphenated: `usage["dry-run"]`. Not shell-escaped. |
| `vars` | HashMap | Values from `[vars]` |
| `xdg_cache_home` / `xdg_config_home` / `xdg_data_home` / `xdg_state_home` | PathBuf | XDG directories |

**Key Tera functions:**

| Function | Description |
|----------|-------------|
| `exec(command, [cache_key], [cache_duration])` | Execute shell command, return stdout. `cache_duration="1d"`. Runs (and resolves relative paths) in the **config file's directory** — for `.config/mise.toml` that's `.config/`, not the project root. In `[env]` it sees earlier `[env]` entries. **Runs whenever the template renders, including during `--dry-run`** (except bootstrap hook commands) — keep it side-effect free. Blocked under `safe = true`. |
| `get_env(name, [default])` | Get env var from the **original process** env (compat helper; prefer `env.*`). `default` applies only when absent, not when empty. Refused in safe mode. |
| `read_json(path)` / `read_yaml(path)` / `read_toml(path)` | **2026.10.15.** Parse a file relative to the config file's directory: `{{ read_json(path='package.json')['volta']['node'] }}`. Need a trusted config; refused in safe mode, `.miserc.toml`, and OCI dotfile templates. |
| `arch([x64=…], [arm64=…])` | System architecture (`x64`, `arm64`); kwargs rename: `{{ arch(x64="amd64", arm64="aarch64") }}` |
| `os([macos=…], …)` | Operating system (linux, macos, windows); kwargs rename |
| `os_family()` | OS family (unix/windows) |
| `libc([gnu=…], [musl=…])` / `avx2([yes=…], [no=…])` | **2026.10.9.** Host libc (`gnu`/`musl`) and AVX2 support (`yes`/`no`), with optional renames |
| `num_cpus()` | CPU count |
| `now([timezone])` | Current datetime — **Tera v2 signature**; defaults to UTC, accepts IANA names |
| `choice(n, alphabet)` | Random n-char string |
| `haiku([words], [separator], [digits])` | Random readable name — defaults `words=2`, `separator="-"`, `digits=2` ⇒ `fragrant-hummingbird-32`. Useful for ephemeral branch/preview/container names. |
| `read_file(path)` | Read file contents. Blocked under `safe = true`. |
| `range(end, [start], [step_by])` | Integer array |
| `get_random(start, end, [seed])` | Random integer (`seed` makes it reproducible) |
| `task_source_files([only_changed])` | Resolved source file paths (task scripts only); unmatched patterns omitted. **2026.8.15+:** `only_changed=true` narrows to files changed since the last successful run. |
| `throw(message)` | Raise error |

**Key Tera filters:**

| Filter | Description |
|--------|-------------|
| `lower`, `upper`, `capitalize`, `title` | Case transforms |
| `kebabcase`, `shoutykebabcase`, `snakecase`, `shoutysnakecase`, `lowercamelcase`, `uppercamelcase` | Case conversion (`shoutykebabcase` ⇒ `HELLO-WORLD`) |
| `slug` / `slugify`, `striptags`, `spaceless` | Text normalization |
| `trim`, `trim_start`, `trim_end`, `truncate(length, [end])` | Whitespace / shortening |
| `int`, `float` | Conversions. On a non-number, `int`/`float` (with or without `default=`) still return the default or `0` but **warn** (`tera-v1-int-default`); from 2027.4.0 they error |
| `replace(from, to)`, `regex_replace(pattern, rep)` | Substitution |
| `quote` | Escape and quote string |
| `split(pat)`, `join(sep)`, `shuffle([seed])` | Array operations |
| `first`, `last`, `length`, `reverse` | Collection operations |
| `basename`, `dirname`, `extname`, `file_stem`, `join_path` | Path operations |
| `absolute`, `canonicalize` | Path resolution (`canonicalize` throws if missing; `absolute` doesn't require existence) |
| `file_size`, `last_modified` | File metadata |
| `hash([algorithm], [len])` | SHA256 (default) or BLAKE3 hashing |
| `hash_file([len])` | File BLAKE3 hash |
| `b64_encode([url_safe], [padded])`, `b64_decode([url_safe])`, `json_encode([pretty])` | Encoding |
| `date(format, [timezone])` | Format datetime (jiff crate) |
| `default(value, [boolean])` | Fallback for **undefined** only — empty/false input is kept unless `boolean=true` |
| `abs`, `filesize_format` / `filesizeformat`, `format(spec)` | Numeric formatting |
| `urlencode`, `urlencode_strict` | URL-safe encoding |
| `map(attribute)`, `concat(with)` | **Deprecated v1 compat** — use comprehensions / spread |

**Tera tests:**

| Test | Description |
|------|-------------|
| `defined` | Variable exists |
| `string`, `number`, `map` | Type checks |
| `starting_with(arg)`, `ending_with(arg)`, `containing(arg)`, `matching(regex)` | String checks |
| `before(other, [inclusive])`, `after(other, [inclusive])` | Date comparison |
| `dir`, `file`, `exists` | Path checks (mise custom; refused in safe mode) |
| `semver_matching(requirement=…)` | Version satisfies a semver requirement: `{% if v is semver_matching(requirement=">=1.2") %}`. The **keyword is required** (a positional string fails to parse). One comparator per test: `>=1.2`, `=1.2.3`, `^1.0.0`, `~1.2.0`, `*` work; caret/tilde need a full `X.Y.Z` (partial `^1`, `~1.2` silently don't match); compound ranges (`>=1.2, <2`) and bare versions (`1.2`) are errors — combine two tests with `and`. Verified on 2026.10.25 |
| `odd`, `even`, `divisible_by(n)` | Numeric checks |

**Template syntax:**
- `{{ }}` — Expressions · `{% %}` — Statements · `{# #}` — Comments · `{% raw %} {% endraw %}` — Skip rendering
- Operators: `+`, `-`, `/`, `*`, `%`, `==`, `!=`, `>=`, `<=`, `and`, `or`, `not`, `~` (concat), `in`

**Tera v1 → v2 migration:**

| v1 | v2 |
|----|-----|
| `value \| trim_start_matches(pat="v")` | `value \| trim_start(pat="v")` |
| `value \| trim_end_matches(pat="-beta")` | `value \| trim_end(pat="-beta")` |
| `items \| slice(start=0, end=2)` | `items[0:2]` |
| `[base] \| concat(with="file.txt")` | `[base, "file.txt"]` |
| `items \| map(attribute="name")` | `[item.name for item in items]` |
| `items \| filter(attribute="active")` | `[item for item in items if item.active]` |
| `value \| as_str` | `value \| str` |
| `value \| escape` | `value \| escape_html` |
| `value \| linebreaksbr` | `value \| newlines_to_br` |
| `value is divisibleby(divisor=3)` | `value is divisible_by(divisor=3)` |
| `value is object` | `value is map` |
| `value \| indent(prefix=">")` | `value \| indent(width=1)` (spaces only) |
| `value \| truncate` | `value \| truncate(length=255)` |
| `value \| addslashes` | no direct equivalent — chain `replace(from=…, to=…)` calls |
| `value \| int(default=0)` on a non-number | returns 0 with a `tera-v1-int-default` warning; errors from 2027.4.0 — check the value first |

Other v2 differences: `first`/`last`/`nth` on an empty list still return `""` with a deprecation warning and return null from 2027.4.0 — check `items | length` first; the `unique` filter stays case-insensitive until **2027.4.0**, then becomes case-sensitive; under `tera_v1`, `now()` keeps its old signature.

New v2 syntax: slices (`parts[0:2]`, `parts[-1]`, `name[::-1]`), spread (`[first, ...rest]`, `{...base, key: value}`), comprehensions, optional chaining (`env?.NODE_ENV or "development"`), ternaries (`"prod" if release else "dev"`). Undefined-variable access is **stricter** in v2, and **Tera v1 macros are unsupported**.

```toml
[settings]
tera_v1 = true    # escape hatch — WARNS NOW (since 2026.10.0), removed 2027.4.0
```

> In a shared `mise.toml`, prefer the env form `[env] MISE_TERA_V1 = true` — older mise versions treat it as a normal env var rather than erroring on an unknown setting.

> **v1 compatibility now warns (2026.10.x, verified).** Using a v1 helper prints e.g. `mise WARN deprecated [tera-v1-trim-start-matches]: … Use trim_start(pat=...) instead. This will be removed in mise 2027.4.0.` (likewise `[tera-v1-concat]`, `map`, …), and `tera_v1 = true` itself emits `deprecated [setting.tera_v1]`. Migrate now: `items | concat(with=extra)` → `[...items, ...extra]`.

**Tera components (v2 only)** replace v1 macros: define `{% component wrap(opt="none") %}…{% endcomponent %}` and call `{{ <wrap opt={vars.opt} /> }}`. Definitions must live in the **same** `run`/template string — there is no shared component library.

**Shell-style variable expansion** — `env_shell_expand` **defaults to `true`** (verified on 2026.8.0):
```toml
[env]
LD_LIBRARY_PATH = "$MY_LIB:$LD_LIBRARY_PATH"
PATH_SAFE = "${VAR:-default}"      # With default
CLEAN = "${UNSET_VAR:-}"           # Empty if unset (no warning)
```

Supported forms: `$VAR`, `${VAR}`, `${VAR:-default}`, `${VAR:-}`. Expansion runs **after** Tera rendering, so both can be mixed. Undefined vars without a default are left unexpanded and produce a warning. Opt out with `env_shell_expand = false`.
