## Deprecation Calendar

**Contents**

- [Deprecation Calendar](#deprecation-calendar)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

Dates below come from mise's own `deprecated_at!` macros and `settings.toml` (`deprecated_warn_at` / `deprecated_remove_at`) at v2026.11.5 (every date re-verified) — the docs disagree with the source in places, and the source wins.

| Removal | What | Migrate to |
|---------|------|------------|
| **2026.12.0** | `shorthands_file` · `env.mise.*` namespace · `value`/`values` keys in `_.file`/`_.path`/`_.source` | `[plugins]` · `env._.*` · `path` (string or array) |
| **2026.12.0** (warn) | `[monorepo].lockfile` unset behavior; `aqua.registry_url`; `auto_env` platform-config warning | Set them explicitly; `aqua.registries` |
| **2027.1.0** | **`ubi:` backend** (warns since 2026.4.0) | `github:` (give each binary its own `[tool_alias]`) |
| **2027.2.0** | Flat `task_*` settings (`task_output`, `task_timeout`, `task_skip`, …) — **warnings live since 2026.8.0** | Dotted `task.*` |
| **2027.2.0** (warn) | Flat `task.cache_remote_*` settings | Nested `task.cache.remote_*` (removal 2027.8.0) |
| **2027.3.0** | Hook `script`/`scripts` spawned table form (warns since 2026.9.0); legacy `{version}` template syntax | `run`; `{{version}}` |
| **2027.3.3** | `[bootstrap].config_roots` and `mise bootstrap config-roots` (warns since **2026.9.3**, hidden from help) | `conf.d` folder fragments (the warning gives the exact path) |
| **2027.4.0** | `tera_v1` / `MISE_TERA_V1` and Tera v1 helpers (**warnings live since 2026.10.0**; includes `int`/`float` returning the default or 0 on a non-number and `first`/`last`/`nth` returning `""` on an empty list); top-level `env_file` / `dotenv` / `env_path` (warn since 2026.4.17); `mise b` alias. Also: Tera `unique` filter becomes **case-sensitive** | Tera v2; `_.file` / `_.path`; `mise backends` |
| **2027.4.0** (warn) | `auto_update` / `auto_update_check_duration` settings (`MISE_AUTO_UPDATE*`) — renamed in 2026.10.4 | `self_update.auto` / `self_update.check_duration` (removal 2028.4.0) |
| **2027.5.0** | Tera task-arg functions `{{arg()}}`, `{{option()}}`, `{{flag()}}` in run scripts; `mise github …` | `usage` spec + `$usage_*`; `mise token github` |
| **2027.7.0** | `[tools] python = { virtualenv = … }` tool option; `python.uv_venv_auto = true` (legacy value); `--local` / `-l` on `mise ls`, `outdated`, `upgrade`, `tasks ls` (warns from 2027.1.0; `settings --local` and `lock --local` unaffected) | `env._.python.venv`; `"source"` / `"create\|source"`; `--no-global` (or `--project` on `ls`) |
| **2027.8.0** | Automatic `all_compile = true` distro defaults on **NixOS** and **Alpine**; flat `task.cache_remote_*` | Enable `nix-ld`, or set `all_compile = true` explicitly; `task.cache.remote_*` |
| **2027.8.5** | `-l` as the `--bump` shorthand on `upgrade`/`outdated` | `-b` / `--bump` |
| **2027.8.10** | Dotted **unconditional** `conf.d` filenames (`node.tools.toml`) start acting as environment selectors (warns since 2026.8.10) | Hyphens (`node-tools.toml`), or opt in now with `env_conf_d = true` |
| **2027.8.14** | Task/`[task_config]` `rust_cache` — already a **no-op** | [mbx](https://mr-boxington.jdx.dev/getting-started) |
| **2027.9.0** | `mise generate bootstrap` | `mise generate install-script` |
| **2027.9.3** | Task `output = "quiet"` mode (warns since **2026.9.3**) | `output = "interleave"` + `task.quiet = true` |
| **2027.9.13** | `[dotfiles]` `exclude` patterns where `*` crosses `/` (warns since 2026.9.13) | `**` |
| **2027.10.0** | `install_before` (**warning live since 2026.10.0**); `lockfile_mode = "merge"` / `MISE_LOCKFILE_MODE=merge` (deprecated 2026.10.24, warns from 2027.4.0). `locked_require_checksum` **starts warning** here | `minimum_release_age`; delete the setting (default `generate`) |
| **2027.10.4** | Bootstrap part aliases `shell`, `defaults`, `launchd`, `systemd` in `--only`/`--skip`; commands `mise bootstrap launchd`, `mise bootstrap systemd`, `mise bootstrap macos-defaults`; `mise direnv` / `use mise` (warn since **2026.10.4**) | `mise-shell-activate`, `macos-defaults`, `macos-launchd-agents`, `linux-systemd-units`; `mise bootstrap macos launchd-agents`, `linux systemd-units`, `macos defaults`; `mise activate` + `[env]` |
| **2027.10.5** | No-op settings `task.cache.stats_report`, `sops.age_recipients`, `plugin_autoupdate_last_check_duration` (warn since **2026.10.5**) | Remove them (`--task-cache-stats`; `mise set --age-encrypt --age-recipient`) |
| **2027.11.0** | `credential_command` legacy single positional argument (the hostname arg); `*.default_packages_file`; `dotnet.package_flags`; **`always_keep_download`** (**warnings live since 2026.11.0**) | `MISE_CREDENTIAL_HOST`/`MISE_CREDENTIAL_PROVIDER`; tool-level `postinstall`; `prerelease` option; the download cache |
| **2027.12.0** | `experimental_monorepo_root` (warns since 2026.7.7); `aqua.registry_url` | `monorepo_root`; `aqua.registries` |
| **2028.4.0** | `auto_update` / `auto_update_check_duration` | `self_update.auto` / `self_update.check_duration` |
| **2028.5.0** | `mise tasks add` replacing a **different** existing task without `--force` is refused (warns from 2027.5.0; re-adding an identical task is a no-op) | `mise tasks add --force`, or `mise tasks set` |
| **2028.10.0** | `locked_require_checksum` setting (added 2026.11.0; warns from 2027.10.0) — locked mode will always require checksums | Remove the setting |

> **Not deprecated:** the top-level `mise dotfiles` / `mise dot` command. A 2026.7.16 note announced a deprecation (removal 2028.2.0), but it was **reversed in 2026.9.8** — re-verified on 2026.11.5 (no `deprecated_at!`, no warning; `mise bootstrap dotfiles` is now the hidden alias).

**Default flips ahead:** `auto_env` → `true` in **2027.6.0** (warns from 2026.12.0) · `[monorepo].lockfile` → root lockfiles in **2027.6.0** · `cargo.binstall_native` warns 2027.1.0 and defaults on **2027.7.0**.

**Recent default changes (2026.11.x):** `npm.provenance` → **on** (2026.11.1) · new lockfiles are **revision 4** (2026.11.1) · `--locked` **requires checksums** (`locked_require_checksum` unset ⇒ on, 2026.11.0) · `skills.dir` → **unset**, preferring `.agents/skills` (2026.11.0) · `task.history` → **on**, 20 runs per task (2026.11.3).

**Recent default changes (2026.10.x):** `node@lts` → **Node.js 26** (2026.10.22; pin `node@24` or `lts-krypton` to stay) · `lockfile_mode` → **`generate`** (2026.10.19) · `java.shorthand_vendor` → **`temurin`** (2026.10.5) · the 24h `minimum_release_age` default now covers **every backend** (2026.10.15) · `fetch_remote_versions_cache` unset ⇒ install/use/run reuse a list for 23h (2026.10.11) · Windows + `experimental = true` ⇒ **identity** install layout (2026.10.10) · `_.file` `find_up` starts at the **config root** (2026.10.16/17) · an untrusted project's `[settings]` is **ignored** (2026.10.13) · `--affected` counts **working-tree** changes (2026.10.6) · `pin` also affects `upgrade --bump` (2026.10.18) · Node GPG checks cover **all** versions (2026.10.5; a missing `.sig` fails since 2026.10.19) · an npm `allow_builds` list approves **only** the listed packages (2026.10.16).

**Earlier default changes:** `ruby.compile` now defaults to **precompiled binaries** (2026.8.0), and since **2026.8.2** `ruby.compile = false` is a *strict precompiled-only* mode — installs error with `no precompiled ruby found` instead of falling back to ruby-build, and version listings are filtered to versions with a precompiled binary for your platform (unset and `true` unchanged; Windows unaffected) · `windows_powershell_no_profile` → `true` (2026.7.13) · structured env-file values are **literal by default** again (2026.7.14) · `cargo.binstall_quickinstall` = `false`.

> **`minimum_release_age` has a built-in `24h` default** applied when the setting is unset — on **every backend** since 2026.10.15 (previously only timestamp-reporting ones). `mise settings get` reporting it "not set" and the schema carrying no `default` are both true and both misleading — `settings.toml` uses `default_docs = "24h"`, and the constant `DEFAULT_MINIMUM_RELEASE_AGE = "24h"` lives in mise's source. Use `"0s"` to actually disable it.

**Other breaking changes in the 2026.8.x line:**
- **2026.8.5** — `disable_tools = ["python"]` no longer activates a `_.python.venv`. The venv stays on disk and is restored when Python is re-enabled.
- **2026.8.9** — legacy `RTX_*` environment variables (incl. `RTX_TOOL_OPTS__*`, `RTX_ADD_PATH`) **removed** from asdf/vfox plugin hooks; use `MISE_*`. Standard `ASDF_*` remain.
- **2026.8.9** — `conf.d` fragments with an extra dot before `.toml` became environment-specific — **reverted in 2026.8.11**; now a deprecation (selectors by default in 2027.8.10). Use hyphens.
- **2026.8.9** — `vlang` configs pinning `2026.x`-style versions must move to a real upstream version (`0.5.2`, `weekly.*`).
- **2026.8.6** — `mise use --global` alongside a path is now rejected rather than silently ignored.

**Breaking changes in the 2026.9.13 – 2026.10.3 line:**
- **2026.9.13** — `pkgx:` backend removed · a TOML `[tasks.x]` with `run` now **replaces** a same-named file task (metadata-only blocks configure it) · locked tools stay on their locked backend when the registry moves them (`mise backends switch`) · tool-stub lock data moved into the project `mise.lock`.
- **2026.9.15** — `mise tasks validate` exits 1 on an unparseable `usage` spec.
- **2026.9.16** — new lockfiles are **revision 3** (unreadable by older mise) · SLSA provenance requires a matching signer identity or is skipped.
- **2026.9.17** — `mise self-update` and the installer wait for a 24h release age · paranoid mode ignores `--yes`/CI auto-confirm for trust.
- **2026.9.18** — tool keys with inline `[options]` require trust in `mise.toml`.
- **2026.10.0** — same for `.tool-versions` (GHSA-wcqh-j26q-g44x) · keyless cosign needs a pinned identity (vfox plugins must set `cosign_certificate_identity`) · aqua on musl hosts installs the registry-named asset unless `libc = "musl"` · Ctrl-C exits 130 everywhere · `--from-git` removed.
- **2026.10.1/10.2** — timeouts actually stop tasks, and a timed-out task fails even on exit 0.
- **2026.10.3** — dotenv files parsed by `mise-dotenv` (a file's own values beat ambient env) · `__MISE_DIFF`/`__MISE_SESSION` hold digests, not values.

**Breaking changes in the 2026.10.4 – 2026.10.19 line:**
- **2026.10.4** — `mise secrets` (and `installs`) became built-in commands — run tasks with those names via `mise run <name>` (likewise `vars` in 10.6, `lsp` in 10.18) · a failing tool `postinstall` fails the install.
- **2026.10.5** — Java shorthand defaults to **Temurin** · experimental gate **added** to `git::` task files, `git::`/`oci::` task includes, OTel export, and `spinel:` installs · `mise.local.toml` now really overrides `mise.<env>.toml` · `mise settings set/add` refuse writes that would have no effect · MCP `mise://tasks` `env` is an array · `mise watch` `-E`/`-q` now mean mise's flags (`--watchexec-env`/`--watchexec-quiet`) · task-name extension stripping applies to file tasks only · `mise run ./x` resolves from the cwd · `mise dot pull` takes no paths.
- **2026.10.6** — `--affected` also counts uncommitted/untracked changes (use `--affected-committed` for the old behavior).
- **2026.10.7** — `{ value, required = true }` env entries are a parse error.
- **2026.10.9** — `path:` versions need trust · `opencode@latest` is OpenCode 2 (`npm:@opencode/cli`; pin `opencode@1` to stay on v1) · `mise dot status` output format changed (scripts: `-v`/`--json`).
- **2026.10.13** — untrusted project `[settings]` are ignored · `[wrappers]` values are Tera templates.
- **2026.10.14** — unknown tool options **fail installs** · paranoid `env_files` needs content-bound trust · `#MISE` headers accept every task field (behavior change for previously-ignored `timeout`/`vars`/sandbox keys).
- **2026.10.16/17** — `_.file` `find_up` starts at the config root · an npm `allow_builds` list approves only the listed packages.
- **2026.10.18** — `unuse` prunes only the named version · `pin` affects `upgrade --bump`.
- **2026.10.19** — `lockfile_mode = "generate"` by default · trust prompts no longer implied by `--help`/shims/hooks/MCP/config writes · project `.miserc.toml` template limits · broader safe mode · Node `.sig` required · SLSA from branch builds no longer verifies · packslip pins by repo ID · GitHub tokens only to API hosts (`github.enterprise_hosts` for GHE) · credentials in URLs refused · Linux write-sandbox needs kernel ≥ 6.2 · archives with escaping symlinks refused · `enter`/`leave`/`watch_files` hooks run in the project root · 16 MiB config/lock/task file limit · `MISE_OVERRIDE_*_FILENAMES` match literally · relative `go.repo` resolves from `$HOME` · dynamic completion needs trust.

**Breaking changes in the 2026.10.20 – 2026.10.25 line:**
- **2026.10.20** — `system_packages.elevate_with` and `ruby.apply_patches` are global-only · a tool-option template that mixes `version` with `env`/`vars` renders at install time and no longer sees `env`/`vars` · bare `mise use <tool>` reuses the newest installed version.
- **2026.10.21** — `mise dot add` refuses a `mode = "template"` target without `--force`.
- **2026.10.22** — `node@lts` is Node.js 26 · `elevate_with = ""` means never elevate · remote git task sources reject an empty, `HEAD`, or `-`-prefixed `ref`.
- **2026.10.23** — project task grants can't use keys from your personal fnox config · `[wrappers]` skip only the running wrapper, so nested commands go through other wrappers · `mise uninstall ruby 3.4.1` matches only exact or `.`/`-`/`+` continuations.
- **2026.10.24** — `latest` for `gem:` skips RubyGems prereleases.

**Breaking changes in the 2026.11.0 – 2026.11.5 line:**
- **2026.11.0** — `go.mod` patch-less `go X.Y` and `CMakeLists.txt` `cmake_minimum_required` floors are **no longer read** (`idiomatic_version_file_ignore_minimum_versions` removed — now an unknown setting) · `--locked` fails on a URL entry without a checksum · `MISE_SAFE=1` ignores project lockfile URLs · a project `.miserc.toml` can't drop global/system environments · `skills.dir` default removed.
- **2026.11.1** — `npm:` installs verify provenance by default (an unprovenanced package installs unless the lock pins a repository; Sigstore must be reachable) · new lockfiles are revision 4 (`unsupported lockfile version 4` on older mise).
- **2026.11.2** — registry `temporal` installs the CLI (`aqua:temporalio/cli`); the server bundle is `aqua:temporalio/temporal`.
- **2026.11.3** — `github:` asset autodetection skips `.whl` and `.gem` assets.
- **usage 7.1.5 (bundled from 2026.11.x)** — a multi-target or repeated relationship property (`conflicts="--b --c"`) is a load-time error.

**Undated deprecations:** `[alias]` → `[tool_alias]` (now prints a warning) · `asdf_compat` (no longer supported) · `go.set_gopath` (set `GOPATH` in `[env]`) · `idiomatic_version_file` / `idiomatic_version_file_disable_tools` → `..._enable_tools` and `legacy_version_file*` → `idiomatic_version_file*` (these now have **no effect**) · `mise-env-fnox` plugin / `_.fnox-env` → `[secrets.fnox]` · `mise tasks run` (hidden) → `mise run` · `mise bootstrap dotfiles` (hidden) → `mise dot` · `npm.bun` → `npm.package_manager` · `profile`/`MISE_PROFILE` → `MISE_ENV` · monorepo automatic filesystem discovery → explicit `config_roots` · registry shorthands `localstack` → `lstk`, `actionlint` → `jactionlint`, `mprocs` → `dekit` (alias kept, 2026.10.5), `opencode` and `native-sdk` moved to npm (and `mbx` now means `mr-boxington`).

**Already removed:** `go.mod`/`CMakeLists.txt` minimum-version floors and `idiomatic_version_file_ignore_minimum_versions` (**2026.11.0**, warned since 2026.8.10; `toolchain goX.Y.Z` and an exact `go X.Y.Z` still work) · `pkgx:` backend (**2026.9.13**, no deprecation period — lockfile `[pkgx-packages]` sections still load and are dropped on the next write) · `mise bootstrap --from-git` / `mise bootstrap remote --from-git` (**2026.10.0**; use `--adopt`) · embedded `[lock]` sections in tool stubs (2026.9.13; ignored and removed — use `mise generate tool-stub --lock`) · `vars.mise` namespace · non-string `postinstall` hooks · unknown table fields in hook definitions · `mise oci push --tool` · project-local `*_default_*_shell_args` (2026.7.14) · `--monorepo` placeholder on `upgrade`/`outdated`/`prune` · `mise dot pull [PATH]…` positional (2026.10.5).

> **On the Tera task-arg removal date:** the docs now agree with the source — **2027.5.0** (`src/task/task_script_parser.rs` calls `deprecated_at!(…, "2027.5.0", …)`; the older "2026.11" figure in some docs is stale). Either way, do not use them — opt out early with `task.disable_spec_from_run_scripts = true`.
