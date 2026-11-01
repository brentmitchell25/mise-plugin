## Monorepo Tasks and Workspace Graph

**Contents**

- [Workspace Project Graph](#workspace-project-graph)
- [Affected Tasks](#affected-tasks)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

```toml
# Root mise.toml
monorepo_root = true

[monorepo]
config_roots = ["packages/frontend", "packages/backend", "services/*"]
lockfile = true    # true = single root lockfile; false = per-subproject; unset = current behavior
```

Stable since v2026.6.6 — no longer requires `MISE_EXPERIMENTAL=1`. Provides implicit trust for descendants, lazy task loading, and tool/env/vars inheritance from parent configs.

> `experimental_monorepo_root` is **deprecated and emits a warning**. Removal: **2027.12.0**. Use `monorepo_root`.

**Path syntax:**
```bash
mise //projects/frontend:build    # Absolute path from root
mise :build                       # Task in current config_root (leading : recommended)
mise '//projects/frontend:*'      # All tasks in frontend
mise //...:test                   # Test task in all projects
mise //projects/.../api:build     # ... matches any directory depth
mise '//...:test*'                # Wildcard task names across all projects
```

`...` matches directory depth (bazel/buck2 style); `*` matches task names. Path globs (`*`/`**` in the path portion) are **not** supported yet. mise never defines commands starting with `//` or `:`, so direct invocation is safe here. `mise run :<TAB>` completes the `:task` shorthand (2026.10.1).

**Short names for deep project paths (`[monorepo.path_aliases]`, 2026.9.16):**
```toml
monorepo_root = true

[monorepo]
config_roots = ["foo/bar/baz/abc/123"]

[monorepo.path_aliases]
"123" = "foo/bar/baz/abc/123"
```
`mise run //123:build` then runs `//foo/bar/baz/abc/123:build` — also in `depends`, patterns (`//123:*`), and child paths (`//123/sub:build`). An alias must be a **single path segment**, must point at a root in `config_roots`, can't contain `...`, and can't overlap an existing root. The full path stays the canonical name in listings and output prefixes.

**Listing & install:**
```bash
mise tasks                          # current config_root hierarchy
mise tasks --all                    # whole monorepo, including siblings
mise tasks '//projects/frontend:*'
MISE_ENV=ci mise install --monorepo
mise install --monorepo node
mise ls --monorepo
```

`[monorepo].config_roots` supports single-level `*` globs only (`**` unsupported) and skips filesystem walking; a glob skips dirs with no mise config, `.mise/tasks/`, or `mise-tasks/`, and absolute paths or paths with `..` are ignored with a warning. **Automatic filesystem discovery is deprecated** — declare `config_roots` explicitly (`mise install --monorepo` and `mise ls --monorepo` **fail** without it). With `lockfile = true` the root holds `mise.lock`, `mise.ci.lock`, and `mise.local.lock`. Each declared root loads its full config hierarchy, so a child with no tool overrides keeps the root's tools (2026.10.21); `turbo.json` is parsed as JSONC, so comments are fine (2026.10.24).

Inside a config root, a bare `build` equals `:build` (CLI and `depends`); `:task` resolves against the monorepo root when no config root encloses the cwd. More wildcards: `'//projects/frontend:**'`, `'//...:**'`, `//.../frontend:build`.

Discovery tuning when relying on walking:
```toml
[settings.task]
monorepo_depth = 5
monorepo_exclude_dirs = ["dist", "node_modules"]
monorepo_respect_gitignore = true
```

**Nested roots:** the **nearest** `monorepo_root = true` wins (e.g. git worktrees inside the main checkout). Tasks from the enclosing monorepo are not loaded, but the enclosing config remains an ancestor for tools, env vars, and vars.

> `[monorepo].lockfile` rollout: unset keeps per-subproject lockfiles today; **warns in 2026.12.0, defaults to root lockfiles in 2027.6.0**. Pin `lockfile = false` for mixed-version teams. Old subproject lockfiles auto-migrate into the root lockfile (root entries win).

### Workspace Project Graph

Requires only `monorepo_root = true` — **the experimental gate was removed in 2026.10.6** for the graph, `--affected`, `[monorepo.task_defaults]`, `[monorepo.projects]`, and `task.auto_infer` (docs: `https://mise.jdx.dev/tasks/workspace-graph.html`). mise discovers projects and their internal dependency edges **without the underlying toolchain installed** — a project does not need its own `mise.toml` to appear in the graph.

| Ecosystem | Project ID | Discovery |
|-----------|-----------|-----------|
| **Cargo** | `cargo:<package>` | `[workspace]` members/exclude + root `[package]`; edges from local `path` deps across normal/dev/build/target tables, renamed deps, and `workspace = true` inheritance. No `cargo` binary. |
| **uv (Python)** | `uv:<package>` | `[tool.uv.workspace]` member globs/exclusions; edges only when `[tool.uv.sources]` selects a workspace member (`workspace = true`) or an in-repo `path`. Covers main, optional, `[dependency-groups]`, and legacy dev deps. No `uv`/Python invoked. |
| **Go** | `go:<module-path>` | `use` directives in `go.work` plus each `go.mod`. **Does not infer edges** from `require`/`replace` — declare them via overrides. |
| **Gradle** (2026.10.12) | `gradle:<project path>` (`:app` → `gradle::app`, root → `gradle::`) | Literal `include` directives in the root `settings.gradle`/`settings.gradle.kts` — **never runs Gradle**. Edges from `project(":x")` / `project(path: ":x")` / `project(path = ":x")` in a top-level `dependencies {}` block. Doesn't see `subprojects{}`/`allprojects{}`, convention plugins, `includeBuild`, or version catalogs. A non-literal/conditional `include`, a non-`file("…")` `projectDir`, or **both** settings files existing is an **error** — fix by declaring the projects (below). |
| **Node** | `node:<pkg>` | npm/pnpm/Yarn/Bun via `pnpm-workspace.yaml` or the `workspaces` field (pnpm wins when both exist; Yarn Classic `workspaces.packages` and a single-string `workspaces` also read); for pnpm/Yarn a root `package.json` with a `name` is a project. Edges from `dependencies`, `devDependencies`, `optionalDependencies`, `peerDependencies` (version strings are opaque — `workspace:*`, `catalog:`, ranges all yield the same edge). Discovery skips `.git` and `node_modules` but doesn't read `.gitignore`. |

uv names are normalized (`my_package`/`my.package`/`my-package` → `uv:my-package`); local path sources are projects even when excluded from the workspace.

```bash
mise tasks graph              # inspect the graph
mise tasks graph --explain    # provider + metadata-source provenance per project/edge/task
mise tasks graph --json
```

Cycles are **reported**, not silently dropped. Config overrides are labeled `configuration` rather than misattributed to inference.

**Explicit overrides (`[monorepo.projects."<id>"]`):**
```toml
[monorepo.projects."go:example.com/acme/api"]
depends_add = ["go:example.com/acme/lib"]   # depends replaces; depends_add/depends_remove adjust

[monorepo.projects."gradle::app"]
root = "app"                                # replace the inferred root (or add a project)

[monorepo.projects."custom:docs"]           # a NEW project needs a namespaced ID + root
root = "docs"
metadata = { owner = "docs-team" }

[monorepo.projects."node:legacy-ui"]
remove = true                               # drop an inferred project (cannot combine with other fields)
```

| Field | Type | Notes |
|-------|------|-------|
| `root` | string | Replaces the inferred root; with a new ID, adds a project |
| `depends` | string[] | Replaces inferred edges |
| `depends_add` / `depends_remove` | string[] | Adjust inferred edges |
| `metadata` | table | Free-form |
| `remove` | bool (`false`) | Remove the project; exclusive with every other field |

**Provider failure takeover (2026.10.12):** when a provider can't read its metadata, `mise tasks graph` and `--affected` **fail**. Declaring any project in that provider's namespace makes mise ignore the error and use your declarations as that provider's whole contribution.

**Node package scripts as tasks** (opt-in) — `package.json` scripts become first-class tasks with no per-package `mise.toml`:
```toml
monorepo_root = true

[settings]
task.auto_infer = ["node"]   # no longer needs experimental (2026.10.6)
```
Task ID `node:@acme/web#build`, with the monorepo path `//apps/web:build` as an alias. Runs in the package directory through the detected workspace package manager (`pnpm-workspace.yaml` → `packageManager` → lockfile bun/pnpm/yarn/npm → npm) as `<pm> run <script> --`, with raw arg passthrough. An explicit mise task at that path **replaces** the imported one, and both names resolve to it. A `<package>#<task>` entry in `turbo.json` beats a plain `<task>` entry. The Node provider also imports `inputs`, `outputs`, `cache`, and `dependsOn` from matching **`turbo.json`** entries (tracking `turbo.json` as a task-definition source); Turbo expressions mise can't preserve (e.g. `$TURBO_ROOT$`) are left unset. mise never guesses outputs or cacheability from a command string.

**Root task defaults** (applied by task name to inferred and explicit workspace tasks):
```toml
[monorepo.task_defaults.build]
sources = ["src/**", "package.json"]
outputs = ["dist/**"]
cache = { enabled = true }
depends = ["^build"]

[monorepo.task_defaults.test]
env = { NODE_ENV = "test" }
```

**Precedence** (high → low): the task's own fields → the `extends` template → `[monorepo.task_defaults.<name>]`. Map fields (`env`, `vars`, `tools`) merge across layers; collection fields (`depends`, `sources`, `outputs`) take the complete value from the highest-precedence layer that defines them — **not** concatenated.

**Upstream dependencies (`^`)** — runs the same task in every upstream project first, expanding transitively and skipping projects that lack it. Supported in **`depends` only**, not `depends_post` or `wait_for`. Does nothing for a config root that isn't in the graph.
```toml
depends = ["^build"]
```

**Relative (`./`) dependencies** resolve from the declaring task's own monorepo location, so one aggregate declaration works at root, in nested apps, and in leaves. A trailing `...` includes the base project itself as well as its descendants:
```toml
[tasks.test]
depends = [{ task = "./...:groups:tests:*", optional = true }]
```

### Affected Tasks

`mise run --affected <task>` selects only the projects owning changed paths (the **deepest** owning project root), then adds all reverse dependents. A changed file outside every project, or matching `task_config.global_inputs`, selects the whole workspace. Only `pnpm-lock.yaml` is attributed per project; other lockfiles count as ordinary files. A bare task name means `//...:<task>`. **Not experimental since 2026.10.6.**

```bash
mise run --affected test
mise run --affected --affected-base origin/main --affected-head HEAD test
mise run --affected --affected-committed test     # only committed base...head changes
mise run --affected --affected-explain --dry-run build
mise run --affected --affected-json build
```

**Working-tree changes count (2026.10.6).** By default the change set is committed `base...head` **plus** staged, unstaged, and untracked (unignored) files — but only when head is the current checkout. Naming any of `--affected-committed`, `--affected-uncommitted`, `--affected-untracked` limits selection to those sources (they combine); the latter two error when head isn't the checkout.

Defaults are `HEAD~1` / `HEAD`; `MISE_AFFECTED_BASE` / `MISE_AFFECTED_HEAD` override them; explicit CLI options win. **CI defaults:** GitHub Actions — base `origin/$GITHUB_BASE_REF`, head `$GITHUB_SHA`; GitLab MR — base `$CI_MERGE_REQUEST_DIFF_BASE_SHA` (or `origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME`), head `$CI_COMMIT_SHA`. Clone with enough history (`fetch-depth: 0`). `--affected-explain` reasons: `changed path`, `workspace-global path`, `lockfile change`, `depends on affected project`.
