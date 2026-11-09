## Task Dependencies and Freshness

**Contents**

- [Dependencies](#dependencies)
- [Freshness with sources/outputs](#freshness-with-sourcesoutputs)
- [Conditional Skip (skip_if)](#conditional-skip-skip_if)
- [Redactions (Experimental)](#redactions-experimental)
- [Error Handling](#error-handling)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### Dependencies

```toml
[tasks.deploy]
depends = ["build", "test", "lint"]  # Run before (parallel by default)
depends_post = ["notify"]             # Run after (even on failure)
wait_for = ["db:migrate"]             # Wait if running, don't add
```

If a dependency fails, the dependent task skips execution. Shared dependencies run once.

### Freshness with sources/outputs

```toml
[tasks.build]
sources = ["Cargo.toml", "src/**/*.rs"]
outputs = ["target/release/myapp"]
run = "cargo build --release"
# Skips if sources unchanged and outputs exist
```

A task is fresh when output mtime is newer than the newest source. The config file that defines the task is an implicit source (editing it — even to change a different task — makes the task stale), as is the script its `file` names, missing declared outputs force a run, and relative entries resolve from the task `dir` (`..` allowed).

**Staleness also triggers when** (caching docs, 2026.10.x): the recorded list of source files (with each file's size and mtime) differs from the last successful run — so deleting, adding, or restoring an older copy re-runs it; any source has mtime 0 (1970-01-01); or, in content-hash mode, the outputs changed (same-size edits that restore the mtime are rehashed, 2026.10.17). A failed run records nothing. `--force` also re-runs the task's dependencies. A `sources` glob that matches nothing **warns** (`task build has sources defined but no matching files found`, 2026.10.5; the artifact cache adds `; artifact caching disabled`) and the task still runs.

> ⚠️ **`global_inputs` gotcha (verified):** a task with **no `sources` of its own** still receives `task_config.global_inputs` plus `outputs = { auto = true }` — so it is **skipped** while those global inputs are unchanged. Give such tasks explicit `sources`, or run with `--force`.

> ⚠️ **Freshness does NOT respect `.gitignore`.** A change to a gitignored file matched by `sources` re-runs the task (verified). Only `mise watch` honors VCS ignores by default (override per task with `watch = { no_vcs_ignore = true }`).

**Exclusions** use gitignore-style `!` prefixes (escape a literal `!` path with `\!`); later entries override earlier ones. Since 2026.7.15, `outputs` supports the same ordered exclusions and re-inclusions:
```toml
sources = ["src/**/*.ts", "!src/**/*.test.ts", "!src/**/*.spec.ts"]
outputs = ["dist/**", "!dist/*.map"]
```
Excluded output paths are omitted from freshness hashes and cache archives; existing excluded files are preserved on restore.

Brace globs work in both (2026.8.0+): `sources = ["{src,lib}/**/*.ts"]`.

**Templated sources/outputs (2026.9.5+):** both accept `{{usage.*}}`, so freshness can be scoped per invocation:
```toml
[tasks.build]
usage = 'arg "<target>"'
sources = ["src/{{usage.target}}/**/*.ts"]
outputs = ["dist/{{usage.target}}/**"]
run = "tsc --build {{usage.target}}"
```

**Auto outputs:**
```toml
[tasks.build]
sources = ["src/**/*.rs"]
outputs = { auto = true }  # Implicit tracking via task hash (state in ~/.local/state/mise/task-outputs/<hash>)
run = "cargo build"
```

**Dependency invalidation:** when a depended-on task re-runs because *its* sources changed, the dependent task also re-runs even if its own sources are unchanged.

Enable content-hash (blake3) checking instead of mtime via `task.source_freshness_hash_contents = true` (more accurate, slower). `task.source_freshness_equal_mtime_is_fresh = true` treats equal source/output mtime as fresh.

### Conditional Skip (`skip_if`)

Added **2026.10.10**. Run a probe command; if it **exits 0** the task is skipped and counts as success. Unlike `sources`/`outputs`, it asks the world ("does the workspace already exist?") rather than comparing file times.

```toml
[tasks.setup]
skip_if = "examplectl workspace show"   # exit 0 ⇒ already set up ⇒ skip
run = "examplectl workspace init"

[tasks.deploy]
usage = 'arg "<env>"'
depends = ["build"]
skip_if = "./scripts/is-deployed.sh {{usage.env}}"   # Tera + usage.* allowed
run = "./scripts/deploy.sh {{usage.env}}"
```

Order: `depends` → `skip_if` → `confirm` → `run`. The probe runs in the task's dir/shell/env/sandbox, with no stdin, no `secrets`, and output discarded; it shares the task's `timeout` (if it can't start or times out, the task **fails**). `--force` bypasses it; `--dry-run` prints `skip_if: … (not checked: dry run)`. A skipped task does not invalidate dependents. File tasks: `#MISE skip_if="…"`.

### Redactions (Experimental)

Hide sensitive values from task output:

```toml
redactions = ["API_KEY", "PASSWORD", "SECRETS_*"]
```

Redactions intercept task output line-by-line; tasks with `raw = true` bypass them. A variable marked `redact = false` opts **out** of matching `redactions` patterns (2026.8.0+), so a short non-sensitive value no longer pollutes the global scrubber.

**CI integration (GitHub Actions)** — mask redacted values safely (never `for v in $(…)`, which splits on whitespace):
```bash
mise env --redacted --json | jq -r '.[] | select(length > 0)
  | "::add-mask::" + (gsub("%"; "%25") | gsub("\r"; "%0D") | gsub("\n"; "%0A"))'
```

`jdx/mise-action@v5` handles this automatically. Note `mise env` itself prints **plaintext** — `--redacted` only filters *which* vars are shown. The default `prefix`/`interleave` output styles print full logs with redactions applied; only `replacing`/`timed` hide lines.

### Error Handling

Tasks run with `set -e` semantics by default (the default inline shell is `sh -o errexit -c`; simple commands may run without a shell at all). Disable locally:
```toml
run = '''
set +e
cd /nonexistent
echo "This will not fail the task"
'''
```
