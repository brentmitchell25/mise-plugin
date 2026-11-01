## Sandboxing and Safe Mode

**Contents**

- [Sandboxing ([settings.sandbox])](#sandboxing-settingssandbox)
- [Safe Mode (safe / MISE_SAFE=1)](#safe-mode-safe--mise_safe1)
- [Paranoid Mode (paranoid / MISE_PARANOID)](#paranoid-mode-paranoid--mise_paranoid)
- [Trust](#trust)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

Two independent mechanisms: **sandboxing** restricts what a task/exec can touch; **safe mode** makes mise itself refuse to execute anything a config asks for.

### Sandboxing (`[settings.sandbox]`)

Any `--deny-*`/`--allow-*` flag implicitly enables sandboxing. Applies to `mise run` **and** `mise exec`.

```toml
[settings.sandbox]
deny_all = true      # or individually:
deny_read = true
deny_write = true
deny_net = true
deny_env = true
```

Env vars: `MISE_SANDBOX_DENY_ALL`, `MISE_SANDBOX_DENY_READ`, `MISE_SANDBOX_DENY_WRITE`, `MISE_SANDBOX_DENY_NET`, `MISE_SANDBOX_DENY_ENV`.

Per-task fields (flat, not nested) and the matching CLI flags compose with these:

```toml
[tasks.fetch-deps]
deny_all = true
allow_net = ["registry.npmjs.org"]
allow_read = ["{{config_root}}"]
allow_write = ["{{config_root}}/node_modules"]
allow_env = ["NODE_*", "npm_*"]
run = "npm ci"
```

```bash
mise run build --deny-net --allow-read /src
mise x --deny-all --allow-read=. --allow-write=./dist -- npm install
```

**Implicit access:** system paths stay readable (Linux `/usr /lib /lib64 /bin /sbin /etc /dev /proc /sys /tmp /nix /snap /home/linuxbrew` plus the file `/etc/resolv.conf` points to; macOS `/System /Library /usr /bin /sbin /dev /etc /var/run /tmp /private/tmp /private/etc /private/var/run /opt/homebrew /nix` — **not** the rest of `/private`, since 2026.10.19) along with mise tool dirs. `/tmp` (and `/private/tmp`) and `/dev` stay writable. `--allow-write` paths are implicitly readable. `--deny-env` still passes `PATH`, `HOME`, `USER`, `SHELL`, `TERM`, `COLORTERM`, `LANG` (Windows adds `SystemRoot`, `SystemDrive`, `windir`, `ComSpec`, `PATHEXT`, `TEMP`, `TMP`, `USERPROFILE`, `USERNAME`), plus a task's `pass_through_env` and cache `env` vars; `--allow-env <VAR>` implies deny for everything else and supports wildcards.

**Platform matrix:**

| Feature | Linux | macOS | Windows |
|---------|-------|-------|---------|
| Deny/allow reads & writes | Landlock (kernel ≥5.13; **restricting writes needs ≥6.2** since 2026.10.19) | Seatbelt | ❌ |
| Deny all network | seccomp-bpf (x86_64/arm64; also blocks io_uring) | Seatbelt | ❌ (runs unrestricted) |
| Per-host network (`allow_net`) | ❌ **errors** | ❌ **errors** (since 2026.10.5) | ❌ runs unrestricted |
| Env filtering | Built-in | Built-in | ✅ (since 2026.10.5) |

If Landlock is unavailable or cannot apply filesystem restrictions, the command **fails**. On Windows a warning is printed and filesystem/network restrictions are not applied. An empty `allow_write` blocks **all** writes; to write through a project symlink that points outside, name its target. Sandboxing is **not** marked experimental.

### Safe Mode (`safe` / `MISE_SAFE=1`)

**Global-config-only** setting (mise 2026.7.12+) that turns mise into an inert config reader — useful for untrusted repos, fork PRs, and bots that only need to read or bump versions. Safe mode beats paranoid mode.

```bash
MISE_SAFE=1 mise lock --bump --dry-run --json
```

| Safe mode… | What (as of 2026.10.25) |
|------------|-------------------------|
| **Refuses (errors)** | Template `exec()`, `read_file()`, `read_json/yaml/toml()`, `get_env()`; filters `hash_file`, `file_size`, `last_modified`, `canonicalize`; tests `dir`/`file`/`exists`; **running tasks**; tool options `postinstall`, `install_env`, spm `install_command`; asdf plugin scripts (incl. `exec-env`) and plugin installs; **all S3 requests**; `mise secrets`/`--secrets`; `mise daemons`; `[doctor.checks]` |
| **Ignores (skips silently)** | Hooks and `[[watch_files]]` everywhere (incl. bootstrap hooks); `_.source` everywhere; project `[env]`, `[vars]`, and env directives; project `[settings]`; project `path:` versions; project `[shell_alias]`, `[wrappers]`, rust `mr_boxington`, and **lazy** tools; project `[tool_alias]` entries targeting global tools; remote includes and remote task files; `[deps]`; daemon declarations; project `[bootstrap]`, `[dotfiles]`, `[dotfile_groups]` |
| **Template context** | `env` is **empty** outside global/system config — a project template reading an env var without a `default` fails to render |
| **Credentials** | Forge tokens and netrc credentials are not sent to hosts that project config chooses |

Version resolution still works for HTTP-based backends and Go. Because the config cannot do anything, safe mode **skips the trust requirement** (and a write in safe mode records no trust).

### Paranoid Mode (`paranoid` / `MISE_PARANOID`)

Global-only. Since **2026.10.14** `paranoid` is `true`/`false` **or a list of categories** (`MISE_PARANOID=config,env_files`):

```toml
# ~/.config/mise/config.toml
[settings]
paranoid = ["env_files", "plugins"]   # or true (= all four)
```

| Category | Effect |
|----------|--------|
| `config` | Every non-global file needs trust **bound to its content** (re-approve after every change), and **each file is trusted on its own** — trusting `mise.toml` doesn't trust `mise.local.toml`, task files, idiomatic version files, or a project-named aqua registry. No automatic, CI, `--yes`/`MISE_YES`, or worktree trust. A trusted monorepo root trusts only itself. Local `include` files are an error. |
| `env_files` | Every `_.file`, `_.source`, and `env_file` file needs content-bound approval (`mise trust .env`; re-run after edits). `mise trust` with no path trusts the config plus its env files. (A global single-file entry without a glob is exempt.) |
| `plugins` | Unregistered short-name plugins are refused (`--yes`/`--force` still allow them). |
| `provenance` | `locked_verify_provenance` is on; mise-versions "no attestations" answers are confirmed with GitHub. |

A remote [`include`](settings.md#remote-config-includes-include) must be pinned by full commit SHA (**40 or 64 hex digits**) or OCI digest.

### Trust

Outside CI, untrusted configs must be approved with `mise trust` (`--all`, `--ignore`, `--show`, `--untrust`; plus the separate `mise untrust`).

- mise **auto-trusts** configs when it detects a CI environment — unless paranoid (`config`) mode is on.
- **No trust needed:** a file with no template syntax containing only `min_version`, plain `[tools]` strings, and `[tasks]` — **unless** a task lists `secrets`, a tool key contains `[` (inline options, 2026.9.18; `.tool-versions` too since 2026.10.0, GHSA-wcqh-j26q-g44x), or a version is **`path:`** (2026.10.9). `.miserc.toml` never needs trust.
- 🔴 **An untrusted project config's whole `[settings]` table is ignored** (2026.10.13). Until its config is trusted, a project `mise.lock` only pins versions and checksums.
- **The prompt** offers Yes / No / All. **No adds the directory to a persistent ignore list** until you run `mise trust`. Without a terminal the command fails; shell activation never prompts (it warns and skips, and other trusted configs keep loading).
- **Implicit trust** (2026.8.9+): `mise run`, naked `mise <task>`, `mise install`, `mise exec`, `mise watch`, and `mise daemons start|restart|register` trust and persist the active config (and its parent configs). Since **2026.10.19** these do **not** count as consent: `--help`, a shim-launched `mise x`, a hook's `mise run`, an MCP call, and config writes by `mise set`/`unset`/`tasks add` (they prompt, or fail without a terminal). `mise use`/`set` writes trust that config root after asking about other untrusted files in it. A task is trusted only if **every** config contributing to it is trusted.
- **Where trust lives:** trust belongs to a config root and is recorded by **resolved path** (trust the real file behind a symlink). A `mise/`, `.mise/`, or `.config/` directory that is its own git checkout counts as a separate project. A linked **git worktree** is auto-trusted if the equivalent path in the main checkout is trusted (one-way; `--ignore` wins; not under paranoid) — worktrees must be registered at their current location (`git worktree repair` after moving one). `mise trust --all` walks subdirectories, respecting `.gitignore` and skipping hidden dirs, `node_modules`, `vendor`, `target`, `dist`, `build`.
- When a monorepo root is trusted, descendant configs are trusted **while the root stays trusted** (not in paranoid mode); after `--ignore` on a monorepo root, subprojects need their own trust (2026.10.19).
- `trusted_config_paths` overrides the ignore list and declined prompts (`["/"]` turns trust checks off); `ignored_config_paths` overrides `trusted_config_paths`. Trust-sensitive keys (`ci`, `paranoid`, `trusted_config_paths`, `yes`) only apply from global config, CLI flags, and env vars.
- Shell completion omits `complete run=`, `delegate=`, `choices run=`, and `mount` until the project is trusted (2026.10.19).
- **Limits (2026.10.19):** config, lock, and task files over **16 MiB** are refused; a project can't mark a task `global`; a relative `go.repo` resolves from `$HOME`.
