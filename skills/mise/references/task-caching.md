## Task Output Caching (Experimental)

**Contents**

- [Local Artifact Cache](#local-artifact-cache)
- [Cache Inputs](#cache-inputs)
- [Remote Task Cache](#remote-task-cache)
- [Rust Compiler Action Cache — REMOVED (rust_cache is a no-op)](#rust-compiler-action-cache--removed-rust_cache-is-a-no-op)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

Introduced in **v2026.7.15** and expanded through **v2026.8.1**. Distinct from freshness checking: instead of merely *skipping* a task, mise **restores its declared outputs and replays its stdout/stderr** from a content-addressed archive. Requires `experimental = true`.

> Docs: `https://mise.jdx.dev/tasks/caching.html`. Artifacts live in `$MISE_CACHE_DIR/task-artifacts/v2`. Unsupported outputs: `{ auto = true }`, absolute paths, or paths escaping the task dir. `raw`/`interactive` tasks bypass the artifact cache. `command_inputs` must exit 0 with non-empty output (≤16 MiB), inherit the task timeout (or 30s), and don't run during dry runs.

### Local Artifact Cache

Requires `sources` plus either explicit `outputs` or `outputs = []`.

```toml
[tasks.build]
run = "cargo build --release"
sources = ["src/**/*.rs", "Cargo.toml"]
outputs = ["target/release/myapp"]
cache = { enabled = true }

# Result-only caching for checks that produce no files (lint/test/typecheck)
[tasks.lint]
run = "cargo clippy"
sources = ["src/**/*.rs"]
outputs = []                 # caches the successful result + replayable logs, no archive
cache = { enabled = true }
```

| `cache` key | Type | Default | Description |
|-------------|------|---------|-------------|
| `enabled` | bool | `false` | Opt in |
| `env` | string[] | `[]` | Ambient env var names folded into the cache key |
| `command_inputs` | string[] | `[]` | Commands whose text + stdout/stderr hash into the key (2026.7.16+) |
| `audit` | bool | `false` | Report undeclared reads/writes (strace; Linux only) |

**Cache key** = source contents + task config + args + the task's own `env` values + all visible `[vars]` and task `vars` + declared/allowlisted ambient env (`cache.env`, `global_env`) + resolved tools + OS + arch + whether the task runs as `depends_post`, plus the artifact keys of upstream dependencies. Other ambient/`[env]` variables count only when listed in `cache.env`. Source paths are relative to the outermost config root. Cache failures degrade to misses or warnings — never a task failure. Artifacts are checksum-verified on restore. Tasks that list `secrets` **skip the artifact cache** (freshness still applies).

**Cache-miss reasons** (printed after `cache miss:`): `no matching cache entry`, `cache entry was corrupt`, `cache entry exceeded its age limit`, `forced execution`, `cache reads are disabled`, `dependency completed without a cache key`. `cache.enabled` on an ineligible task errors (e.g. `task build cache requires at least one source`). `mise cache task` / `mise cache clear --task` refuse to run without `experimental`.

```toml
[tasks.build]
run = "cargo build --release"
sources = ["src/**/*.rs"]
outputs = ["dist/app"]
cache = { enabled = true, env = ["CI"], command_inputs = ["rustc --version"] }
```

**Per-run control and inspection:**
```bash
mise run --task-cache off build          # read-write (default) | read-only | write-only | off | local-only
mise run --task-cache-explain build      # structural breakdown of the key; works with --dry-run
mise run --dry-run --task-cache-explain-json build # JSON Lines; --dry-run is required
mise run --task-cache-stats build
mise cache task build                    # stored size, restorable bytes, saved time, last access, outputs
mise cache task build --json
mise cache clear --task build            # leaves working-tree outputs untouched
```

`--task-cache-explain` reports input categories, counts, env/var names and presence, and platform — and deliberately emits **no secret-derived hashes**.

**Size and age caps** (2026.8.1+), independent of `cache_prune_age`:
```toml
[settings]
task.cache_dir = "~/.cache/mise/task-artifacts"
task.cache_max_size = "2GiB"    # LRU eviction after writes
task.cache_max_age = "30d"      # expired entries rejected on restore
```

### Cache Inputs

Reusable and global input declarations live in `[task_config]`:

```toml
[task_config]
global_inputs = ["mise.toml", "@group:lockfiles"]
global_env = ["CI", "NODE_ENV"]
global_pass_through_env = ["GITHUB_TOKEN"]   # available under deny_env, NOT part of the key

[task_config.input_groups]
lockfiles = ["package-lock.json", "pnpm-lock.yaml"]
rust = ["Cargo.toml", "Cargo.lock", "@group:lockfiles"]   # groups nest

[tasks.build]
sources = ["src/**/*.rs", "@group:rust"]
outputs = ["dist/app"]
cache = { enabled = true }
```

`pass_through_env` is also a per-task field. Use it for tokens: they stay available to the command (even under `deny_env`) without making every token rotation a cache miss.

### Remote Task Cache

Added **v2026.8.1** (docs: `https://mise.jdx.dev/tasks/remote-cache.html`). A composite store: reads local first, promotes remote hits, and commits locally **then** mirrors writes so a remote failure never loses a local hit. Requests are hardened, verified, and streamed. Requests **carrying credentials** require HTTPS outside loopback (`localhost`, `127.0.0.0/8`, `::1`); an unauthenticated HTTP endpoint is permitted with a warning. An unreachable server only warns and falls back to the local cache.

**Server:** the reference implementation is **`mbx-cache`** (`cargo install --git https://github.com/jdx/mr-boxington-cache`). Since 2026.10.5 mise sends `mbx-cache-*` headers, which old `mise-cache` 0.1 servers reject.

> 🔴 **The action description is uploaded verbatim** — task `env` values, `cache.env`/`global_env` values, `vars`, and `command_inputs` text. `redact`/`redactions` do **not** apply to it. Never put secrets in those; use `pass_through_env` (secret values themselves are kept out of remote cache keys since 2026.10.5).

> 🔴 **Renamed:** these settings moved from `task.cache_remote_*` to nested **`task.cache.remote_*`**. The flat spellings are hidden compatibility aliases — they start warning in **2027.2.0** and are removed in **2027.8.0**. Use the nested form.

| Setting | Env | Notes |
|---------|-----|-------|
| `task.cache.remote_url` | `MISE_TASK_CACHE_REMOTE_URL` | Endpoint |
| `task.cache.remote_namespace` | `MISE_TASK_CACHE_REMOTE_NAMESPACE` | **Required when `remote_url` is set**; opaque org/repo namespace isolating entries |
| `task.cache.remote_mode` | `MISE_TASK_CACHE_REMOTE_MODE` | `read-write` (default), `read-only`, `write-only` |
| `task.cache.remote_token` | `MISE_TASK_CACHE_REMOTE_TOKEN` | `Authorization: Bearer`. **Global-config only.** |
| `task.cache.remote_token_file` | `MISE_TASK_CACHE_REMOTE_TOKEN_FILE` | Re-read before each request (rotating creds, K8s projected SA tokens). **Global-config only.** |
| `task.cache.remote_oidc_audience` | `MISE_TASK_CACHE_REMOTE_OIDC_AUDIENCE` | GitHub Actions OIDC. **Global-config only.** |
| `task.cache.audit_report` | `MISE_TASK_CACHE_AUDIT_REPORT` | **2026.8.6+.** JSON Lines file capturing *every* undeclared read/write (`{"task","kind","path"}`). Console still caps at 20 paths/task. Truncated once per invocation, appended by later audited tasks. |
| `task.cache.stats_report` | `MISE_TASK_CACHE_STATS_REPORT` | 🔴 **Deprecated no-op** since 2026.10.5 (removed 2027.10.5) — the built-in action cache it reported on is gone. |

**Credential precedence:** explicit bearer token → global-only token file → GitHub Actions OIDC. The credential settings are global-only so a shared project config cannot supply them.

> **Remote writes are policy-restricted:** only protected-branch push pipelines in GitHub Actions and GitLab CI may write. Pull requests, tags, unprotected branches, other CI systems, and local developer runs are read-only (a `write-only` remote mode disables the remote entirely in those contexts). The server is expected to enforce the same policy independently from verified OIDC claims.

Protocol reference: `https://mise.jdx.dev/tasks/remote-cache-protocol.html` (protocol version 1).

### Rust Compiler Action Cache — REMOVED (`rust_cache` is a no-op)

> 🔴 **`rust_cache` is a deprecated NO-OP. Do not use it.** The JSON schema marks the field `"deprecated": true` with the description *"deprecated no-op; use mbx for Rust action caching instead."* Setting `rust_cache = true` on a task or in `[task_config]` does **nothing** — it neither errors nor caches. Removal is scheduled for **2027.8.14**.

**Migrate to [mbx](https://mr-boxington.jdx.dev/getting-started)** (Mr. Boxington), the separate `jdx` project that now owns Rust action caching:

```toml
# ❌ no longer does anything
[tasks.build]
rust_cache = true

# ✅ use mbx instead — e.g. as a command wrapper
[wrappers.cargo]
command = "mbx"
env = { MBX_CARGO_SHIM_MODE = "1" }
```

The task **output** cache ([Local Artifact Cache](#local-artifact-cache)) is unaffected and still works — it caches whole task results, not individual rustc invocations.
