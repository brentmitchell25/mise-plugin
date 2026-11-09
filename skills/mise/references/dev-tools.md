## Dev Tools Management

**Contents**

- [Backends Overview](#backends-overview)
- [TOML Syntax for Tools](#toml-syntax-for-tools)
- [Per-Tool Options](#per-tool-options)
- [Backend-Specific Configuration](#backend-specific-configuration)
- [Lazy Tools](#lazy-tools)
- [Tool Stubs](#tool-stubs)
- [Shims and Aliases](#shims-and-aliases)
- [Shell Completion (per-directory tab-complete)](#shell-completion-per-directory-tab-complete)
- [Packslip, Man Pages, and Agent Skills](#packslip-man-pages-and-agent-skills)
- [Lockfiles (mise.lock)](#lockfiles-miselock)
- [Auto-Install Controls](#auto-install-controls)
- [Automatic Tool Updates](#automatic-tool-updates)
- [Install Layout, Download Cache, and the global Link](#install-layout-download-cache-and-the-global-link)
- [CLI Commands for Tools](#cli-commands-for-tools)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### Backends Overview

mise supports **20 backends** plus custom backend plugins (`mise backends ls` on 2026.11.5: aqua, asdf, cargo, conda, core, dotnet, forgejo, gem, github, gitlab, go, http, npm, **packslip**, **pypi**, s3, **spinel**, spm, ubi, vfox — `pipx` is an alias of `pypi`, not listed). The count is unchanged from 2026.9.12 because **pkgx was removed (2026.9.13)** and **spinel was added (2026.10.2, experimental)**. The registry assigns tools to backends by an **acceptance-tier** preference order — prefer the highest tier available for a given tool:

| Tier | Backends | When chosen |
|------|----------|-------------|
| **1 — Preferred** | **packslip**, **aqua**, **github**, **gitlab** | packslip when the publisher ships signed release manifests (best provenance + version-matched completions/man pages/agent skills). aqua otherwise offers the most features + security (cosign/SLSA/attestation/minisign, native Windows, no plugin). github/gitlab for releases not yet in aqua. |
| **2 — High bar** | **conda** | Lower bar than tier 3 because mise's conda backend needs no separately-installed package manager. |
| **3 — Very high bar** | **pypi**, **npm**, **gem**, **go**, **cargo**, **dotnet** | Depend on a separately-installed runtime on PATH; silently bind tools to whichever runtime was available at install time. |
| **Other** | **forgejo**, **http**, **s3**, **spm**, **spinel** | forgejo (Codeberg default); http for direct URLs; s3 for private buckets; spm for Swift; spinel (experimental) compiles a Ruby CLI to a native binary. |
| **Not accepted for new registry entries** | **vfox**, **asdf**, **ubi** | vfox/asdf rejected for supply-chain reasons; ubi deprecated. |

> 🔴 **`pipx:` → `pypi:` (2026.9.7).** `pypi:` is the preferred name for the Python CLI backend. `pipx:` remains **fully supported with no warnings**, and settings accept both `pypi.*` and `pipx.*` spellings. But the two are **distinct tool identities** (`pypi-black` vs `pipx-black` install dirs and lock entries), so switching spelling creates a separate installation. mise preserves explicit `pipx:` names in output and lockfiles.

> 🔴 **`pkgx:` backend REMOVED (2026.9.13, breaking).** `"pkgx:…"` tool keys no longer resolve — switch to a registry shorthand or an `aqua:`/`github:` spec. Lockfile `[pkgx-packages]` sections still load and are dropped on the next write. (The `pkgx` *registry entry*, i.e. the pkgx CLI itself, still exists.)

| Backend | Status | Description |
|---------|--------|-------------|
| **packslip** | stable | Signed release manifests published by maintainers; signer pinning (SSH-style), plus version-matched completions, man pages, and agent skills. No Packslip CLI needed. |
| **aqua** | stable | Most features, best security (cosign/SLSA/attestation/minisign). No plugins needed. Native Windows. Registry compiled into the mise binary; the aqua CLI is never used. |
| **github** | stable | GitHub releases with auto OS/arch/libc asset detection |
| **gitlab** | stable | GitLab releases |
| **forgejo** | stable | Forgejo/Codeberg releases |
| **http** | stable | Direct HTTP/HTTPS downloads with URL templating |
| **s3** | stable | S3/MinIO/S3-compatible private buckets (AWS SDK credential chain) |
| **pypi** | stable | Python CLIs in isolated environments (uses uv by default, pipx fallback). `pipx:` is a supported alias with a **separate tool identity**. |
| **npm** | stable | Node packages via the embedded `aube` installer — **node/npm not required to install** |
| **go** | stable | Go packages (requires compilation) |
| **cargo** | stable | Rust packages (uses binstall by default) |
| **gem** | stable | Ruby gems |
| **conda** | stable | Single conda packages direct from anaconda.org (no conda/mamba needed) |
| **dotnet** | stable | .NET tools |
| **spm** | stable | Swift packages |
| **spinel** | **experimental** | Ruby CLI from a GitHub repo compiled to a native binary by Matz's Spinel compiler (2026.10.2); requires `experimental = true` |
| **ubi** | **DEPRECATED** (warns; removal 2027.1.0) | Universal Binary Installer — migrate to `github` |
| **vfox** | stable | **The recommended plugin system**; cross-platform, Windows-supported; default plugin backend on Windows |
| **asdf** | **legacy** | asdf plugins (no Windows; disabled by default on Windows) |
| **core** | stable | Built into the binary: bun, deno, elixir, erlang, go, java, node, python, ruby, rust, swift, zig |

**Backend capability matrix:**

| Feature | Core | Lang PMs | aqua | github | Backend Plugins | Tool Plugins | asdf |
|---------|------|----------|------|--------|-----------------|--------------|------|
| Speed | ✅ | ⚠️ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| Security | ✅ | ⚠️ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Windows | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Env Vars | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Custom Scripts | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |

**Backend selection for a short name** (an explicit spec like `aqua:owner/repo` bypasses all of this), in order:
1. `MISE_BACKENDS_<TOOL>` env override
2. a `[tool_alias]` that names a backend
3. a `[plugins]` URL
4. the backend recorded in **`mise.lock`**, if it can serve the requested version
5. an installed plugin of the same name (unless its backend is disabled)
6. the first **registry** backend that runs on this platform, isn't in `disable_backends`, and supports the version (experimental backends count only with `experimental = true`)

**Version-range backends:** a registry entry can switch backend by version (`max_version` is exclusive) — e.g. `hk` installs from aqua below 1.58.1 and from packslip at 1.58.1+. Boundaries apply only to short names; a prefix spanning the boundary (`hk@1.58`) uses the normal order. Since 2026.10.13 a registry backend can carry different **options** per version range, and `ls-remote` lists every range. `disable_backends` replaces the whole list; installing an explicit identifier with a disabled backend fails; asdf is never used on Windows.

**Locked backend wins over registry moves (2026.9.13, breaking).** When the registry moves a tool to a new backend, a locked tool keeps installing from the old one and mise warns `` `<tool>` is locked to `<old>`, but the registry now installs it from `<new>`. Run `mise backends switch <tool>`. `` Switch explicitly:

```bash
mise backends switch            # every configured tool whose locked backend was replaced
mise backends switch hk@1.58.1  # one tool/version
mise backends switch -n         # dry run;  -g = the global config's lockfile
```
`switch` moves the lock entries to the new backend at the **same versions**, relocks every platform (new URLs/checksums), and reinstalls; if any relock fails, every changed lockfile is restored.

**Registry override** per tool (SHOUTY_SNAKE_CASE; `my-tool` → `MISE_BACKENDS_MY_TOOL`):
```bash
export MISE_BACKENDS_PHP='vfox:jdx/vfox-php'
mise install php@latest
```

**Disable backends** (affects new installs only; does not uninstall existing):
```bash
mise settings disable_backends=asdf
```

**Discovery:**
```bash
mise registry                    # list everything
mise registry --backend aqua     # filter by backend
mise registry --json --security  # per-backend security info (slower)
mise registry --hide-aliased
mise use                         # interactive selector
mise search <query>              # -m equal|contains|fuzzy (default fuzzy)
mise search npm:typescript-lang  # 2026.9.13: prefix npm:/cargo:/gem:/dotnet: to search that package registry
mise search -a <query>           # --all: registry, aqua, installed backend plugins, AND npm/cargo/gem/dotnet
mise tool <name> --url           # registry project URL (2026.9.13)
```

> **mise-versions serves version lists for every backend (2026.10.15).** Registry tools on their registry backend — including **npm, pipx/pypi, cargo, gem, go, dotnet, spm** — and `github:`/`aqua:`/`packslip:` tools for any public GitHub repo (2026.9.14) get version lists, release lookups, and attestation lookups from mise-versions (avoiding rate limits; `mise outdated`/`upgrade` read one index for many tools, 2026.10.11). It is **skipped when the source is redirected**: `.npmrc`/`NPM_CONFIG_REGISTRY`, `pypi.registry_url`, `cargo.registry_name`, a non-default `GOPROXY`/`GOPRIVATE`/`GONOPROXY`, a non-rubygems gem source, `dotnet.registry_url`, `AWS_ENDPOINT_URL`, `url_replacements` rerouting the GitHub API, or a tool option that changes the listing; `conda:` always lists from its channel. It serves only the **newest 1,000 releases** of a repo. mise treats it as an untrusted mirror (URLs must match repo/tag/asset); on timeout, lost connection, 429, or 5xx it falls back to api.github.com (with token), then a cached copy — a 404 or empty answer is trusted. Paranoid mode re-checks "no attestations" answers against GitHub. Bypass with `MISE_USE_VERSIONS_HOST=0` or `mise ls-remote --no-versions-host`.

> **Registry default-backend changes (2026.9.6):** `postgres`, `redis`, and `mongodb` shorthands now prefer **`conda:`**. `mise use <tool>` therefore resolves differently than in earlier versions — pin the backend explicitly (`aqua:…`, `github:…`) if you depend on the old resolution.

**Floating registries** (`registry_floating = true`, default `false`) fetch the shorthand registry published with the latest mise release plus the current official aqua registry, instead of the release-pinned snapshots. Bundled snapshots remain the fallback, and fast/offline commands never refresh. Opt-in, because a floating registry may contain changes made after your installed mise was tested.

### TOML Syntax for Tools

```toml
[tools]
# Simple version
node = "22"
python = "3.12"
ruby = "latest"

# Multiple versions
python = ["3.12", "3.11"]

# With options
node = { version = "22", postinstall = "corepack enable" }
python = { version = "3.11", os = ["linux", "macos"] }
hk = { version = "latest", os = ["linux", "macos/arm64"] }  # OS/arch combos

# Explicit backend
"aqua:BurntSushi/ripgrep" = "latest"
"github:cli/cli" = "latest"
"npm:prettier" = "latest"
"pypi:psf/black" = "latest"
"cargo:eza" = "latest"
"go:github.com/DarthSim/hivemind" = "latest"

# Table form (best for nested options)
[tools."http:my-tool"]
version = "1.0.0"
platforms.macos-x64.url = "https://example.com/my-tool-macos-x64.tar.gz"

# Arbitrary nested options work for any backend
[tools."custom:my-backend".cache.redis]
host = "redis.example.com"
port = 6379
```

Nested options are internally flattened to dot notation (`platforms.macos-x64.url`, `cache.redis.port`). mise reads **TOML 1.1**, so inline tables may span lines with a trailing comma (`mise fmt` keeps them intact since 2026.10.15).

With `python = ["3.14", "3.13"]` the **first** version provides the unversioned command; the other is reached as `python3.13`.

CLI tool-option syntax uses brackets: `mise use "conda:ruff[channel=bioconda]"`, `mise use "github:oxc-project/oxc[matching=oxlint,rename_exe=oxlint]@apps_v1.69.0"`.

**Version formats:**

| Format | Example | Description |
|--------|---------|-------------|
| Exact | `"20.0.0"` | Specific version |
| Prefix | `"20"` | Latest matching prefix |
| Latest | `"latest"` | Most recent stable |
| `lts` | `"lts"` | Active LTS release (node): **26** since 2026.10.22 (`lts-lithium`); older lines `lts-iron`, `lts-jod`, `lts-krypton` (24). Pin `node@24` or `lts-krypton` to stay on 24 |
| `prefix:<P>` | `"prefix:1.19"` | Latest matching prefix (explicit form — needed where a bare `1.20` would be exact, e.g. Go) |
| `ref:<SHA>` | `"ref:master"` | Compile from git ref (npm git sources and spm too; **not** cargo, python, or erlang) |
| `path:<PATH>` | `"path:./shfmt"` | Use a custom install dir. Relative paths resolve from the config root. **Requires trust since 2026.10.9** (in `mise.toml`, `.tool-versions`, and idiomatic files); ignored in safe mode. On Windows use TOML literal strings (`{ path = 'C:\tools\node' }` — `"C:\tools"` silently becomes a tab); paths containing `& \| < > ^ %` are rejected |
| `system` | `"system"` | No mise install — mise adds nothing to PATH, so whatever is already on PATH runs |
| `sub-<PARTIAL>:<BASE>` | `"sub-2:lts"` | **Numeric subtraction**, not "Nth previous release": resolve BASE, subtract PARTIAL's components, resolve as a prefix. `sub-2:lts` → 26 becomes 24; `sub-0.1:latest` → 3.11 becomes 3.10. |
| `tag:<TAG>` | `"tag:v1.0.0"` | Cargo git tag |
| `branch:<BRANCH>` | `"branch:main"` | Cargo git branch |
| `rev:<SHA>` | `"rev:abc1234"` | Cargo git rev |

> **Resolution nuance:** in config files, `node@20` means the latest *installed* 20.x. But `mise install node@20`, `mise latest node@20`, and `mise upgrade node@20` resolve to the latest *available* 20.x. A prefix matches at separators — `1.2` matches `1.2.3` and `1.2+build` but **not** `1.20`. npm-style ranges (`^24`, `>=20`) warn and are then treated as a literal version (which fails to install). `mise use node` with no version writes `latest` but reuses the newest **installed** version (2026.10.20); write `mise use node@latest` to install the newest release. Cargo git sources accept `tag:`/`branch:`/`rev:` but **not** `ref:`.

**Structured tool selectors (2026.7.11+)** are uniform across root `[tools]`, inline task defs, task templates, and file-task headers. Exactly one selector is required; conflicting, missing, or non-string selectors are a hard config error:

```toml
node       = { version = "20" }
go         = { prefix = "1.22" }
python     = { ref = "main" }
shellcheck = { path = "/opt/shellcheck" }
```

Tool versions may now contain a colon (2026.8.1+), so templated versions like `{{ exec(...) | split(pat=': ') | last }}` no longer break config loading.

### Per-Tool Options

Universal options supported by every backend:

| Option | Type | Description |
|--------|------|-------------|
| `version` / `prefix` / `ref` / `path` | string | The selector — exactly one required in table form |
| `os` | string \| string[] | Restrict to OS/arch: `"linux"`, `"macos"`/`"darwin"`, `"windows"`/`"win"`, and **`"unix"`** (2026.9.12+ — every non-Windows platform). Combos: `"linux/x64"`, `"macos/arm64"`. Arches: `arm64`/`aarch64`, `x64`/`x86_64`/`amd64`. A bare OS matches any arch; an entry with `/` requires both to match. A **concrete OS variant wins over a `unix` one**. Non-matching ⇒ mise skips installing **and using** the tool. The same selector syntax applies in `[bootstrap.packages]`, `[doctor.checks]`, and `[dotfiles]` variants. |
| `install_env` | table | Environment variables injected during install (and tool-level `postinstall`). mise also reads GitHub/GitLab/Forgejo token vars from it, but its own HTTP client does **not** read proxy vars from it. |
| `postinstall` | string \| `{ run, when? }` | Command after successful install. Gets `MISE_TOOL_NAME`, `MISE_TOOL_VERSION`, `MISE_TOOL_INSTALL_PATH`, `MISE_CONFIG_FILE`, `MISE_CONFIG_ROOT`, `MISE_PROJECT_ROOT`, the project `[env]`, and `{{ tools.X.path }}` templates; the tool's bin dir is on PATH. Never runs after a failed install, and a **failing postinstall fails the install** (the version stays incomplete, 2026.10.4). **Table form (2026.9.17):** `postinstall = { run = "corepack enable", when = "always" }` — `when = "install"` (default) runs only on a fresh install or repair; `"always"` runs on **every** `mise install` **or `mise use`** that selects the tool, even when already installed (skipped on dry runs). `run` is required; other keys and invalid `when` values are parse errors. Refused in safe mode. |
| `auto_update` | bool \| duration | **2026.10.4/10.7. Global (and system) config only.** Keep this tool updated within its request: `true` uses `tool_update.check_duration` (default `24h`); a duration (`"6h"`, min 1h) sets its own interval; `false` opts out of `tool_update.global_auto`. See [Automatic Tool Updates](#automatic-tool-updates). |
| `depends` | string \| string[] | Install-graph ordering for tools in the current install set. For vfox, `[tools].depends` is merged with the plugin's `PLUGIN.depends` into one install-dependency context — it affects ordering, the PATH seen by `os.execute`/`cmd.exec` in hooks (not `io.popen`), and `tools = true` env values. The asdf backend also puts `depends` tools on PATH for `bin/download`/`bin/install`, and Ruby source builds see them (2026.10.3). An unconfigured dependency may be satisfied from the system PATH. |
| `version_order` | `"source"` \| `"semver"` | **2026.8.4+.** Make `latest` and prefix resolution follow semantic precedence rather than source/chronological order. Supported on **aqua, github, gitlab, forgejo, http**. Fixes releases where a backport line outranked a newer version (neo4j, victoria-metrics, talosctl, tealdeer…). With `"semver"`, `mise ls-remote` also lists in semver order (packslip always orders by semver). |
| `lazy` | bool | **2026.9.0+.** Defer installation until one of the tool's bootstrap shims is invoked. See [Lazy Tools](#lazy-tools). Default `false`. |
| `lazy_bins` | string \| string[] | **2026.9.0+.** Command names (no `/` or `\`) for lazy tools whose backend has no registry `bins` metadata. Single string accepted since 2026.10.2. |
| `minimum_release_age` | string | Per-tool override of the supply-chain delay (e.g. `"1d"`, `"0s"` to disable). A **built-in 24h default applies to every backend when unset** (2026.10.15) — see [Lockfiles](#lockfiles-miselock). |
| `install_before` | string | **Deprecated** — maps to `minimum_release_age` (**warning live since 2026.10.0**, removed 2027.10.0). |
| `prerelease` | bool | Include prereleases — honored by **every backend except core `java`**. With `latest`, the newest installed version **including prereleases** is what runs (`mise x`, PATH, `which`, `where`, `bin-paths`; 2026.11.0) |
| `platforms` / `platform` | table | Per-`<os>-<arch>` overrides (`platforms.linux-x64.url`); `platform` is an alias |

> **Typed tool options in the JSON schema (2026.10.2).** Editors using `https://mise.jdx.dev/schema/mise.json` now validate and autocomplete options per backend prefix (github, gitlab, forgejo, ubi, http, s3, aqua, cargo, npm, pypi/pipx, gem, go, conda, spm, packslip, spinel, core python/java/rust/dotnet, `platforms.<os>-<arch>`, and `[tasks.*.tools]`) — existing configs with typos may start showing editor errors. Boolean options accept `true`/`false`, `"true"`/`"false"`, or `1`/`0`.

> 🔴 **Unknown tool options now FAIL the install (2026.10.14, breaking).** Backends reject options they don't read, including keys under `platforms`:
> ```text
> option `asset_pattern` is not used by the cargo backend
> hint: Remove it, or check the spelling against the backend's documented options
> ```
> Checked: cargo, conda, gem, github, gitlab, forgejo, go, http, npm, packslip, pipx/pypi, s3, spinel, spm, ubi. **Not** checked: aqua, core tools, and plugins — vfox passes unknown options to the plugin (`ctx.options`), asdf exposes them as `MISE_TOOL_OPTS__<KEY>`. `mise config explain` and `mise lsp` flag typos before install.

> 🔴 **Inline options require trust (security, 2026.9.18 / 2026.10.0).** Any tool **key** containing `[` — `"github:cli/cli[api_url=https://ghe.example.com/api/v3]" = "latest"` — makes a `mise.toml` require `mise trust`, and since 2026.10.0 the same applies to `.tool-versions` entries (GHSA-wcqh-j26q-g44x). Inline options can redirect downloads, so they are no longer treated as "safe" config. A tool **table** with any option (`jq = { version = "1.7.1", os = ["macos"] }`) needs trust as well — an untrusted `mise.toml` loads only plain version strings (or arrays of them), `min_version`, and secret-less `[tasks]`.

> `github_attestations` is **not** universal — it is a `github:` backend tool option. The separate *global* `github_attestations` setting (default `true`) applies to supported tools.

**Core-tool options** (typed in the schema):

| Tool | Option | Type / default | Notes |
|------|--------|----------------|-------|
| `rust` | `profile` | string | rustup profile (`minimal`, `default`, `complete`) |
| `rust` | `components`, `targets` | string \| string[] | rustup components / cross targets |
| `rust` | `mr_boxington` | bool, `false` | Wrap cargo with mbx (needs `mr-boxington` in `[tools]`): `mise use --tool-option mr_boxington=true rust mr-boxington` |
| `python` | `patch_sysconfig` | bool, `true` (unix) | Patch sysconfig of precompiled builds |
| `python` | `virtualenv` | string | **Deprecated** — use `env._.python.venv` |
| `java` | `release_type` | `ga` (default) \| `ea` | Early-access builds |
| `dotnet` | `runtime` | `dotnet` \| `aspnetcore` \| `windowsdesktop` | Install a shared runtime instead of the SDK. `dotnet@lts` / `dotnet@sts` aliases (2026.10.15) |

> 🔴 **Java defaults to Temurin (2026.10.5, breaking).** `java.shorthand_vendor` changed `openjdk` → **`temurin`**, so `java@21` now installs Eclipse Temurin. Temurin has no builds for Java 9, 10, or 12–15 — request those as `openjdk-12` etc. Lock entries record the vendor, so an existing locked shorthand entry **fails `--locked`**: either keep `java.shorthand_vendor = "openjdk"` or run `mise lock --bump java` to relock onto Temurin. Explicit vendors (`java = "temurin-21"`, `"openjdk-21"`, `"zulu-21"`, `"corretto-21"`) are unaffected.

Example with dependencies:
```toml
[tools]
python = "3.12.11"
"pypi:ruff" = { version = "latest", depends = ["python"] }
"cargo:usage-cli" = {
    version = "latest",
    os = ["linux", "macos"],
    install_env = { RUST_BACKTRACE = "1" }
}
```

Tool option values support template expansion referencing `env.*` and `vars.*` (including values produced by `_.source`/`_.file`); nested option arrays/tables are Tera-rendered too (2026.7.7+). Tera filters on `version` work (2026.10.20): `url = "https://example.com/tool-{{ version | replace(from='-build1', to='') }}.tar.gz"`. Keep `env`/`vars` and `version` in **separate** `{{ }}` blocks — an expression that mixes them is rendered at install time, where `env`/`vars` render empty without an error (`{{ env.D ~ version }}` loses `env.D`).

### Backend-Specific Configuration

#### Packslip Backend (Preferred where available)

Installs from **signed release manifests published by maintainers**. mise verifies the release, picks the build for your platform, and installs its executables — plus any completions, man pages, and agent skills the release declares. No Packslip CLI needed.

```toml
[tools]
"packslip:github.com/jdx/hk" = "latest"
"packslip:jdx/hk" = "latest"              # GitHub host may be omitted
"packslip:tool.example.com" = { version = "latest", pubkey = "/path/to/vendor.pub" }
```

| Identifier | Source |
|------------|--------|
| `packslip:github.com/owner/repo` | A GitHub repository's releases |
| `packslip:github.com/owner/repo/tools/mytool` | One tool in a GitHub monorepo |
| `packslip:tool.example.com` | A signed release list hosted by the publisher |
| `packslip:example.com/tools/mytool` | One tool on a publisher's domain |

The project **must publish Packslip manifests** — this backend never infers install instructions from arbitrary release filenames. GitHub projects get built-in release discovery and signer identity rules; other hosts need a signed release list plus explicit signer configuration.

**Tool options:**

| Option | Default | Purpose |
|--------|---------|---------|
| `variant` | none | Select a publisher-declared alternative build (`fips`, `baseline`). Without it, only artifacts with **no** variant are considered. |
| `workflow` | registry default | **2026.10.5.** Accept only signatures from these GitHub workflow(s), run on a tag (or the stated ref): `workflow = ["release-plz.yml", "release.yml"]` or `"release.yml@refs/heads/main"`. A list lets the signer pin move between the named workflows. Not combinable with `pubkey`/`identity`/`identity_prefix`/`issuer` (any of those replaces a registry default). Registry packslip entries are now workflow-pinned. |
| `pubkey` | unset | Pin a minisign-format public key; a value that parses as a key is used as the key, otherwise it's a path to a `.pub` file |
| `identity` / `identity_prefix` / `issuer` | derived from a recognized forge | Expected keyless signer + OIDC issuer. Keep the trailing slash in a repository prefix. |
| `list_identity_prefix` | release signer policy | Pin a different workflow for the **vendor release list** only. Requires `issuer`; cannot combine with `pubkey`. |
| `prerelease` | `false` | Include prerelease versions |
| `trust` | configured stampers | `"vendor"` exempts this tool from stamp requirements |
| `allow_unlogged` | `false` | Accept key-signed bundles without transparency-log evidence |
| `ignore_requirements` | `false` | Install despite confirmed host requirement failures (keeps the GNU build instead of falling back to musl) |

```toml
"packslip:github.com/aubepkg/aube" = { version = "latest", workflow = ["release-plz.yml", "release.yml"] }
```

`minimum_release_age` uses the **verified transparency-log time**; a logged entry with no verified time counts as too new. Private repos retry the download through the API with your token. Packslip pins are checked by **repository ID** (10.19) — after a rename run `mise packslip forget <tool>`. Whether a GitHub repo publishes `.well-known/packslip.json` is looked up through mise-versions first (2026.10.21), falling back to GitHub for private repos, `use_versions_host = false`, or a `url_replacements` rule; the result is verified either way.

**Settings:** `packslip.exec` (default `false` — run a tool's own command at install time to produce a resource its packslip offers only as an `exec` entry; does **not** disable on-demand completion generation), `packslip.stampers` (hosts whose signed stamp lists say which releases may be installed, each with its pin).

**Version selection.** Only releases with Packslip manifests are visible. Semantic versions, including compatible date versions like `2026.9.1`. Prereleases excluded unless opted in. `latest` follows the publisher's `latest` pointer in a signed release list → GitHub's latest release → the highest eligible semver; a publisher may recommend an older supported release over a newer major.

> **The 24-hour `minimum_release_age` applies here**, so a just-published release may be absent until it cools. That 24h is mise's **built-in default for every backend** whenever the setting is unset (packslip uses the verified transparency-log time) — see [Lockfiles](#lockfiles-miselock).

**Private GitHub repos** work with the same credentials as `github:` (`MISE_GITHUB_TOKEN`, `GITHUB_API_TOKEN`, `GITHUB_TOKEN`) and need nothing in config. The token is transport-only: signature, project identity, signer continuity, and digest checks are unchanged, and no credential is read from the signed manifest.

**Signer pinning.** mise remembers which signer it accepted, the way SSH remembers hosts. A later release from another signer, a weaker scheme, a repackager where the vendor signed before, or one that drops build provenance is **refused until a person says so**:

```bash
mise packslip pins [--json]              # list accepted signers
mise packslip forget github.com/jdx/hk   # reset the pin so the next accepted release sets it again
```

`forget` also resets vendor release-list continuity. It does **not** change explicit signer options, erase stamper-list state, or remove a signer commitment from `mise.lock` — configure the new signer policy first, and if a lockfile entry conflicts, remove and regenerate it with `mise install`.

mise retains the verified manifest as `.mise-packslip.json` in the install directory.

> Verification authenticates the **publisher and bytes**, not that the software is safe. mise records whether build provenance links are present but does not fetch and verify that linked provenance.

#### PyPI Backend (formerly pipx)

```toml
[tools]
python = "3.14"
uv = "latest"
"pypi:black" = "latest"
"pypi:harlequin" = { version = "latest", extras = ["postgres", "s3"] }
"pypi:azure-cli" = { version = "latest", with = ["pip"] }
"pypi:ansible" = { version = "latest", expose = ["ansible-core"] }
"pypi:ansible" = { version = "latest", uvx = false, expose = [], pipx_args = "--include-deps" }
"pypi:black" = { version = "latest", python = "3.12" }   # 2026.10.12: per-tool private CPython
```

mise uses **uv**: with dependency locking it runs `uv sync --frozen`; version-only installs use `uv tool install` (passing `--python <mise python>` when mise manages Python, 2026.10.4 — a package whose `requires-python` excludes it now fails; override with `--python` in `uvx_args`), falling back to `pipx install` when uv is unavailable. With **no Python interpreter at all** (2026.11.0) mise warns and installs from the version alone; `--locked` installs and tools with a `python = …` option still fail.

> **`pipx:` is the same backend under its old name** (the `pipx.html` doc page is now just a redirect to `pypi.html#compatibility-with-pipx`). It keeps its **own tool identity** (`pipx-black` install dir, lock sidecars, and entries), and `pipx.registry_url`/`pipx.uvx` are hidden "older name for `pypi.*`" settings.

**Package sources:** `pypi:black` (PyPI) · `pypi:psf/black` (GitHub) · `pypi:git+https://github.com/psf/black.git[@main]` (Git; the `.git` suffix is optional since 2026.9.15) · `pypi:git+ssh://git@github.com/org/repo` (2026.9.14). For **GitHub** sources `latest` is the **newest GitHub release**, falling back to the default branch only when there are no releases (2026.9.15); for other Git URLs `latest` resolves default-branch HEAD to a concrete commit. `latest` skips PEP 440 `.devN` releases (2026.9.14). Direct HTTPS archive URLs are unsupported.

**Monorepo subdirectories (2026.9.15)** — each subdirectory is its own tool:
```bash
mise use 'pypi:git+https://github.com/runpantheon/ltui#subdirectory=ltui@main'
mise use 'pypi:runpantheon/ltui#subdirectory=jtui@main'
```
Use the `extras` option (not `#egg=`) and set `package_name` if the guessed distribution name is wrong.

**Tool options:**

| Option | Notes |
|--------|-------|
| `extras` | String or array of Python package extras. Works with Git sources. Inline: `mise use 'pypi:psf/black[extras=jupyter]@latest'` |
| `package_name` | For Git repos whose name differs from the Python distribution name |
| `with` | Extra requirements installed **into** the tool environment. Requires uv; participates in locking. |
| `expose` | Like `with`, but also exposes their entry points. Requires **uv ≥ 0.8.5**; participates in locking. |
| `dependency_prereleases` | uv prerelease policy: `disallow` \| `allow` \| `if-necessary` \| `explicit` |
| `registry_url` | Per-tool index (include a `{}` placeholder for the package name); overrides `pypi.registry_url`. Covers listing and locking only — version-only installs need the index in `uvx_args`/`pipx_args`. A URL with credentials or a query string can't be recorded in the lockfile |
| `registry_index` | **2026.10.24.** How dependency locking uses `registry_url`: `"default"` (the registry replaces PyPI — the tool and every dependency must come from it) or `"priority"` (registry first, then `pypi.registry_url`). Each project comes only from the first index that hosts it. In `priority` mode locking ignores `UV_EXTRA_INDEX_URL`, `UV_INDEX_URL`, and `install_env`. Affects locking only — use it for Gitea/Forgejo/pypiserver indexes that host only your packages: `"pypi:my-tool" = { version = "latest", registry_url = "https://gitea.example/api/packages/org/pypi/simple/{}/", registry_index = "priority" }` |
| `python` | **2026.10.12.** A CPython minor or exact patch installed as a **private** dependency of the tool and recorded as a `python` entry in the same lockfile scope. Works for uv and pipx installs; can't be combined with `--python`/`-p` in `uvx_args`/`pipx_args` |
| `uvx` | `false` selects the pipx installer (version-only locking). `uvx = true` cannot override `pypi.uvx = false` |
| `uvx_args` / `pipx_args` | Free-form installer args. **Version-only installs only** — `mise lock <tool>` naming the tool errors; a bare `mise lock` sweep warns and writes a version-only entry (2026.10.21). |
| `install_env` | Env during install |

> A `with`/`expose` requirement pinned to an exact version **narrows the locked Python range** to what that release supports, so the tool may need a newer interpreter than it declares. Guard with a marker: `"legacy==1.0.0; python_version < '3.12'"`.

**Settings:** `pypi.registry_url`, `pypi.uvx` (`pipx.registry_url` / `pipx.uvx` are compatibility aliases), **`pypi.keyring_provider`** (`disabled` default \| `subprocess`, env `MISE_PYPI_KEYRING_PROVIDER`, 2026.10.16 — when the index URL names a user but no password, mise runs `keyring get <url> <user>` for listing/locking and sets `UV_KEYRING_PROVIDER=subprocess`). The legacy names `uvx`/`uvx_args` control **uv** installation — they do not mean mise runs the `uvx` command.

Honors `minimum_release_age` via uv `--exclude-newer` (needs uv ≥ 0.2.22) or pip `--uploaded-prior-to`; PEP 691 JSON indexes give release dates on private indexes (2026.10.9). Simple-only indexes need `data-requires-python`. **`mise upgrade python` reinstalls every PyPI tool automatically** (2026.10.12), and on Unix version-only installs survive patch upgrades via the minor-version path — only graph-locked installs, or minor changes made outside `mise upgrade`, need `mise install --force pypi:black`.

#### Aqua Backend (Preferred)

```toml
[tools]
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:cli/cli" = { version = "latest", symlink_bins = true }  # Filtered .mise-bins directory
"aqua:flutter/flutter" = { version = "3.32.8", channel = "stable" }
"aqua:scenarigo/scenarigo" = { version = "0.21.0", vars = { go_version = "1.24" } }
```

**Tool options:** `symlink_bins` (bool — filters bundled bins into `.mise-bins`, using the registry's `files` field when defined; solves e.g. `aws-cli` bundling its own Python), `vars` (table, aqua registry template variables — may also be given at top level, e.g. `channel = "stable"`), `channel`, `prerelease` (bool — for `github_tag` packages `ls-remote` lists every tag, but `latest`/prefix skip prerelease-looking tags unless `prerelease = true`; drafts always excluded), and:

| Option | Since | Notes |
|--------|-------|-------|
| `libc` | 2026.9.16 | `"glibc"` \| `"gnu"` \| `"musl"` — **strict** asset selection on Linux (no fallback to the other libc). Overrides the global `libc` setting, but not a musl host or a `linux-*-musl` lock platform. Recorded in the lockfile; already-installed versions keep their build until `mise install --force`. A registry template var literally named `libc` must be passed as `vars.libc`. |
| `slsa_signer_identity` + `slsa_signer_issuer` | 2026.10.0 | Replace the registry's SLSA signer (both required, non-empty; identity supports aqua's `{{.Version}}`). Don't enable SLSA where the registry has no `slsa_provenance`. |

```toml
[tools]
"aqua:domcyrus/rustnet" = { version = "latest", libc = "musl" }
# signer = the BUILDER workflow in the certificate (here the SLSA generator), not the repo's own workflow
"aqua:google/osv-scanner" = { version = "2.6.0", slsa_signer_identity = "https://github.com/slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@refs/tags/v2.1.0", slsa_signer_issuer = "https://token.actions.githubusercontent.com" }
```
Look the signer up in the project's release provenance rather than copying it from an unreviewed source.

> 🔴 **musl hosts changed (2026.10.0, breaking).** On Alpine/musl with no explicit request, aqua now installs **whatever asset the registry names** — a gnu build then needs glibc/gcompat. Set `libc = "musl"` per tool to keep musl builds.

**Security (all default `true`):**

| Setting | Default | Description |
|---------|---------|-------------|
| `aqua.cosign` | `true` | Verify cosign signatures |
| `aqua.slsa` | `true` | Verify SLSA provenance |
| `aqua.github_attestations` | `true` | Verify GitHub Artifact Attestations |
| `aqua.minisign` | `true` | Verify minisign signatures |
| `aqua.baked_registry` | `true` | Use built-in (compiled-in) aqua registry |
| `aqua.registries` | none | Extra registry sources (string[], `MISE_AQUA_REGISTRIES`), loaded before the baked-in registry |
| `aqua.registry_url` | none | **Deprecated** — warns 2026.12.0, removed 2027.12.0. Use `aqua.registries`. |
| `aqua.registry_cache_ttl` | `1w` | Registry source freshness TTL (`0s` = always refresh) |
| `aqua.cosign_extra_args` | none | Additional cosign arguments (string[]) |

`aqua.registries` accepts a repository URL, a direct `registry.yaml`/`.yml` URL, or an absolute `file://` directory (local sources bypass the download cache, so edits are read on the next load). Packages resolve by checking configured registries in order, then the baked registry.

> **Aqua registry aliases are local to the registry that defines them** — use `[tool_alias]` to point a mise shorthand at a package from another registry.

`aqua.registries` also accepts a `file://` registry **file** or a plain path relative to the config root (`registry.yml` is a fallback name); `MISE_AQUA_REGISTRIES` is comma-separated and must use absolute `file://`; `aqua.registry_url` is ignored when `aqua.registries` is set. Under paranoid mode a project-named aqua registry needs its own trust (2026.10.19).

**Other aqua behavior:** recipes that build from source (`cargo`, `go_install`, `go_build`) **fail** and point you to the cargo/go backend. On glibc Linux mise picks the glibc build even when the recipe names musl, and `ls-remote` lists versions for the host libc. Universal macOS archives are found (2026.10.15). SLSA also checks `slsa_provenance.source_uri` and `source_tag` (`source_tag = "-"` disables the tag check).

Aqua verification is native Rust (no cosign/slsa-verifier/gh CLIs) covering GitHub attestations, cosign, SLSA, minisign, and SHA256/512/1/MD5 checksums (always on). Verification failure aborts the install.

**Stricter signer checks (2026.9.16 – 2026.10.0, security):**
- **SLSA** passes only with a matching certificate identity **and** OIDC issuer (from the registry's `slsa_provenance.signer_identity`/`signer_issuer`, the tool options above, or a vfox `PreInstall`). With no expected signer, the SLSA check is **skipped** (not passed). DSSE bundles must carry a matching SHA-256 subject.
- **Keyless cosign** requires a pinned identity — mise applies the registry's `--certificate-identity[-regexp]`, `--certificate-oidc-issuer[-regexp]`, and `--certificate-github-workflow-*`; unknown or empty `--certificate-*` options are errors.
- **GitHub attestation** `signer_workflow` uses an anchored whole-segment match; an empty value fails.
- A lock entry with a checksum and recorded provenance is trusted digest-only; `locked_verify_provenance` / paranoid mode re-verify and require a signer.
- **42 registry tools** (aube, aqua, pixi, ty, pandoc, fnox, doppler, syncthing, …) declare `attestations_since = "<semver>"` (2026.9.14): from that version a missing GitHub attestation is a **hard error** unless you disabled `github_attestations`.

**Limitation:** Aqua tools can't set env vars or do more than download binaries.

#### GitHub Backend

```toml
[tools]
"github:cli/cli" = "latest"

[tools."github:cli/cli"]
version = "latest"
asset_pattern = "gh_*_linux_x64.tar.gz"
bin = "gh"
filter_bins = "gh"
no_app = true
bin_path = "cli-{{ version }}/bin"
rename_exe = "gh"
version_prefix = "release-"
checksum = "sha256:..."
size = "12345678"
api_url = "https://github.mycompany.com/api/v3"
strip_components = 1

[tools."github:cli/cli".platforms]
linux-x64 = { asset_pattern = "gh_*_linux_x64.tar.gz" }
macos-arm64 = { asset_pattern = "gh_*_macOS_arm64.tar.gz" }
```

**Asset autodetection** skips `.deb`, `.rpm`, `.whl`, and `.gem` assets (the last two since 2026.11.3); an explicit `asset_pattern` can still select them. `asset_pattern` is a **glob** — one that looks like a regex and matches nothing errors with a hint to use `matching_regex`.

**Tool options:** `asset_pattern` (must match the **whole** asset name; shortest match wins), `additional_asset_patterns`, `matching`, `matching_regex`, `version_prefix` (default: strip a leading `v`, and on GitHub also a repo-name prefix like `tectonic@0.15.0`; `version_prefix = ""` keeps tags verbatim), `version_regex` (2026.10.20 — a Rust regex the **raw tag** must match to be listed, resolved, or installed; it matches anywhere, so anchor it: `version_regex = '^(v\d+\.\d+\.\d+|nightly)$'`; with `version_prefix`, a tag needs both; an invalid pattern is a hard error), `platforms` (per-platform `asset_pattern`/`additional_asset_patterns`/`url`/`checksum`/`size`/`bin`/`bin_path`/…), `strip_components`, `bin`, `rename_exe`, `bin_path`, `filter_bins`, `format` (force archive type: `tar.gz`, `zip`, `7z`, `raw`, …), `checksum`, `size`, `no_app`, `api_url`, `prerelease`, `github_attestations`, and **`slsa_signer_identity`** (Tera — `{{ version }}`) + **`slsa_signer_issuer`** (2026.9.16; without both, SLSA is skipped; with `locked_verify_provenance` a missing signer is an error).

```toml
"github:myorg/mytool" = { version = "latest", slsa_signer_identity = "https://github.com/myorg/mytool/.github/workflows/release.yml@refs/tags/v{{version}}", slsa_signer_issuer = "https://token.actions.githubusercontent.com" }
```

**Direct per-platform URL** — skip asset selection entirely: `platforms.linux-x64.url = "https://…/tool-{{ version }}-linux.tar.gz"` (or flat `platform_linux_x64_url`). `{{ version }}` is the resolved version even for `latest`. A **top-level** `url` is ignored by github.

**Listing:** reads 1 page of 100 releases, up to 3 pages while no stable release with assets has been found; releases with **no assets**, and tags excluded by `version_prefix`/`version_regex`, are hidden from `mise ls-remote` (unless every platform has a `url` — a `url` on only some platforms lists asset-less releases everywhere); `MISE_LIST_ALL_VERSIONS=1` reads every page. Autodetection skips SBOM/signature/checksum sidecar assets (2026.10.1); ties go to the shortest name. On a checksum mismatch mise hints that the maintainer probably re-uploaded the asset.

**Security (2026.10.19, fail-closed):** SLSA provenance must come from a workflow run in the tool's repo (or the signing-workflow repo) **and** be built from the release's tag — a `slsa_signer_identity` pointing at a branch build no longer verifies; renames are confirmed only via api.github.com. Attestations aren't checked for a custom `api_url` (GHES). Archives containing symlinks that point outside the install dir are refused. On Windows `filter_bins`/`symlink_bins` `.mise-bins` entries are copies (reinstall older installs).

**Precedence and interaction rules:**
- `asset_pattern` **takes precedence** over `matching`/`matching_regex`, which are then silently ignored — an invalid `matching_regex` is never consulted and never reported.
- `matching` + `matching_regex` together = logical **AND**. `matching_regex` is case-sensitive; prefix `(?i)` for insensitive.
- `matching` also **scopes verification**: checksum lookup and SLSA provenance discovery narrow to the selected asset, so a multi-binary release can't verify one binary against another's provenance.
- **`matching` is NOT part of the tool identity** — two `github:owner/repo` entries with different `matching` are treated as **one tool**: mise uses the first entry's options and never installs the other binary. Give each binary its own `[tool_alias]`.

**`rename_exe` table form (2026.7.13+)** exposes several executables from one archive:
```toml
[tools."github:DanielGavin/ols"]
version = "latest"
rename_exe = { "ols-*" = "ols", "odinfmt-*" = "odinfmt" }
```

**`additional_asset_patterns` (2026.7.14+)** overlays multiple release archives from the same tag into one install dir (array or comma string; settable per platform). Each pattern selects one archive — if several match, the shortest name wins; supplemental assets must be archives (bare binaries unsupported) and are extracted **without** the primary asset's `strip_components`/`bin`/`rename_exe`; on path collision the later archive wins. Each supplemental artifact is independently locked and verified, and `--locked` fails if the recorded set no longer matches.
```toml
[tools."github:ollama/ollama"]
version = "latest"
additional_asset_patterns = ["ollama-linux-amd64-rocm.tgz"]
```

> Use `[tool_alias]` for **independent** binaries (each gets its own install dir); use `additional_asset_patterns` when several archives must compose **one** runnable tool.

**`bin_path` templating:** `{{ version }}` plus the **functions** `{{ os() }}` and `{{ arch() }}`, which take remap kwargs — `{{ arch(x64="x86_64", arm64="aarch64") }}`. There are **no bare `{{ os }}` / `{{ arch }}` variables and no `{{ x86_64_arch }}`-style aliases.** Use a single-quoted TOML string when the template contains double quotes.

**Binary path lookup order (github/gitlab/forgejo):** `bin_path` → `bin/` → `Contents/MacOS` (stripped `.app`) → install root if it holds an executable → each immediate subdir (`*.app` contributes `Contents/MacOS`, others `bin/` or the dir itself) → root. With `filter_bins`, only `.mise-bins` goes on PATH. Per-platform keys: `asset_pattern`, `additional_asset_patterns`, `url`, `no_app`, `format`, `strip_components`, `bin_path`, `bin`, `rename_exe`, `filter_bins`, `checksum`, `size`.

**Asset autodetection** scores OS compatibility, arch compatibility, libc variant (gnu/musl on Linux, msvc on Windows), archive-format preference, and build type (avoiding debug/test builds). `.exe` assets are preferred on Windows; OS package/installer assets (`.apk`, `.deb`, `.rpm`, DMG/PKG, MSI/MSIX/AppX) are skipped; `linux-musl` is a fallback for Android/Termux.

**GitHub token priority:** `MISE_GITHUB_ENTERPRISE_TOKEN` (non-github.com hosts) → `MISE_GITHUB_TOKEN` / `GITHUB_API_TOKEN` / `GITHUB_TOKEN` → `github.credential_command` → native OAuth → `github_tokens.toml` → gh `hosts.yml` → `git credential fill`. Since **2026.10.19** tokens are sent **only to the API origin** (GitHub API hosts or the configured `api_url`); list GitHub Enterprise hosts in **`github.enterprise_hosts`** (global-only, `MISE_GITHUB_ENTERPRISE_HOSTS`, https only) to send the token to their `/api/v3`. For GHE data residency, key tokens by `<name>.ghe.com`. gh keyring workaround: `credential_command = 'gh auth token --hostname "$MISE_CREDENTIAL_HOST"'` (`%MISE_CREDENTIAL_HOST%` on Windows). A `credential_command` runs without shims, with `GIT_TERMINAL_PROMPT=0`, at most once per host.

**Auth settings:** `github.credential_command`, `github.gh_cli_tokens` (default `true`), `github.github_attestations`, `github.slsa`, `github.use_git_credentials` (default `false`), `github.enterprise_hosts`. OAuth device-flow: `github.oauth_client_id`, `github.oauth_api_url`, `github.oauth_auth_url`, `github.oauth_export_env` (default `GITHUB_TOKEN`), `github.oauth_open_browser`, `github.oauth_scopes`, `github.oauth_copy_code` (2026.10.9), `github.oauth_cache_get` + `github.oauth_cache_set` (global-only, both required — external token cache commands, 2026.10.9), `github.oauth_cache_secure_enclave` (macOS Touch ID, global-only, 2026.10.10). Env: `MISE_GITHUB_TOKEN`. Debug with `mise token github [--unmask] [--oauth [--refresh] [--raw]]`, `mise token gitlab <host>`, `mise token forgejo`.

#### GitLab / Forgejo Backends

Same forge option surface as GitHub (`asset_pattern`, `additional_asset_patterns`, `matching`, `matching_regex`, `version_prefix`, `version_regex`, `platforms` incl. per-platform `url`, `bin`, `bin_path`, `rename_exe`, `filter_bins`, `format`, `checksum`, `size`, `strip_components`, `no_app`, `api_url`). `slsa_*` and `github_attestations` are **github-only**. Forgejo defaults `api_url` to `https://codeberg.org/api/v1`. **GitLab** matches the release-link **display name** (e.g. `asset_pattern = "binary: Linux amd64"`, then set `bin`/`format`), reads only the newest 100 releases (`MISE_LIST_ALL_VERSIONS=1` for all), and `latest` is the newest listed release; `prerelease` works on both — `latest`/prefix skip prerelease-looking names unless `prerelease = true`.

```toml
"gitlab:gitlab-org/gitlab-runner" = "16.8.0"
"forgejo:user/repo" = "latest"  # Defaults to Codeberg
```

**GitLab auth order:** `MISE_GITLAB_ENTERPRISE_TOKEN` → `MISE_GITLAB_TOKEN` → `GITLAB_TOKEN` → credential_command → `gitlab_tokens.toml` → `glab` CLI → git credential fill. Settings: `gitlab.glab_cli_tokens` (default `true`), `gitlab.use_git_credentials` (default `false`).

**Forgejo auth order:** `MISE_FORGEJO_ENTERPRISE_TOKEN` → `MISE_FORGEJO_TOKEN` → `FORGEJO_TOKEN` → credential_command → `forgejo_tokens.toml` → `fj` CLI → git credential fill. Settings: `forgejo.fj_cli_tokens` (default `true`), `forgejo.use_git_credentials`, `forgejo.credential_command`.

A `credential_command` runs in the configured default inline shell and receives `MISE_CREDENTIAL_HOST` and `MISE_CREDENTIAL_PROVIDER`. The legacy single positional hostname argument warns from **2026.11.0** and is removed in **2027.11.0**. For security, `*.credential_command` is **global-config-only** — it is stripped from project/local `mise.toml`.

#### HTTP Backend

```toml
[tools."http:my-tool"]
version = "1.0.0"
url = "https://example.com/releases/my-tool-v{{version}}.tar.gz"   # also file:///abs/path.tar.gz (2026.9.13)
bin_path = "bin"
format = "tar.gz"  # Explicit override
strip_components = 1

[tools."http:my-tool".platforms]
macos-x64 = { url = "https://example.com/tool-macos-x64.tar.gz", checksum = "sha256:..." }
linux-x64 = { url = "https://example.com/tool-linux-x64.tar.gz", checksum = "sha256:..." }
```

**URL template functions:** `{{ version }}`, `{{ os() }}`, `{{ arch() }}`, `{{ os_family() }}`, and (2026.10.9) `{{ libc() }}` (`gnu`/`musl`) and `{{ avx2() }}` (`yes`/`no`). Remapping: `{{ os(macos="darwin") }}`, `{{ arch(x64="amd64") }}`, `{{ libc(gnu="", musl="-musl") }}`, `{{ avx2(yes="", no="-baseline") }}`.

**Auth headers (2026.10.4)** — Tera-templated, sent to the download, `version_list_url`, and `checksum_url`, and **dropped on cross-origin redirects** unless forwarded:
```toml
[tools."http:internal-tool"]
version = "1.4.0"
url = "https://ghcr.io/v2/org/tool/blobs/{{ version }}"
headers = { Authorization = "Bearer {{ env.GHCR_TOKEN | b64_encode }}" }
headers_forward = { Authorization = ["cdn.example.com", "*.assets.example.com"] }   # names must also be in headers; never forwarded to plain http
```
**URL credentials:** userinfo is stripped from `mise.lock`; credential query parameters (`?token=`, `?sig=`, `?X-Amz-Signature=`…) are **refused** (2026.10.19) — use `headers` or netrc. `checksum_url`, `headers`, and `windows_script_interpreter` can be set per platform. Format detection uses the final URL after redirects.

**Platform keys:** `macos-x64`, `macos-arm64`, `linux-x64`, `linux-arm64`, `windows-x64`, `windows-arm64` (`darwin`/`amd64` variants also accepted; mise will also make sense of near-misses like `darwin-aarch64`).

**Checksums:** `checksum`, plus `checksum_url` and `checksum_expr`. `checksum_url` resolves checksums for **all** target platforms without downloading the artifacts, so one machine can produce a complete cross-platform lockfile; it accepts an individual checksum file, a SHASUMS-style file, or a manifest, with the algorithm detected from the filename (`*.sha512`, `SHA512SUMS`, `*.md5`, `*.b3`; default sha256). `checksum_expr` is an expr-lang expression over `body`, `version`, `os`, `arch`, `url`, `filename` returning a qualified `algo:hash`.

**Version discovery options:**
- `version_list_url` — plain text, line-separated (blank and `#` lines skipped), JSON array of strings, JSON array of objects (fields `version`/`tag_name`/`name`/`tag`/`v`), or a wrapper object (`versions`/`releases`/`tags`); `v` prefixes auto-stripped; prereleases excluded unless `prerelease = true`
- `version_regex` — first capturing group (whole match if no group)
- `version_json_path` — jq-like path (`.[]`, `.field[]`, `.data.versions[]`, `.[?field=value]`); wins over `version_regex`
- `version_expr` — expr-lang over `body` **and** `versions` (the output of `version_regex`/`version_json_path`); it is the **final post-processing step**, not a precedence switch. `sortVersions(array)` is available.
- **Release dates for `minimum_release_age` (2026.10.15):** `version_date_expr` (expr-lang returning `[version, date]` pairs or a map), optional `version_date_url`, and `version_date_next_expr` for pagination (same host, ≤100 pages).

**Other http options:** `file://` URLs (2026.9.13) copy a local archive, still verify `checksum`, and work offline (the URL is locked exactly as written). `windows_script_interpreter = "python"` (Windows only) writes a `.cmd` launcher that runs a raw script with that interpreter. `bin` names a downloaded raw/compressed single binary (ignored for archives; OS/arch suffixes are auto-stripped). Setting `bin_path` disables automatic root stripping.

> **expr-lang gotchas:** write the predicate placeholder as `{ #... }` **with a space** after `{`, because `{#` is the Tera comment delimiter. Index a map by runtime value with `[version + ""]` — a bare `[version]` is read as the literal key `"version"`.

> 🔴 **Changed default (2026.9.6): HTTP tools now extract into their own install directory.** Previously installs were **symlinks into a deduplicated store** (`http-tarballs/`), which meant `uninstall`/`prune` could not reclaim the disk. Own-directory extraction is now the default so that space is actually freed.
>
> Opt back into deduplication per tool with **`shared_extraction = true`**. Existing symlinked installs are unaffected until `mise install --force`, and **legacy `http-tarballs` entries are not reclaimed automatically**.

```toml
[tools."http:my-tool"]
version = "1.0.0"
url = "https://example.com/my-tool.tar.gz"
shared_extraction = true    # opt back into the old shared/dedup cache layout
```

**Shared-store behavior (when `shared_extraction = true`):** entries live in **`$MISE_DATA_DIR/http-tarballs/`** — deliberately *outside* the cache dir so `mise cache clear` can't break the links — keyed by Blake3 of the content plus extraction options (effective filename, root stripping, renaming, format/launcher). Installs are symlinks into it, so identical artifacts are shared across tools. **Neither `mise prune` nor `mise cache prune` reclaims entries** (there is no auto-prune). `--system`/`--shared`/`install-into` always get their own files.

**http bin path lookup** has only 4 steps (no install-root-executable step): `bin_path` → `bin/` → subdirs containing `bin/` → root.

#### S3 Backend

Install from Amazon S3 / S3-compatible storage (MinIO, DigitalOcean Spaces) — useful for private/internal tools.

```toml
[tools."s3:my-internal-tool"]
version = "latest"
url = "s3://tools-bucket/releases/my-tool-{{ version }}.tar.gz"
endpoint = "http://minio.internal:9000"  # MinIO / S3-compat
region = "us-east-1"
version_list_url = "s3://tools-bucket/releases/versions.json"
bin_path = "bin"
```

**Options:** `url` (required), `endpoint`, `region`, `checksum`, `size`, `bin`, `rename_exe`, `bin_path`, `format`, `strip_components`, `platforms`, `version_list_url`, `version_json_path`, `version_expr`, `version_prefix` (also usable as an S3 key prefix for object-listing discovery), `version_regex`.

**Auth:** AWS SDK default chain (env vars → `~/.aws/credentials` → IAM roles). **Safe mode refuses every S3 request.** `bin` also applies to archives. **Not supported:** `checksum_url`, `checksum_expr`, `headers`, `headers_forward`, `shared_extraction`, `windows_script_interpreter`, `version_order`. A manifest (`version_list_url`) wins over object listing; listing is alphabetical (`1.9.0` sorts after `1.10.0`); default `version_regex` is `([0-9]+\.[0-9]+\.[0-9]+)`; `prerelease` is supported.

#### Cargo Backend

```toml
[tools]
"cargo:eza" = "latest"
"cargo:cargo-edit" = { version = "latest", features = "add" }
"cargo:demo" = { version = "latest", default-features = false }
"cargo:demo" = { version = "latest", bin = "demo", crate = "demo", locked = false }
```

**Git-based installation:**
```bash
mise use cargo:https://github.com/user/repo@tag:v1.0
mise use cargo:https://github.com/user/repo@branch:main
mise use cargo:https://github.com/user/repo@rev:abc123
```

**Tool options:** `features` (**skips binstall**), `default-features` (default `true`; `false` **skips binstall**), `bin`, `crate` (monorepo select), `locked` (default `true`), `rename_exe` (2026.10.14 — e.g. `rename_exe = "wit-lsp"`), `install_env`. `bin`/`crate`/`locked` are passed through and do not skip binstall. The GitHub shorthand works for Git sources (`cargo:owner/repo@tag:v1`); a Git source lists only `HEAD`. cargo writes credentials from a `--git` URL into `.crates.toml`, so keep them in a git credential helper. `rust`, `cargo-binstall`, and `sccache` in config install first.

Fallback to `cargo install` happens **only** on binstall exit code **94** (no prebuilt artifact) — other errors do not trigger it. With `cargo.binstall_only = true` there is no fallback. Explicit Git sources always use `cargo install --git`.

Since **2026.7.18**, installs record the effective `features`/`default-features`/`bin`/`crate`/`locked` and auto-reinstall the same version when any change. Feature names are normalized, so reordering or switching string↔array does not force a needless reinstall.

**Settings:** `cargo.binstall` (default `true`), `cargo.binstall_only` (default `false`), `cargo.binstall_quickinstall` (default `false` — mise passes `--disable-strategies compile,quick-install`; `true` → `--disable-strategies compile`), `cargo.binstall_native` (**graduated from experimental in 2026.7.16**; now also discovers conventionally named GitHub release artifacts from a crate's linked repo when `package.metadata.binstall` is absent, and works under the default `locked = true` path — warns 2027.1.0, becomes the default 2027.7.0), `cargo.registry_name`.

#### Pipx Backend (alias of `pypi:`)

`pipx:` is a fully supported alias for [`pypi:`](#pypi-backend-formerly-pipx) — no warnings, same options, and settings accept both `pipx.*` and `pypi.*` spellings. Its doc page was folded into `pypi.html#compatibility-with-pipx` (2026.10.x). Prefer `pypi:` in new configs.

> ⚠️ **`pypi:black` and `pipx:black` are distinct tool identities** — separate install directories and separate lock entries. Switching the prefix creates a **new installation**; it is not a rename. mise preserves explicit `pipx:` names in output and lockfiles.

#### NPM Backend

```toml
[tools]
"npm:prettier" = "latest"
"npm:some-cli" = { version = "latest", allow_builds = ["esbuild"] }  # approve build scripts
"npm:bibtex-tidy" = { version = "latest", allow_low_downloads = true }
```

**Git sources (2026.10.4):** `mise use 'npm:git+https://github.com/owner/repo'`, `mise use 'npm:github:owner/repo@v1.2.0'` (also `gitlab:`/`bitbucket:` shorthands). The version is a git ref (`ref:`/`tag:`/`branch:`/`rev:` accepted); `latest` is the default branch.

**Private registries** are read from the **user** `~/.npmrc` (or `NPM_CONFIG_USERCONFIG`) and `NPM_CONFIG_*` — never from a project `.npmrc`: scoped registries, auth tokens, `ca`/`cafile`, `strict-ssl`, `cert`/`key`, `tokenHelper`. mise warns when a package installs no executables (2026.10.7).

**Auto-detects package manager:** prefers the embedded `aube` installer, else `npm`, with `aube_cli`/`bun`/`pnpm` as alternatives. Override with `npm.package_manager = "auto|npm|aube|aube_cli|bun|pnpm"` (env: `MISE_NPM_PACKAGE_MANAGER`). Because `aube` is embedded, **Node is only required to run the tool, not to install it**. `aube_cli` installs through a separately installed standalone Aube executable, avoiding Aube's npm compatibility shim.

**Tool options:**

| Option | Applies to | Description |
|--------|-----------|-------------|
| `allow_builds` | aube, aube_cli, pnpm ≥10.4, npm ≥11.16 | Array of package names, or `true` for all. Lifecycle build scripts are **denied by default**. A **list approves only the listed packages** (2026.10.16 — aube's bundled trusted list is turned off). Mapping: embedded aube → `aube.allowBuilds`; aube_cli/pnpm → `--allow-build=<pkg>`; npm 11.16+ → `--allow-scripts=<pkg>`; older npm unsupported (warns); Bun ignores it; `true` → `--dangerously-allow-all-builds`/`--dangerously-allow-all-scripts`. |
| `trust_policy_excludes` | aube / aube_cli | Exempt reviewed packages from trust-policy downgrade checks; supports ranges (`"undici@^5 \|\| >=6 <7"`) |
| `allow_low_downloads` | aube | Bypasses aube's weekly-download popularity gate (**1000** downloads) for the requested package only — transitive deps stay gated |
| `allow_exotic_deps` | aube | **2026.9.10+.** Array of package names, or `true` for all — permits git/file/tarball dependencies that aube blocks by default |
| `aube_args` / `npm_args` / `pnpm_args` / `bun_args` | respective PM | Extra args (`aube_args` is ignored for embedded aube, which installs in-process) |
| `install_env` | aube_cli, pnpm, Bun, npm | Env during install. **Embedded aube (the default) ignores it with a warning.** |

**Lifecycle-script defaults:** npm passes `--ignore-scripts=true` (dropped when `allow_builds` is used with npm ≥11.16.0); bun runs no dependency scripts unless `bun_args = "--trust"`; aube/pnpm need `allow_builds`.

**Setting `npm.shell_out`** (default `false`) routes metadata lookups and installs through the npm CLI instead of the built-in aube implementation.

**Provenance verification (on by default since 2026.11.1).** `npm.provenance` (default `true`, env `MISE_NPM_PROVENANCE`) verifies the Sigstore provenance of the tool's **own** package (not its dependencies) against the sha512 integrity the install locks. Provenance from GitHub Actions and gitlab.com CI/CD is accepted (self-hosted GitLab is not).

- Only the embedded aube installer verifies; any other `npm.package_manager` installs unchecked.
- A package that publishes no provenance (or a private registry that serves none) installs as before — **unless `mise.lock` pins a provenance repository** for it, in which case the install fails. A later version published from a **different** repository, or one that stops publishing provenance, also fails.
- `mise install` (not `mise lock`) records the pin as `provenance = { repository = "github.com/owner/repo" }` on the tool entry, and only into an existing revision-4 `mise.lock` (see [Lockfiles](#lockfiles-miselock)). An already-installed version isn't re-verified — reinstall to check it.
- Verification fetches Sigstore trust roots; on a network that blocks them, installs of provenance-publishing packages fail until you opt out. `provenance_api_failures_fatal` covers the attestation lookups.
- The `paranoid` category `provenance` turns verification on even when `npm.provenance = false`, and makes unverifiable installs (non-aube package managers, `checksum`-pinned archives) **fail** instead of going unchecked.

```toml
[settings.npm]
provenance = false   # opt out (or MISE_NPM_PROVENANCE=0)
```

npm tools resolved from a `mise.lock` pin are auto-trusted through the low-download gate — reproducing an existing lockfile no longer needs `allow_low_downloads`.

Supports minimum-release-age protection for transitive dependencies via compatible package managers (`npm >= 6.9.0` via `--before`, 11.10.0+ via `--min-release-age`; `aube`; `bun >= 1.3.0`; `pnpm >= 10.16.0`). Supply-chain scanning: `AUBE_SECURITY_SCANNER=/abs/path/scanner.mjs` (Node 22.6+), or Socket Firewall as `sfw mise install npm:x` (Socket Firewall intercepts TLS, so give mise its CA: `sfw sh -c 'MISE_EXTRA_CA_CERTS="$NODE_EXTRA_CA_CERTS" mise upgrade'`).

#### Go Backend

```toml
[tools]
"go:github.com/DarthSim/hivemind" = "latest"
"go:github.com/golang-migrate/migrate/v4/cmd/migrate" = { version = "latest", tags = "postgres" }
"go:github.com/grafana/oats" = "v0.7.1-0.20260703092802-96201f1b8136"  # module pseudo-version
```

**Tool options:** `tags` (string or array → `-tags`), `install_env`. mise sets `GOBIN` to the tool install dir *after* applying `install_env`; use `install_env = { GOPROXY = "direct" }` for unreleased revisions. Installs use `-mod=mod` (2026.10.6). `GOPRIVATE` must be in the environment or `[env]` — `install_env` doesn't affect listing, and `go env -w` isn't read.

#### Gem Backend

```toml
[tools]
"gem:rubocop" = "latest"
```

**Tool options:** `install_env`, and **`source`** (2026.9.18) — a per-gem registry URL, added alongside the default sources rather than replacing them:
```toml
"gem:internal-tool" = { version = "latest", source = "https://rubygems.pkg.github.com/myorg" }
```
Credentials go in the URL as basic auth (redacted in output; https required except on localhost). A `https://rubygems.pkg.github.com/<org>` source **without** credentials uses mise's GitHub token (needs `read:packages`). Sources without the JSON versions API (GitHub Packages and similar) are listed from their compact index (2026.10.23), so `latest`, prefixes, `mise outdated`, and `mise upgrade` work; GitHub Packages publishes no release dates, so `minimum_release_age` can't filter those versions. `latest` skips RubyGems prereleases (`4.1.0.beta3`) — request one explicitly or set `MISE_PRERELEASES=1` (2026.10.24).

Requires `gem` (Ruby) on PATH. Each tool gets its own `GEM_HOME`; patch upgrades of mise Ruby keep working via the minor-version path, but a **minor** Ruby change needs `mise install -f "gem:*"`.

#### Conda Backend

```toml
[tools]
"conda:ruff" = "latest"
"conda:postgresql" = { version = "17", channel = "conda-forge" }
"conda:mytool" = { version = "latest", channel = "https://conda.example.com/team" }   # full channel URL
```

Direct anaconda.org API — no conda/mamba/micromamba required. mise **solves and installs transitive dependencies** into a per-tool prefix (only the package's own commands go on PATH; launchers set `CONDA_PREFIX`). It doesn't read `environment.yml`. The package **and every dependency must come from one channel** — so `bioconda` usually can't be used on its own. **Tool option:** `channel` (overrides `conda.channel`, default `conda-forge`; may be a full URL). Mirrors go through `url_replacements` (which refuses https→http with credentials). Release dates are reported for `minimum_release_age` (2026.10.15). Platforms auto-detected (linux-64, linux-aarch64, osx-64, osx-arm64, win-64) with `noarch` fallback.

#### SPM Backend

```toml
[tools]
"spm:tuist/tuist" = "latest"
"spm:swiftlang/swiftly" = { version = "latest", filter_bins = ["swiftly"] }
"spm:org/tool" = "rev:abc1234"   # 2026.8.4+ — pin to a commit, builds from source
```

**Tool options:** `filter_bins` (array or comma-string; filtering happens before `swift build` and fails if a listed name isn't an executable product), `artifactbundle` (tri-state: unset = try prebuilt bundle then source; `true` = require a bundle; `false` = always build from source), `artifactbundle_asset` (required when a release has multiple bundles), `provider` (`github`/`gitlab`, default `github`), `api_url`, `install_command` (**2026.7.16+**; source installs only, cannot combine with `filter_bins`, sets `PREFIX` and `MISE_TOOL_INSTALL_PATH`, fails if nothing lands in `bin/`), `install_env`. **Setting:** `spm.artifactbundle_only` (default `false`).

`rev:<sha>`/`ref:<sha>` always builds from source. Versions are tags verbatim (`@1.2.0` and `@v1.2.0` both resolve). A non-github.com/gitlab.com host needs trust (safe mode refuses it unless `api_url` is set, and refuses `install_command`). With a lockfile checksum, a missing or mismatched artifact bundle **fails** instead of falling back to source. Swift is required even for bundles.

#### Dotnet Backend

```toml
[tools]
"dotnet:GitVersion.Tool" = "5.12.0"
"dotnet:GitVersion.Tool" = { version = "latest", prerelease = true }
```

**Tool options:** `prerelease`, `install_env`. Listing uses netrc for the feed; installs use `NuGet.Config` credentials.

**Settings:** `dotnet.registry_url` (default `https://api.nuget.org/v3/index.json`), `dotnet.isolated` (default `false`), `dotnet.cli_telemetry_optout`, `dotnet.dotnet_root`. `dotnet.package_flags` is **deprecated** (warns 2026.11.0, removed 2027.11.0) — use the `prerelease` tool option or the global `prereleases` setting.

#### Spinel Backend (Experimental)

Added **2026.10.2** (docs: `https://mise.jdx.dev/dev-tools/backends/spinel.html`). Compiles a Ruby CLI from a GitHub repo into a **native binary** with Matz's Spinel compiler — the result needs neither Ruby nor Spinel at runtime. Requires `experimental = true`; may be removed in a future release.

```toml
[settings]
experimental = true

[tools."spinel:tobi/try"]
version = "1.10.1"
entrypoint = "try.rb"
bin = "try"
tag_prefix = "v"
```

| Option | Default | Notes |
|--------|---------|-------|
| `entrypoint` | `main.rb` | The single file compiled |
| `bin` | repo name | Output binary name |
| `tag_prefix` | none | Only tags with this prefix are listed as versions |
| `source_ref` | none | Full 40-char commit SHA; the version becomes a label only |
| `spinel` | `spinel` on PATH | Path to the compiler (mise does **not** install it) |

**Requirements:** macOS or Linux only, `git`, a C compiler, and the `spinel` compiler on PATH. Versions come from `git ls-remote` tags (no GitHub API). Compiles **one** entrypoint — no gems, data files, or submodules, and Spinel handles only part of Ruby.

> **`pkgx:` was removed in 2026.9.13** — see the note under [Backends Overview](#backends-overview).

#### UBI Backend (Deprecated)

**Options:** `exe`, `rename_exe`, `matching`, `matching_regex`, `provider`, `api_url`, `extract_all` (incompatible with `exe`/`rename_exe`), `bin_path` (only meaningful with `extract_all`), `tag_regex`.

> **Migration gotchas:** (1) Neither backend keeps `matching` variants apart — `ubi:o/r[matching=a]` and `[matching=b]` resolve to one tool and only the first installs; give each binary its own `[tool_alias]`. (2) ubi applies substring `matching` only as a *tiebreaker* among assets already matching your OS/arch and skips it when a single asset matches, whereas `github` applies it as a *pre-filter* before autodetection — so you get the named binary or a clear error. (3) ubi lists Git **tags** when a repo has no releases; `github` doesn't. `extract_all` ignores `exe`/`rename_exe`.

#### vfox & asdf Plugins

**vfox (recommended plugin system):** cross-platform (Win/macOS/Linux), built-in Lua interpreter with HTTP/JSON/archive modules, attestation verification, lock files. Plugins can come from a git repo, a zip, or a local dir; backend plugins use `name:tool`. **Tool option:** `install_env` (applies to `cmd.exec` **and `os.execute`** during install hooks — vfox's built-in Lua helpers do not use it); any other options reach hooks as **`ctx.options`** (`MISE_TOOL_OPTS__*` env is legacy). `url_replacements` and netrc apply to the Lua HTTP module (a plugin's own `Authorization` header wins on the same origin).
- **Plugin-API additions (2026.10.x):** `PreInstall` may return `headers` (10.13); per-version `created_at` release dates (10.15); `BackendListVersions` may return `aliases` (10.16); `RUNTIME.cacheDirPath` — a per-plugin cache dir (10.17); attestation fields `slsa_source_repository`/`slsa_source_tag` (10.19). Signed backend plugins: `mise plugins install vfox-backend:my-backend 'packslip:your-org/my-backend#1.0.0'` (10.16). Since 2026.8.1 Lua plugins gain `strip_components = 1` on `archiver.decompress`, plus sorted `file.list`, `file.glob`, and `file.move`. vfox honors `url_replacements` and `netrc`.

- **New hooks:** `hooks/backend_uninstall.lua` (backend plugins; runs before removal on uninstall/upgrade/prune, keeps the install dir if it errors — 2026.9.13) and `hooks/mise_install_satisfied.lua` (re-runs `PostInstall` plus the tool's `postinstall` when options change, e.g. gcloud `components` — 2026.9.15).
- **Signer fields (security):** a `PreInstall` returning SLSA provenance must also return `slsa_signer_identity`/`slsa_signer_issuer` (else SLSA is skipped); keyless cosign must set `cosign_certificate_identity` or `cosign_certificate_identity_regexp` (optional `cosign_certificate_oidc_issuer`) — **breaking in 2026.10.0**.
- **Signed plugin distribution:** `mise plugins install vfox:NAME 'packslip:OWNER/REPO#PLUGIN_VERSION'` or `[plugins] "vfox:NAME" = "packslip:OWNER/REPO#VER"`.
- Registry **vfox** plugins live under the `jdx` org (e.g. `vfox:jdx/vfox-php`); asdf plugins remain under `mise-plugins` (`asdf:mise-plugins/asdf-php`).

**asdf (legacy):** Unix-only, bash scripts, needs curl/jq, no Windows, disabled by default on Windows. New asdf tools rarely accepted for supply-chain reasons. **Tool option:** `install_env`. Since 2026.7.18, `depends` tools are on the PATH given to `bin/download` and `bin/install`, and `bin/list-all`/`bin/latest-stable` receive resolved `[env]` values and `_.path` additions.

> Tool plugins support attestation verification; **backend plugins do not** — a known security gap.

```bash
mise plugin install my-plugin https://github.com/username/my-plugin
mise install my-plugin:some-tool@1.0.0
```

### Lazy Tools

Added **2026.9.0**. `lazy = true` makes mise generate **bootstrap shims** into its normal user/system shim farms without installing the tool. The provider is installed only when one of its commands is first called, then executes immediately; later calls run the real binary with no further mise dispatch.

```toml
[tools]
node = { version = "24", lazy = true }
"github:example/acme" = { version = "1.2.3", lazy = true, lazy_bins = ["acme", "acmectl"] }
```

- **Registry tools** derive their command names from the registry's `bins` metadata. **Explicit or non-registry backends must declare them** with `lazy_bins`.
- A bare `mise install` **skips** lazy declarations — use `mise install --include-lazy` to provision them all.
- Lazy tools also install when their command is invoked from a `mise run` task or `mise x` (2026.9.1+), matching activated-shell behavior. mise inserts the shim farms *after* real tool paths for lazy toolsets.
- Lazy tools are **not** reported as `missing: <tool>` on project entry or a bare `mise install`, regardless of `status.missing_tools` (2026.9.8+), and `mise doctor` doesn't list them as problems (2026.10.21). Ordinary missing tools still are. `mise ls` shows an uninstalled lazy tool as `(lazy)` (2026.10.23) — or `(missing)` when it can't install on first use (explicit backend without `lazy_bins`, or refused by safe mode); `--json` still reports `"installed": false`.
- Invoking a lazy shim also installs that provider's configured `depends`. `mise install <lazy-tool>` installs just that one. Run `mise reshim` after hand-editing a lazy declaration.
- A bare `mise upgrade` **skips** lazy tools with no installed version (2026.10.11); `mise outdated` marks them `[MISSING] (lazy)`. If another version of the tool is installed, the lazy request is treated like any missing version. Lazy install works even with `not_found_auto_install` off. A lazy declaration never overrides a higher-precedence non-lazy one. Safe mode gives **project** lazy tools no bootstrap shims (2026.10.19).

**Related path settings (2026.9.0+):** `shims_dir` (`MISE_SHIMS_DIR`), `system_installs_dir` (`MISE_SYSTEM_INSTALLS_DIR`), `system_shims_dir` (`MISE_SYSTEM_SHIMS_DIR`), plus `mise reshim --system` — for system-scoped and collocated layouts.

> For a machine-wide catalogue of ordinary tools, prefer `lazy = true` over standalone [tool stubs](#tool-stubs). Tool stubs remain useful when the executable file itself should carry a portable, self-contained definition.

### Tool Stubs

An executable file that records how to obtain and run one tool — commit it so `./bin/py` selects the intended runtime. mise installs the tool on first execution; a normal project `mise install` does **not** discover and install arbitrary stub files.

```toml
#!/usr/bin/env -S mise tool-stub

tool = "python"
version = "3.14"
bin = "python"
```

```bash
chmod +x ./bin/py && ./bin/py --version
```

Fields sit at the **top level** — a stub is a tool declaration, not a whole `mise.toml`, so do **not** wrap them in `[tools]`. `tool`, `version` (default `latest`), `bin` (default the stub filename), `os`, and `install_env` control the stub; other keys pass to the selected backend. Omitting `tool` makes a top-level or platform-specific `url` select the HTTP backend; otherwise the **filename** is used as the tool name. A stub requires `mise` on PATH unless generated with the optional bootstrap wrapper (`--bootstrap`). A Windows `.cmd` launcher is generated beside it, and executed stubs are tracked in `~/.local/state/mise/tracked-stubs` so `mise prune` keeps their versions.

> 🔴 **Stub locking moved to `mise.lock` (2026.9.13, breaking).** An embedded `[lock]` section in a stub is now **ignored and removed**. `mise generate tool-stub ./bin/node --lock [--version 26]` writes lock data to the `mise.lock` of the **nearest project config above the stub** (it fails if there is none) and lists the stub under a top-level `tool-stubs = ["bin/node"]` array. The stub keeps its fuzzy request; `mise lock --bump` re-resolves stubs and prunes deleted ones. In locked mode a stub with no entry for the current platform is rejected.

`mise generate tool-stub` flags: `--lock`, `--version`, `--bootstrap`/`--bootstrap-version`, `--platform-url [PLATFORM:]URL`, `--platform-bin PLATFORM:PATH`, `--checksum-algorithm blake3|sha256` (default blake3; not with `--lock`/`--skip-download`), `--fetch`, `--skip-download`, `--http`, `-u/--url`, `-b/--bin`.

> `env -S` is required because Unix shebangs traditionally allow only one argument after the interpreter — it splits the line into `env` → `mise` → `tool-stub`.

### Shims and Aliases

**Shim location:**
- Linux/macOS: `~/.local/share/mise/shims`
- Windows: `%LOCALAPPDATA%\mise\shims`

Three activation methods:
1. **PATH activation** (`mise activate`) — updates PATH per prompt via `mise hook-env`
2. **Shims** (`mise activate --shims`) — tiny intercepting executables
3. **Explicit** (`mise exec`, `mise run`, `mise en`)

```bash
# Shim mode (non-interactive shells — .bash_profile/.zprofile)
eval "$(mise activate zsh --shims)"

# PATH mode (interactive shells — .bashrc/.zshrc)
eval "$(mise activate zsh)"
```

**Best practice:** Use both — shims in your login profile for non-interactive/GUI processes, PATH activation in your rc file for interactive shells.

> How the combination behaves depends on `not_found_auto_install`. **Enabled (the default):** `mise activate` *keeps* the shims dir in PATH, behind the tool paths it manages, so resolved tools win and shims remain an auto-install fallback; `mise doctor` does not flag this. **Disabled:** `mise activate` removes the shims dir from PATH.

**Shims vs PATH:**
- **Shims**: `[env]` vars only load when a shim is called; `watch_files` unsupported; only `preinstall`/`postinstall` hooks work; `which` points to the shim (use `mise which` for the real path)
- **PATH**: full environment, all hooks (`cd`, `enter`, `leave`, `watch_files`), `which` shows the actual binary. When a fuzzy version is active, the PATH entry may use the requested-version symlink (`installs/python/3.15/bin`) rather than the fully resolved patch dir.
- **Recommendation**: PATH for interactive shells; shims for IDEs/cron/CI/non-interactive

Shells with a cd hook: `bash` (chpwd emulation that wraps `cd`/`pushd`/`popd`, plus `PROMPT_COMMAND`), `zsh` (`chpwd`), `fish` (`--on-variable PWD`; `mise_fish_mode = eval_after_arrow | disable_arrow`), `xonsh` (`on_chdir`), `elvish` (`after-chdir`), PowerShell 7+ (`LocationChangedAction`). **PowerShell 5.1 and Nushell** update only at the next prompt, so `cd a && node -v` on a single line uses the *original* directory's tools there — shims always work.

**Shim internals:** on Unix a shim is a symlink to `mise`; on Windows a copy of `mise-shim.exe` (`mise doctor`/`reshim` flag a stale one — fix with `mise reshim --force`). Shims make `command -v tool` succeed for uninstalled tools — use `command mise which tool` in scripts that probe availability. To fail instead of falling back/auto-installing, set **both** `not_found_system_fallback = false` and `not_found_auto_install = false`. Don't install shells (bash, zsh) with mise.

**`mise reshim`** regenerates shims for **all installed** tools, not just active ones. It only replaces or removes entries it recognizes as mise shims (`-f/--force` rebuilds mise-owned shims; it does not adopt unrelated files), so a shared `shims_dir` like `~/.local/bin` works for reshim — though not for `mise activate`/hook-env. Runs automatically on install/update/remove. Exclude tools from shim generation with `[settings.shims] exclude = [...]` (2026.9.10 — also skips lazy bootstrap shims and plugin shims; version-qualified shims like `python3.12` stay); `activate_shims` and `not_found_system_fallback` both default `true`.

**`windows_shim_mode`** (default `exe`): `exe` (copies native `mise-shim.exe`; recommended — works with all shells, package managers, and `where.exe`), `file` (`.cmd` batch + extensionless bash script for Git Bash/Cygwin), `hardlink` (NTFS, same filesystem; needs `mise reshim --force` after upgrading mise), `symlink` (needs admin or Developer Mode).

> Windows (2026.8.1+): mise **warns when the generated PATH exceeds ~8191 characters**, the limit at which `cmd.exe` silently drops the variable and every command appears unrecognized. Shims are the documented workaround.

**Tool aliases (remap to different backend):**
```toml
[tool_alias]
node = 'github:company/our-custom-node'
erlang = 'aqua:company/our-custom-erlang'

# Install multiple independent binaries from the same GitHub release
dhall-json = 'github:dhall-lang/dhall-haskell'
dhall-lsp  = 'github:dhall-lang/dhall-haskell'

[tool_alias.node.versions]
lts = '22'
my_custom_20 = '20'
```

```toml
[tools]
dhall-json = { version = "v1.42.2", matching = "dhall-json" }
dhall-lsp  = { version = "latest",  matching = "dhall-lsp-server" }
```

> **Aliases are not an overlay mechanism** — each alias creates a separate install directory. Adding a version alias also creates a symlink (`installs/node/20 -> ./20.x.x`).

> `[alias]` is **deprecated** in favor of `[tool_alias]` (no removal version announced).

**Template-driven version aliases:**
```toml
[tool_alias.node.versions]
project-lts = "{{ env.PROJECT_NODE_VERSION | default(value='24') }}"
```

> Don't compute a tool's version by **invoking that same tool** (e.g. `exec(command='node --version')`) — the docs now explicitly discourage it: resolution can happen before the tool exists, or re-enter mise through a shim.

**Shell aliases:**
```toml
[shell_alias]
ll = "ls -la"
gs = "git status"
```

Manage from the CLI with `mise shell-alias get|set|unset|ls` and `mise tool-alias get|set|unset|ls`.

**Custom plugin repos (`[plugins]`)** — affects new installations only:
```toml
[plugins]
node = "https://github.com/myorg/asdf-node#v2"  # optional #GITREF suffix
my-tool = "vfox:myorg/vfox-my-tool"             # asdf:/vfox:/vfox-backend: prefixes
example = "./plugins/mise-example"              # local path (2026.7.18+)
```

The type prefix is optional — if omitted, mise clones first and detects the type. **Local filesystem paths** (absolute, `~/`, or explicit `./`/`../` relative to the config root) install as **symlinks**, so source edits take effect immediately; use `mise plugins install --force <NAME>` to replace an existing plugin with a local source. `[plugins]` replaces the deprecated `shorthands_file` setting (**removed 2026.12.0**).

> The `[_]` table holds arbitrary user data that mise never parses — useful for sharing values with external tooling.

### Shell Completion (per-directory tab-complete)

Combined with `mise activate`, shell completions make `mise <TAB>` and `mise run <TAB>` automatically list the tasks/tools defined in the current directory's `mise.toml`.

```bash
# bash
mise completion bash > /etc/bash_completion.d/mise
# zsh (ensure the dir is on $fpath)
mise completion zsh > /usr/local/share/zsh/site-functions/_mise
# fish
mise completion fish > ~/.config/fish/completions/mise.fish
# powershell
mise completion powershell | Out-String | Invoke-Expression
# nushell (2026.10.20; `nu` also works)
mise completion nushell --install   # then `source` the printed path in config.nu
```

`mise completion --install` writes self-contained scripts directly.

`mise completion <shell> --install` destinations: bash `~/.local/share/bash-completion/completions/mise`; zsh `~/.local/share/zsh/site-functions/_mise` (prints the `fpath` line to add); fish `~/.config/fish/completions/mise.fish`; PowerShell `~/.config/powershell/completions/mise.ps1` (Windows: `%LOCALAPPDATA%\PowerShell\completions\mise.ps1`); Nushell `~/.config/nushell/completions/mise.nu`. Homebrew, pacman, the Ubuntu PPA, and COPR packages install completions for you; the mise.run installer does not.

> 🔴 **2026.8.11:** mise's own CLI moved from clap to usage-rs, and completions/help are now generated from compiled usage metadata rather than an external `usage` CLI. As a result **`--include-bash-completion-lib` and `--usage` are hidden no-ops.** Command behavior, flags, and aliases are otherwise preserved.

> **Standalone usage scripts** need a separate one-time opt-in: `source <(usage g completion-init bash)` in `~/.bashrc` (zsh: same in `~/.zshrc`; fish: `usage g completion-init fish | source`). Since usage 3.5.6 the completion spec cache lives in `${XDG_CACHE_HOME:-$HOME/.cache}/usage` — **regenerate completion scripts** to pick it up.

### Packslip, Man Pages, and Agent Skills

Tools installed through the [`packslip:` backend](#packslip-backend-preferred-where-available) can ship **man pages, shell completions, and agent skills that match the version active in your project**. The publisher declares them in the release manifest. An existing installation from another backend does **not** acquire them.

**Completions** (zsh, bash, fish, PowerShell) become available automatically with `mise activate` — the installed completion follows the tool version active in each project, so changing versions needs no reinstall. mise registers a loader and reads the publisher's script only when you complete a command; leaving the project removes the registration.

```bash
mise completion zsh --tool hk --install    # manual setup without shell activation
mise completion zsh --tool hk              # print without installing
```

`--tool` accepts the command name (`hk`) or a backend identifier (`packslip:github.com/jdx/hk`); with `--install`, pass the executable name. Without `--tool`, `mise completion` generates completions for mise itself. mise writes the completion file but never edits your shell config, and preserves an existing file it did not create unless you pass `--force`.

**Man pages** declared as a static `man` resource are added to `MANPATH` while that tool version is active (in an activated shell and under `mise exec`/`run`/`env`). They live under `.mise-packslip/man` in the tool's installation; a tool installed with an older mise must be reinstalled once. Generated `exec` resources and pages derived only from a `cli-spec` are **not** installed automatically.

**Agent skills** — a directory holding `SKILL.md` and its supporting files, in the Agent Skills format. mise knows which tool version is active, so it hands an agent the skill for exactly that version.

```bash
mise skills ls [--json]                    # skills the active tools declare
mise skills sync                           # link them where your agent looks (--dir, --prune, -g/--global)
```

| Setting | Default | Description |
|---------|---------|-------------|
| `skills.fetch` | `true` | Fetch the agent skills a tool's packslip declares when installing it |
| `skills.auto_sync` | `false` | Link active tools' skills into the project after `mise install`, `mise use`, and `mise upgrade` (sync also writes `.mise-skills.json` and a `.git/info/exclude` block) |
| `skills.dir` | unset | Where `mise skills sync` links skills — under the project root, or under `$HOME` with `--global`. **Unset (2026.11.0):** use `.agents/skills` if it exists, else an existing `.claude/skills`; if neither exists, create `.agents/skills` plus a `.claude/skills` symlink to it (existing `.claude/skills` projects are unchanged) |
| `skills.prune` | `false` | Remove links mise made for skills that are no longer active when syncing |

**Global tools (2026.11.0):** with `skills.auto_sync`, `mise use -g` links a global tool's skills under `$HOME`, and `mise install`/`upgrade` keep those home links current. A project directory gets only the tools its own config declares — global-only tools link only under home (2026.11.2; stale project links stay until `mise skills sync --prune`). `mise skills sync --global` links only tools from global and system config.

> If a completion needs the publisher's generator command, mise runs it **on demand** and caches successful output per installed version/executable/shell. `packslip.exec` controls resource generation **at install time**; setting it to `false` does **not** disable on-demand completion generation.

> Since **2026.10.1** a packslip install **fails** if a declared skill can't be fetched; the next install fetches only the missing skills. Packslip pins by forge **repository ID** (lockfile revision 3), so a renamed (2026.9.16) or transferred (2026.10.2) repo keeps installing with a one-time warning, while a different repo under the old name is refused — the error names what to clear (`mise packslip forget`, the `mise.lock` entries, or both). The registry has moved timoni, worktrunk, helmfile, and dagu (newer versions) to packslip; older versions remain reachable via an explicit `aqua:` spec. **usage 6.12** itself ships a version-matched skill: `mise use usage && mise skills sync`.

### Lockfiles (`mise.lock`)

**Lockfiles are not created automatically.** With `lockfile` unset, mise **updates existing lockfiles but doesn't create new ones** — create one with `mise lock` (or `touch mise.lock && mise install`). `MISE_LOCKFILE=1` keeps that update-only behavior and is *not* the same as `lockfile = true`. Global lockfiles are only created by `mise lock --global`. The docs now recommend **committing `mise.lock`**.

> 🔴 **Lockfiles are rebuilt completely by default (2026.10.19).** `lockfile_mode` now defaults to **`generate`** (no longer experimental): installs rebuild `mise.lock` from the current requests, reuse unchanged entries, and also lock the **other platforms**. `lockfile_mode = "merge"` still restores the old behavior but is **deprecated** (2026.10.24; warns from 2027.4.0, removed in 2027.10.0) — remove the line instead. Neither mode creates lockfiles. `lockfile_auto_prune` (default `true`, 2026.10.4) makes a full `mise lock` drop tools absent from the active config — set it `false` for a lockfile shared by different envs. Generate mode deletes unreferenced sidecars; merge mode never does.

**What gets locked:** config `[tools]`, **task tools** (including task templates and included task files), and tool stubs. `.tool-versions` tools are **not** locked. `conf.d` files share the parent directory's lockfile; idiomatic files use the nearest `mise.toml`'s lockfile. `lockfile = false` together with `locked = true` fails as soon as a tool must actually be installed (`locked mode requires lockfile to be enabled`); `mise ls` and already-installed tools still work. **Default platforms for a new lockfile:** `linux-x64`, `linux-x64-musl`, `linux-arm64`, `linux-arm64-musl`, `macos-x64`, `macos-arm64`, `windows-x64`, plus the current platform (`--platform` qualifiers: `gnu`, `glibc`, `musl`, `msvc`, `baseline`, `musl-baseline`; Bun adds `-baseline` on CPUs without AVX2).

```toml
lockfile_version = 4

[[tools.node]]
version = "20.11.0"
backend = "core:node"
specifiers = ["20"]

[tools.node."platforms.linux-x64"]
checksum = "sha256:a6c2..."
url = "https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-x64.tar.xz"

[[tools.ripgrep]]
version = "14.1.1"
backend = "aqua:BurntSushi/ripgrep"
options = { exe = "rg" }
```

The writer uses a **quoted** platform key — `[tools.node."platforms.linux-x64"]` — and omits `size`.

Tool entry fields: `version` (required), `backend`, `specifiers` (the original requests that resolved here), `options` (backend-specific artifact identity), `platforms`, plus `aube` / `uv` (revision 2 — sidecar path + digest for npm / Python dependency graphs) and `provenance = { repository = "github.com/owner/repo" }` (revision 4 — the source repository an npm package's [provenance](#npm-backend) must come from; written by `mise install`, not `mise lock`, and sometimes as a `[tools."npm:x".provenance]` sub-table). The tool-level `provenance` table is unrelated to the platform-level `provenance` string below. A top-level `tool-stubs = [...]` array lists locked [tool stubs](#tool-stubs) (2026.9.13); top-level `[conda-packages.<platform>]` holds conda dependency artifacts.
Platform sub-fields: `checksum` (SHA256 or Blake3), `url`, `url_api` (authenticated asset requests), `provenance` (which method verified it — SLSA / cosign / minisign / GitHub attestations), `signer` / `attested_by` (packslip identity), **`repository_ids = { repository = "<id>" }`** (revision 3 — the forge repository ID packslip pins), `checksum_source = "mise-versions"`, `install = "source"`, `additional_artifacts`, `conda_deps`, `size` (**legacy, read-only now**). `provenance_verified` is kept on unchanged artifacts but never read. Full field reference: `https://mise.jdx.dev/dev-tools/mise-lock-reference.html`.

**Backends exempt from strict URL-locking** (`--locked` does not require a resolved URL): `asdf`, `cargo`, `dotnet`, `gem`, `go`, `npm`, `pypi`/`pipx`, `spinel`, `spm`, `ubi`, vfox **backend** plugins, `core:dotnet`, `core:rust`, `core:swift`. URL-recording — `aqua`, `github`, `gitlab`, `forgejo`, `http`, `s3`, `packslip`, `conda` (URL + checksum + deps), the core archive tools (bun, deno, elixir, erlang, go, java, node, python, ruby, zig), and vfox **tool** plugins — must have a resolved URL for the current platform. spm records the bundle URL and checksum. Tool stubs follow the same rules via the project `mise.lock`.

**`--locked` requires checksums (2026.11.0, breaking for hand-written or old lockfiles).** In locked mode, a lock entry whose URL for the current platform (including additional GitHub release assets) has no checksum fails **before downloading**: `No lockfile checksum found … (--locked mode)` — hint: `Run mise lock to record checksums, or set locked_require_checksum = false`. `mise lock` fills missing checksums (downloading and hashing the artifact if the release publishes none). Backends that don't lock URLs (cargo, go, …) are unaffected; `lockfile_mode = "merge"` is exempt unless `locked_require_checksum = true`.

**Safe mode ignores lockfile URLs (2026.11.0).** Under `MISE_SAFE=1` a project `mise.lock` can't choose download URLs, even in a trusted project: mise downloads from the backend's own release source and still checks the locked checksum, and `--locked` accepts entries that have a checksum but no URL. `MISE_SAFE=1 MISE_LOCKED=1 mise install` works for node, python, and go (2026.11.2).

**Lockfile security (2026.10.19):** a `mise.lock` checksum is now **enforced** for erlang, npm, spm, and http; an **untrusted** lockfile can't switch a backend, add inline options, or supply download URLs (until its config is trusted it only pins versions and checksums); `mise lock` refuses URLs with credential-like query parameters; version strings containing template syntax are refused. A monorepo root at `.config/mise.toml` writes `.config/mise.lock` (2026.10.18). `mise lock` warns when a named platform has no build (2026.10.12).

**Runtime resolution:** when a project and global config both declare a tool, the **project's** lock entry wins (no fallback to the global pin). In locked mode a missing entry errors even for `mise which`. Locked installs don't update lockfiles. `mise upgrade node@24.11.0` / `mise lock node@X` rewrite `mise.toml` at the same precision when X is outside the request.

> A `provenance` field alone is **not proof the bytes were verified**. The lockfile is a trust input requiring review, not an automatic guarantee.

**Multiple entries per version** occur when artifact identity depends on more than the platform key — e.g. Swift's per-distro Linux tarballs. Entries match on options **exactly**, so a machine only verifies against its own distro's entry:
```toml
[[tools.swift]]
version = "6.3.1"
backend = "core:swift"
options = { swift_platform = "ubuntu24.04" }
```

```bash
mise lock                       # Update lockfile from current config (installs nothing)
mise lock node python           # Update specific tools only
mise lock --bump                # Advance fuzzy selectors (latest/lts/"20") without installing
mise lock --bump --dry-run --json
mise lock -p linux-x64,macos-arm64   # Add/update entries for specific platform(s)
mise lock --local               # Update mise.local.lock instead of mise.lock
mise lock -g                    # Target global config lockfiles
mise lock --minimum-release-age "30d"
mise lock node@22.15.0          # Pin a version in the lockfile without reinstalling
mise lock --sidecars [--json]   # 2026.9.18: list the native dependency sidecar dirs to commit (writes nothing)
mise lock --upgrade             # migrate to the newest lockfile revision (transactional)
```

`mise lock --bump` checks remote versions for every selector and **fails** when a list can't be fetched (2026.9.13 — it used to exit 0 with stale data). For bots, `MISE_SAFE=1 mise lock --bump --json` is recommended. `mise lock` also resolves **task tools** into the lockfile.

> `mise lock --bump` re-resolves fuzzy version selectors **without installing or touching `mise.toml`**; exact pins resolve to themselves. Use `mise upgrade --bump` to rewrite config pins. `--json` reports only *version-level* changes, so a plain `mise lock --json` typically prints `[]` while still refreshing checksums and URLs.

Per-config lockfiles pair with their config: `mise.toml`→`mise.lock`, `mise.test.toml`→`mise.test.lock`, `mise.local.toml`→`mise.local.lock` (gitignore the local ones). Scoping is strict, so CI that doesn't set `MISE_ENV` depends only on `mise.lock` and dev-tool bumps won't invalidate CI caches.

**Command × lockfile behavior:**

| Command | Installs | Updates `mise.toml` | Updates `mise.lock` |
|---------|----------|---------------------|---------------------|
| `mise use node@22` | Yes | Yes | Yes |
| `mise install` | Yes | No | Yes |
| `mise install node@22.15.0` | Yes | No | **No** (one-off, not config-driven) |
| `mise upgrade` | Yes | No | Yes |
| `mise upgrade --bump` | Yes | Yes | Yes |
| `mise lock` | No | No | Yes |
| `mise lock --bump` | No | No | Yes |

**Backend lockfile support by family:** download backends (aqua, github, gitlab, forgejo, http, s3) record per-platform artifact metadata (URL + checksum + provenance); `packslip` records signed artifact info plus signer and repository commitments; `conda` records URL, checksum, and `conda_deps`; `npm` (embedded aube) and `pypi` (uv) add revision-2 dependency-graph sidecars; other language package managers (cargo, gem, go, dotnet, spinel) lock top-level versions only; vfox tool plugins are URL-lockable; asdf and vfox backend plugins have no strict URL requirement. **Provenance support:** `aqua`, `github`, `core:python`, `core:ruby`, `core:zig`.

For the current platform, `mise lock` **downloads the artifact and performs full cryptographic verification at lock time**, so the entry is backed by real verification rather than registry metadata. Cross-platform entries record detected provenance without verifying. The old `github_attestations = "unavailable"` marker is **no longer written** and is ignored on read — mise probes again.

Settings: `lockfile` (unset ⇒ update existing lockfiles, never create new ones), `locked = true` (require lockfile-resolved URLs; blocks API calls — good for CI), `locked_require_checksum` (unset ⇒ on, except in merge mode; 2026.11.0, deprecated along with merge mode — warns from 2027.10.0), `locked_verify_provenance` (default `false`; auto-on with `paranoid`/`paranoid = ["provenance"]`), `lockfile_platforms`, `lockfile_mode` (**`generate`** default since 2026.10.19 \| `merge`, deprecated 2026.10.24), `lockfile_auto_prune` (default `true`), `locked_scopes` (default `["global","project","system"]`, global-only; `[tool_config] locked` applies even when `locked_scopes` excludes the scope).

**Lockfile revisions** (`lockfile_version`):

| Revision | Since | Adds |
|----------|-------|------|
| 0 | — | Unversioned legacy format |
| 1 | 2026.8.11 | Binds each original request to the entry it resolved, so `"1"` and `"1.0.0"` can lock different versions |
| 2 | 2026.9.7 | npm/pypi dependency-graph sidecars (see below) |
| 3 | 2026.9.16 | `repository_ids` — packslip pins by forge repository ID, so a **renamed** (2026.9.16) or **transferred** (2026.10.2) repo keeps installing with a one-time warning, while a *different* repo re-created under the same name is refused |
| **4** | **2026.11.1** | Tool-entry `provenance = { repository = … }` — the npm provenance repository pin |

🔴 **New and empty lockfiles are written as revision 4, which mise older than 2026.11.1 rejects** (`unsupported lockfile version 4`) — upgrade collaborators and CI before committing one. Existing lockfiles keep their revision during ordinary `lock`/`install`/`upgrade` (below revision 4 mise omits the provenance pin and warns); run **`mise lock --upgrade`** (transactional; rolls back on failure; takes no tool arguments) to migrate — to add pins to an older lockfile, `mise lock --upgrade` then `mise install --force`. Since 2026.9.17 mise warns when a lockfile's format was superseded more than 6 months ago — revision 0 after 2027-02-23, revision 1 after 2027-03-13, revision 2 after 2027-03-27, revision 3 after 2027-05-03.

#### Dependency-Graph Sidecars (revision 2+, 2026.9.7)

Revision 2 records the **full transitive dependency graph** of `npm:` tools (via embedded aube) and `pypi:` tools (via uv), then replays it with a strict frozen install — so two projects on the same top-level version can each keep their own reviewed graph.

Graphs live in **native sidecar files** (`uv.lock` / `aube-lock.yaml` plus a manifest) under `.mise/locks/<backend-tool>/<version>/`, referenced from `mise.lock` by relative path and SHA-256 digest, so the lockfile itself stays small.

```bash
mise lock --upgrade            # move an existing lockfile to the newest revision and resolve graphs
mise install --locked          # replay the recorded graphs
mise lock --bump pypi:black    # refresh Black's transitive deps without changing its version
uv tree --project .mise/locks/pypi-black/24.10.0    # inspect a sidecar directly
```

- **Commit the sidecar directory alongside `mise.lock`.** If you gitignore `mise.local.lock`, also ignore its sidecar subdirectory (e.g. `.mise/locks/mise.local/`).
- Sidecar locations follow the lockfile (`.mise/locks/`, `.config/mise/locks/`) or a symlink target; digests normalize CRLF. `mise lock --sidecars` lists what to commit.
- Ordinary `mise install` validates and accepts hand-edited sidecars; `mise install --locked` **rejects digest mismatches** until you run `mise lock`.
- Python graph locking needs **uv ≥ 0.12.10** and published **wheels** for the target platform — explicit `mise lock` and locked installs never build sdists. Git sources, standalone pipx installs, and free-form `uvx_args`/`pipx_args` stay **version-only**.
- `mise lock` resolves against the tool's selected Python whenever mise has it installed, even off `PATH`; for a per-tool `python` that isn't installed yet, uv may download one just for resolution (not with `offline`) (2026.10.24).
- When the tool's own release has no upload time (common on self-hosted indexes), `mise lock` exempts the tool from the `minimum_release_age` cutoff while its dependencies keep it. A graph that can't be locked, or that `uv sync --frozen --no-build` says isn't installable (2026.10.21), produces a warning and a version-only entry.

> 🔴 **Revision 2+ is not readable by older mise versions.** Revision 2+ `--locked` installs fail if a recorded graph is missing or its digest doesn't match.

> ✅ **Fixed 2026.9.7:** installing from a committed `mise.lock` no longer fails when the locked release is younger than `minimum_release_age`. The cutoff still applies to unlocked fuzzy requests and to generating/bumping a lockfile, so a reviewed lock entry reproduces immediately in CI.

> ⚠️ **Settings are global in scope.** `locked = true` in a project's `mise.toml` applies to *all* tool resolution, including tools from `~/.config/mise/config.toml`. Fix warnings about global tools with `mise lock -g`.
>
> ✅ **Use `[tool_config]` for project-scoped strictness instead (2026.8.6+).** This is a top-level config key — not a setting — and applies only to tools declared by configs sharing that config root, without forcing global or parent-root tools into strict mode:
> ```toml
> [tool_config]
> locked = true
> ```
> `[settings] locked`, the `--locked` flag, and `MISE_LOCKED` remain invocation-wide.

`minimum_release_age` (env `MISE_MINIMUM_RELEASE_AGE`) filters fuzzy version requests by release date — supply-chain delay protection. Accepts relative durations (`7d`, `90d`, `6mo`, `1y`) and absolute dates (`2024-06-01`, `2024-06-01T12:00:00Z`); `"0s"` disables it. A version **without a release date** isn't held back, but `install`/`upgrade`/`use` print a one-time warning that the cutoff couldn't be checked (exact pins and `env`/`ls`/`outdated`/`ls-remote` don't). Only **`npm:` and `pypi:`/`pipx:`** forward the cutoff into transitive dependency resolution. Exempt tools with `minimum_release_age_excludes = ["trivy", "npm:*"]` (backend wildcards, registry shorthands, or full IDs; lists **merge across configs**, while `MISE_MINIMUM_RELEASE_AGE_EXCLUDES` replaces), or per-tool:
```toml
[tools.trivy]
minimum_release_age = "1d"
```
Note durations use jiff format — months are `6mo`, not `6m`. The deprecated `install_before` setting maps to this (**warning live since 2026.10.0**, removed 2027.10.0). The cutoff is also overridable per command with `--minimum-release-age` on `use`, `install`, `upgrade`, `ls-remote`, `latest`, and `lock`.

> **mise itself now honors a release age (2026.9.17).** Unpinned `mise self-update`, automatic updates, update notifications, and the `curl https://mise.run | sh` installer pick the newest stable release **at least 24h old**. Precedence: `self-update --minimum-release-age <DUR>` → `self_update.minimum_release_age` (`MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE`; optional, no default of its own) → `minimum_release_age` → `24h`. Because it inherits `minimum_release_age`, a `7d` tool cutoff also delays mise upgrades by 7 days. Explicit versions bypass it (`--force` does not). The installer reads env vars only (`MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE` → `MISE_MINIMUM_RELEASE_AGE` → `24h`); set `MISE_VERSION` for reproducible installs. Self-update also requires a **signed packslip** for releases ≥ 2026.9.3.

> 🔴 **There IS a built-in 24h default — the setting being "unset" does not mean "off."** This is a subtle trap. `mise settings get minimum_release_age` reports *not set* and the JSON schema carries **no `default`**, because `settings.toml` declares `default_docs = "24h"` (a docs-display value) rather than `default`. But mise's source defines `DEFAULT_MINIMUM_RELEASE_AGE = "24h"` and applies it whenever the setting is unset — and since **2026.10.15 it covers every backend** (previously only timestamp-reporting ones). More backends now report dates: conda, aqua `github_tag` packages, core dotnet/deno/erlang/python/ruby/rust, dart/flutter, http via `version_date_expr`, and vfox plugins that return `created_at`. A tool whose **only** version is under 24h old resolves to "no versions found".
>
> The `go` backend dates only the newest 100 (proxy) / 10 (`go list`) versions, checks undated ones individually, and **warns and allows** a version if the date lookup fails. Set `minimum_release_age = "0s"` to genuinely disable it, or a longer duration to harden. An already-installed matching version is never deactivated; `--minimum-release-age 0s` lets you take one release early, and a later upgrade won't downgrade it. A fresh release being invisible for 24h is expected behavior, not a bug.
>
> **Never filtered regardless:** an explicit pin (`hk = "2.0.1"`) and a lockfile-resolved version — both are decisions already taken and committed. Since 2026.9.7, installing from a committed `mise.lock` no longer fails when the locked release is younger than the cutoff.

CI flag: `--locked` (global flag on any command) errors if any version isn't resolved via the lockfile.

### Auto-Install Controls

All are enabled by default and all require the master `auto_install`:

- `auto_install` (default `true`) — master switch
- `exec_auto_install` (default `true`) — `mise x`/`mise r`
- `task.run_auto_install` (default `true`)
- `not_found_auto_install` (default `true`) — maps command → tool via registry `bins` metadata, so it covers **configured tools even if never installed**; not raw backend specs (`cargo:x`, `ubi:x`)
- `not_found_auto_install_registry` (default **`false`**, 2026.9.17) — for an **unconfigured** command: if **exactly one** enabled registry tool provides it, install its latest version and add it to the **global** config (like `mise use --global`). Ambiguous matches are skipped
- `auto_install_disable_tools = ["..."]` — per-tool skip list

### Automatic Tool Updates

Added **2026.10.4** (background prompt updates 2026.10.7; installed-only 2026.10.9). Keeps **global** tools current within their request — aimed at fast-moving CLIs like coding agents (docs: `https://mise.jdx.dev/dev-tools/self-updating-tools.html`).

```toml
# ~/.config/mise/config.toml — GLOBAL (or system) config only
[settings]
tool_update.global_auto = "12h"     # bool | duration; every global tool (default false)
tool_update.check_duration = "24h"  # interval for `auto_update = true` (min 1h)

[tools]
claude = { version = "latest", auto_update = true }
node   = { version = "24", auto_update = "6h" }      # its own interval wins
python = { version = "3.13", auto_update = false }   # opt out of global_auto
```

- **How it runs:** a shim or `mise x` launch runs `mise upgrade <tool>` before starting it; under `mise activate` the update starts **detached in the background at the prompt** (output discarded; failures show in `mise doctor`).
- **Limits:** stays within the request (exact pins never move); only already-installed tools; `minimum_release_age` applies (so an update may lag a release by 24–48h); never offline, in CI, under `locked`, from tasks, from hook-env, or from project configs. A global lockfile entry moves to the new version; project config and lockfiles are never touched.
- **Scheduled service** (checks hourly; launches/prompts then stop updating on their own):
  ```toml
  [bootstrap.services.mise-tool-update]
  builtin = "tool-update"            # then: mise bootstrap services apply
  ```
- **Self-updating tools** can detect a mise install via `.mise.backend.toml` (`short`) or `.mise-install.toml` (`requested_as`), check with `mise outdated --global <tool> --json`, update with `mise upgrade --global <tool>`, and switch a running session with `exec mise x -- <tool> <resume args>`.

> Not to be confused with **mise's own** self-update: `self_update.auto` (bool or interval, renamed from `auto_update` in 2026.10.4) and `self_update.check_duration` (default `7d`).

### Install Layout, Download Cache, and the `global` Link

**Identity install layout (experimental, 2026.10.4)** — docs: `https://mise.jdx.dev/dev-tools/install-layout.html`:

```toml
[settings]
experimental = true
install_layout = "identity"   # legacy (default) | identity; env MISE_INSTALL_LAYOUT
```

- Each install goes to `installs/<label>-<hash>/` (label = last part of the backend project name, ≤24 chars; hash = 8 base32 chars); `installs/<tool>/<version>` stays as a link. Each install has a `.mise-install.toml` receipt (`digest`, `dir`, `requested_as`, `[identity]`); the catalog lives in `installs/.mise/`.
- Shorthand and full ID **share one install** (`age` = `aqua:FiloSottile/age`). Variants that differ by platform, an install-affecting option (e.g. `matching`), `install_env` (hashed, never recorded), a pinned checksum, or a dependency graph **coexist**; `prerelease`, `depends`, `minimum_release_age`, `version_order`, `auto_update`, and `postinstall` don't split installs.
- **Windows:** `experimental = true` alone turns the identity layout on (2026.10.10) — set `install_layout = "legacy"` to opt out. Installs go to `%LOCALAPPDATA%\mise\i\` with junction links; `MISE_INSTALL_STORE_DIR` picks the store on any platform.
- Stays legacy: `http:`, `rust`, `dotnet`, `--system`, `--shared`, `install-into`, `mise link`, `path:`.

```bash
mise installs ls [-J] [--no-header] [TOOL]…    # Status: selected | pinned | shared  (default subcommand)
mise installs select <dir|path>                 # which install unlocked requests use (machine-wide)
mise installs migrate [-n] [TOOL@VERSION]…      # reinstall legacy installs into the new layout
```
`mise uninstall` asks for `--all` when several variants exist; `mise ls` shows `1.2.1 [age-hlencrst]`.

**Stable `global` link for IDEs (2026.10.8, always on):** `~/.local/share/mise/installs/<tool>/global -> ./<version>` points at the version **global and system config** select. Project config and `MISE_*_VERSION` don't change it, so an IDE setting that needs a fixed path (Maven home, JDK home) can point at it. It is rebuilt with the `latest`/major links (e.g. after `mise use -g` or `mise install`); tools no global config selects have no link, and a real directory named `global` is left alone.

**Verified download cache (2026.10.7):** finished downloads are stored in `$MISE_CACHE_DIR/downloads-cas` and **re-hashed before reuse**; requests send ETag/Last-Modified so a `304` reuses the file; tools with a pinned checksum (tool option, `mise.lock`, packslip digest) skip the network entirely (http, github, aqua, packslip, precompiled core tools). Under `--locked`/paranoid only pinned-checksum entries are reused. Settings: `download_cache` (default `true`; `MISE_DOWNLOAD_CACHE=0` off), `download_cache_max_size` (default `2GiB`, `0` = unlimited, LRU). Supersedes `always_keep_download` (deprecated; warns 2026.11.0, removed 2027.11.0).

**Extraction guard (2026.10.19):** `extract_max_ratio` (global-only, default `100`, `0` disables) fails an extraction that exceeds 256 MiB **and** is more than ratio × the download size; HTTP bodies are capped at 128 MiB. **`prune.exclude`** (2026.10.4) names tools never removed by `prune`, `ls --prunable`, or post-upgrade cleanup; prune also keeps versions used by running processes (for a process started through a runtime link such as `installs/claude/latest`, only the version it runs — 2026.10.22) and versions needed by tasks from `task_config.includes`, default task dirs, and `#MISE tools=` scripts (2026.10.24).

### CLI Commands for Tools

```bash
mise use node@22            # Install + activate + write to mise.toml
mise use -g node@22         # Write to global config
mise use --pin node@22      # Pin exact resolved version (e.g. 22.5.1)
mise use -e staging node@22 # Write to mise.staging.toml (subcommand -e/--env; the GLOBAL -E only selects
                            # the env for loading, so `mise -E staging use` still writes mise.toml)
mise use --fuzzy node@22    # Keep the fuzzy request (default unless MISE_PIN=1)
mise use --file mise.local.toml node@22   # --file and --path are interchangeable (2026.8.1+)
mise use --postinstall 'corepack enable' node@22   # 2026.8.15+ set a per-tool postinstall
mise use --tool-option matching=oxlint github:oxc-project/oxc  # 2026.9.2+, repeatable
mise use --remove python    # Remove a tool from config
mise use --dry-run-code     # Exit 1 if changes would be made (also on install, prune, uninstall, upgrade)
mise use --override node@24 # 2026.10.14: write ~/.config/mise/config.override.toml (beats ALL project config)
mise install node@20        # Install without activation
mise install                # Install all configured tools
mise install -f "gem:*"     # Force reinstall pattern
mise install --monorepo     # Install across all monorepo config roots
mise install --include-task-tools   # Install every task's [tools] without running tasks (+ --monorepo)
mise install --shared <DIR> # Install into a shared directory
mise install --system       # Install to the system data dir. On Unix (2026.9.12+) mise downloads,
                            # verifies, and unpacks AS THE INVOKING USER, then uses sudo only to
                            # publish into the system install/shim dirs. Supports relocatable tools
                            # from aqua, github, gitlab, forgejo, http, s3 without a tool postinstall.
                            # system_packages.sudo = false disables elevation.
mise ls                     # List installed
mise ls --current           # Active versions only
mise ls --prunable          # Tools eligible for prune
mise ls --outdated          # Tools with newer versions
mise ls --monorepo          # Across monorepo config roots
mise ls -b aqua --grouped   # 2026.9.13: filter by backend (repeatable; works with --json) /
                            # one section per backend (not with --json)
mise ls -m / --missing      # Configured but not installed (also --all-sources, --prefix, -i, -g)
mise ls --no-global         # 2026.10.22: skip global/system config (--local / -l are deprecated aliases)
mise ls --project           # 2026.10.22: only the nearest directory with a mise config
mise ls-remote node         # List available versions
mise ls-remote --prerelease # Include pre-releases
mise ls-remote --all node   # Every version, ignoring filters; -J includes created_at
mise ls-remote --no-versions-host node  # Bypass mise-versions (--strict-metadata needs --json too)
mise latest node            # Latest available version (no install); accepts prefix: (2026.10.5)
mise which node             # Show real binary path (-t/--tool <TOOL@VER>, --plugin, --version)
mise where node@22          # Show install directory
mise bin-paths              # List active runtime bin paths
mise tool <TOOL>            # Backend/description/config-source/tool-options for one tool (--url: registry URL)
mise uninstall node@20      # Remove an installed tool version (matches 20, 20.x, 20-…; never 200)
mise uninstall --project --dry-run  # 2026.10.22: remove every version the nearest project's
                            # config files name (installs are shared; path:/system skipped)
mise shell node@22          # Set MISE_NODE_VERSION in this shell; --unset --all clears every
                            # MISE_<TOOL>_VERSION (2026.10.22)
mise unuse node@20          # Remove one version from config (keeps siblings); prunes ONLY that version
                            # (2026.10.18); --no-prune keeps it installed
mise link node@custom ./dir # Symlink an external install into mise
mise x python@3.12 -- script.py  # Run with specific tool
mise reshim                 # Rebuild shims
mise registry               # List all available tools
mise backends ls            # List available backends (`mise b` alias deprecated, removed 2027.4.0)
mise backends switch [-n] [-g] [TOOL@VER]  # 2026.9.13: move locked tools to the registry's new backend
mise fmt                    # Format mise.toml (sort keys, clean whitespace)
mise outdated               # Check for updates (--global, --no-global, --inactive, -J); lists tools
                            # whose version lookup failed instead of "All tools are up to date"
mise upgrade                # Update versions (respects mise.toml ranges)
mise upgrade --global       # 2026.10.7: only global/system requests, even inside a project
                            # (as if run outside any project; --bump writes global config)
mise upgrade --bump         # Bump mise.toml to absolute latest (keeps precision)
mise upgrade --bump --pin   # 2026.10.18: write exact resolved versions (also when pin/MISE_PIN=1)
mise upgrade -b             # 2026.8.6+ shorthand for --bump (old -l deprecated, removed 2027.8.5)
mise upgrade --no-prune     # Keep the replaced version indefinitely (2026.8.1+)
mise upgrade --prune        # Force IMMEDIATE removal, skipping the upgrade.prune_after grace period
mise upgrade -i             # Interactive selection
mise upgrade -x node        # --exclude a tool; also --inactive, --no-global
mise upgrade node@latest --bump   # 2026.9.13: persists the selector `latest`, not a concrete version
mise install --force        # 2026.8.4+ works with no tool args: reinstall every configured tool
mise lock --upgrade         # Migrate a lockfile to the newest revision (3)
mise prune --dry-run        # Explains WHY each version is prunable (2026.8.11+)
mise run --all              # Interactive picker across the whole monorepo (2026.8.6+)
mise prune                  # Remove unused versions (destructive; --configs also prunes stale links;
                            # honors prune.exclude)
mise lock                   # Update lockfile checksums/URLs
mise search <query>         # Search registry (-m equal|contains|fuzzy; -a/--all; npm:/cargo:/gem:/dotnet: prefixes)
mise cache clear            # Clear cached downloads
mise cache task <task>      # Inspect a task's cached artifacts
mise sync node --nvm        # Import versions from nvm
mise sync python --uv       # Import versions from uv
mise sync ruby --brew       # Import versions from brew
mise token github           # Show resolved host token (--unmask to reveal)
mise generate install-script -w ./bin/mise  # Self-contained install script for CI
                            # renamed from `mise generate bootstrap` in 2026.8.11;
                            # old spelling is a hidden alias, removed 2027.9.0
mise generate task-stubs    # Generate task stub wrappers (bin/)
mise generate task-docs     # Generate markdown docs for tasks
mise generate config        # Generate sample config
mise generate github-action # Generate sample GitHub Action workflow
mise generate devcontainer  # Generate devcontainer spec
mise generate git-pre-commit # Generate pre-commit hook (staged files exposed as STAGED)
mise generate tool-stub     # Generate a standalone tool stub
mise install-into <tool> <path>  # Install a tool into a specific path
mise doctor                 # Diagnose installation issues (mise doctor path)
mise doctor project         # Run the project's [doctor.checks] diagnostics (--json)
mise self-update            # Update mise — newest stable release ≥24h old (--minimum-release-age <DUR>)
mise mcp                    # Run mise as a Model Context Protocol (MCP) server
mise bootstrap              # Provision a whole machine (alias: bs)
mise oci build|push|run     # Build/inspect OCI container images with mise tools
mise deps add|install|remove  # Project dependency preparation (alias: prepare)
mise en                     # Spawn a sub-shell with the project env loaded
mise test-tool <tool>       # Verify a tool installs/builds correctly
mise implode                # Remove mise entirely

# New in 2026.10.x
mise installs ls|select|migrate   # [experimental] identity install layout (2026.10.4)
mise lsp                    # Language server for mise config files (2026.10.18)
mise config explain [KEY]   # Explain each key in a config file (2026.10.14)
mise vars [ls|prompt|unset] # Per-machine [vars] answers (2026.10.6)
mise secrets [ls]           # [experimental] fnox secrets a project grants (2026.10.4)
mise env --paths            # PATH entries mise manages, in order (2026.10.11)

# New command groups (2026.9.x)
mise daemons                # [experimental] Manage project daemons via pitchfork (alias: daemon)
mise dotfiles | mise dot    # Manage [dotfiles] + the whole history system
mise skills ls|sync         # Agent skills the active tools ship, from their packslips
mise packslip pins|forget   # Signers mise accepts packslips from
mise ssh [DEST] [-- CMD]    # SSH session, optionally borrowing read-only GitHub access
mise tool-stub <file>       # Execute a tool stub
mise install --include-lazy # Also provision tools declared lazy = true
mise reshim --system        # Rebuild system-scoped shims
mise lock --sidecars        # List dependency-graph sidecar dirs to commit (2026.9.18)
mise bootstrap packages export --format nix   # Emit a NixOS module from nix: declarations
mise bootstrap packages use --no-install      # Record declarations without touching managers
mise patrons | mise sponsors  # Project supporters
```

**`mise ssh` GitHub relay flags:** `--github-relay-read-only` (borrow read-only GitHub access for this session only), `--github-relay-repo <OWNER/REPO>` (repeatable allowlist), `--github-relay-all-repos`, `--github-relay-log-requests` / `--github-relay-no-log-requests`, `--github-relay-log-format <text|jsonl>`, `--github-relay-max-duration <DUR>` (`0s` = session lifetime). Tuned by the `github_relay.*` settings.

`mise use` writes to the **lowest-precedence file in the highest-precedence directory** (so `mise.toml`, not `mise.local.toml`). Target order: `--global` → `--override` → `--path`/`--file` → `--env` → `MISE_DEFAULT_CONFIG_FILENAME` → first of `MISE_OVERRIDE_CONFIG_FILENAMES` → `mise.toml`. With `--global`, new entries go to `write_targets.tools` when set (global-only setting, 2026.10.4; relative/templated paths since 2026.10.16 — e.g. `"{{ config_source | dirname }}/10-tools.toml"`); `write_targets.packages` / `write_targets.dotfiles` do the same for `bootstrap packages use|import --global` and `mise dot add|edit`.

> Since 2026.8.0, `--path <dir>` correctly targets a config *inside that directory* for `use`, `unuse`, `set`, `unset`, `dotfiles add`, and the `system` subcommands — previously it was silently discarded when the cwd already had a config in scope, so `mise unuse --path ../other` could remove a tool from the wrong file.

> Since 2026.8.0, `mise unuse node@20` on `node = ["20", "22"]` removes **only the matching version**, preserving the rest (including structured options and `.tool-versions` entries). The whole key is removed only for an unversioned request or after the last version is unused.
