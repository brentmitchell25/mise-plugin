# mise Plugin for Claude Code

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Use Cases](#use-cases)
  - [Task Definition with Arguments](#1-task-definition-with-arguments)
  - [File-Based Tasks for Migrations](#2-file-based-tasks-for-migrations)
  - [Build Pipeline with Caching](#3-build-pipeline-with-caching)
  - [Dev Tool Setup](#4-dev-tool-setup)
  - [Environment Configuration](#5-environment-configuration)
  - [Hooks and Watchers](#6-hooks-and-watchers)
  - [New Developer Machine Setup](#7-new-developer-machine-setup)
  - [Sandboxing Untrusted Tasks](#8-sandboxing-untrusted-tasks)
  - [Task Output Caching](#9-task-output-caching)
  - [Monorepo Affected-Task Selection](#10-monorepo-affected-task-selection)
  - [Project Services as Task Prerequisites](#11-project-services-as-task-prerequisites)
  - [Project Requirements Checks](#12-project-requirements-that-tool-versions-cant-express)
- [Advanced Examples](#advanced-examples)
  - [Multi-Environment Terraform Deployment](#1-multi-environment-terraform-deployment-with-dynamic-completions)
  - [Dockerized Microservice Pipeline](#2-dockerized-microservice-build-pipeline-with-caching)
  - [Release Automation](#3-full-release-automation-with-changelog-generation)
  - [Database Management Suite](#4-database-management-suite-with-migration-tracking)
  - [Kubernetes Deployment Orchestration](#5-kubernetes-deployment-orchestration-with-namespace-management)
- [What's Covered](#whats-covered)
- [Requirements](#requirements)
- [How It Works](#how-it-works)
- [Updating the Skill](#updating-the-skill)
- [License](#license)

---

## Overview

A Claude Code plugin that teaches Claude how to work with [mise](https://mise.jdx.dev/) — the all-in-one developer environment tool. This covers the complete mise ecosystem: task definition, dev tool management, environment configuration, hooks, and the full usage spec for argument handling.

When installed, Claude will:
- Use the `usage` field for all task arguments (never shell-native `$1`/`$@` patterns)
- Generate correct TOML-based and file-based tasks with proper configuration
- Configure dev tools across 20 backends (packslip, aqua, github, npm, cargo, pypi, pkgx, etc.)
- Set up environment variables, configuration environments, dotenv loading, and secrets
- Create hooks and file watchers
- Cache task outputs and replay logs instead of rebuilding (local and remote)
- Declare long-lived project services in `[daemons]` and make tasks wait on them
- Encode project requirements as `[doctor.checks]` for `mise doctor project`
- Wire up monorepos — workspace project graphs and `mise run --affected` in CI
- Sandbox risky tasks and read untrusted configs safely (`MISE_SAFE=1`)
- Provision whole machines with `mise bootstrap`, locally or over SSH (system packages, users, files, services, firewall, Docker Compose, git repos, dotfiles, macOS defaults)
- Track dotfiles in Git with `mise dot` — checkpoints, rollback, undo, and sharing
- Follow mise best practices throughout — including avoiding the usage-spec attributes that are feature-gated out of mise and fail at runtime, and the fields that have quietly become no-ops

This plugin is a pure skill — no commands, hooks, or MCP servers. It activates automatically whenever you work with mise.

## Features

### Tasks
- **Strict `usage` field enforcement** — all arguments use mise's cross-platform usage spec
- **TOML and file-based tasks** — inline tasks and executable scripts with `#MISE`/`#USAGE` headers
- **Complete usage spec v6** — positional args, flags, choices, custom completions, variadic args, defaults, env binding, plus `group`/`flagset`/`output` nodes and relationship attributes (`required_if`, `conflicts`, `requires`, `exclusive`)
- **Task dependencies** — `depends`, `depends_post`, `wait_for` with parallel execution and `optional` globs
- **Freshness** — `sources`/`outputs` with `!` exclusions and brace globs, including `auto` mode
- **Output caching** — `cache = { enabled = true }` restores artifacts and replays logs; `outputs = []` result-only caching; local and remote stores
- **Task templates** — `[task_templates]` + `extends` with documented override/deep-merge rules, and `usage` flags that **merge** rather than replace
- **Daemon prerequisites** — `daemons = [...]` starts and waits for project services before the task body runs
- **Per-task output control** — `output` style field, `timeout`, structured `run`/`depends` with args/env

### Dev Tools
- **20 backends** — core, packslip, aqua, github, gitlab, forgejo, http, s3, pypi (formerly pipx), npm, go, cargo, gem, dotnet, conda, spm, pkgx, vfox, asdf, ubi (deprecated)
- **Packslip** — signed publisher manifests with SSH-style signer pinning, plus version-matched shell completions, man pages, and agent skills
- **Per-tool options** — version, OS restriction (including `unix`), postinstall commands, install env, `additional_asset_patterns`, table-form `rename_exe`
- **Lazy tools** — `lazy = true` generates a bootstrap shim and defers install until first invocation
- **Lockfiles** — `mise.lock` **revision 2** with npm/Python dependency graphs in sidecars, config-root-scoped `[tool_config] locked`, `mise lock --bump`/`--upgrade`, and the built-in 24h `minimum_release_age`
- **Shims and aliases** — shell integration, tool aliasing, version aliasing, `shims.exclude`, tool stubs

### Environments
- **Environment variables** — basic, required, redacted (and `redact = false` opt-outs), lazy evaluation
- **Special directives** — `_.path`, `_.file` (with `expand`), `_.source`, `_.python.venv`
- **Configuration environments** — `MISE_ENV`, `.miserc.toml`, platform auto-envs
- **Templates** — Tera v2 templating with functions, filters, tests, and a v1 → v2 migration table

### Services and Diagnostics
- **Project daemons** — `[daemons]` with PostgreSQL/Redis/CockroachDB/NATS/SpiceDB presets, custom `run`/`task` daemons, `[daemon_groups]`, cross-project references, and `port = "auto"` for running one stack across git worktrees
- **Project diagnostics** — `[doctor.checks]` probes with `hint`, `timeout`, `dir`, `shell`, and `os` selectors for `mise doctor project`
- **Command wrappers** — `[wrappers]` intercepts a command name before it reaches the active toolset

### Configuration
- **Hooks** — cd, enter, leave, preinstall, postinstall triggers, with exact per-hook env vars and cross-config precedence
- **File watchers** — watch patterns with automatic re-execution
- **Settings** — ~324 configuration options across 36 namespaces, with types, defaults, and env var overrides
- **Hierarchical config** — file precedence, merge behavior, configuration environments
- **Monorepos** — workspace project graphs (Cargo/uv/Go/Node), `[monorepo.task_defaults]`, `^task` upstream deps, `mise run --affected`
- **Deprecation calendar** — every dated removal and default flip in one table, including fields that are already silent no-ops

### Security
- **Sandboxing** — `[settings.sandbox]` plus per-task `deny_*`/`allow_*` fields and matching CLI flags
- **Safe mode** — `MISE_SAFE=1` turns mise into an inert config reader for untrusted repos and fork PRs
- **Trust** — auto-trust rules, safe-config auto-load, git-worktree trust inheritance
- **Supply chain** — cosign/SLSA/attestation/minisign verification, packslip signer pinning, and the built-in 24h `minimum_release_age`

### Machine Setup
- **`mise bootstrap`** — declarative end-to-end machine/developer setup across 17 ordered steps: Linux users/groups, vfox plugins, system packages (apt/dnf/pacman/apk/zypper/aur/brew/brew-cask/macos-app/mas/flatpak/nix/winget/scoop), privileged files, system services, firewall rules, Docker Compose projects, git repos, shell activation, macOS defaults, launchd/systemd services and timers, login shell, tools, and a `bootstrap` task
- **Plan/apply workflow** — `mise bootstrap plan` with `--json` and `--detailed-exitcode`, plus per-resource `apply`/`status` subcommands that converge only on real drift
- **Remote provisioning** — `mise bootstrap remote` applies the same project over SSH to a `[bootstrap.remote.hosts]` inventory or ad-hoc targets, detecting each host's OS/arch/libc and fetching a minisign-verified binary
- **Declarative dotfiles** — `mise dot` / `mise bootstrap dotfiles` with symlink/symlink-each/copy/template modes, inline `content`, glob wildcards, `exclude` patterns, block/line edits, per-OS/profile `variants`, `encrypt`, and `unapply`
- **Dotfiles history** — Git-backed checkpoints via `mise dot save`/`history`/`rollback`/`undo`, a background watcher service, and optional sharing through a setup repository
- **Agent skills** — `mise skills ls`/`sync` links the version-matched `SKILL.md` bundles that `packslip:` tools publish into your agent's skills directory
- **OCI images** — `mise oci build/push/run` with a built-in registry client and `[[oci.copy]]` layers, plus official `ghcr.io/jdx/mise` images

## Installation

Add the marketplace and install the plugin (run these inside Claude Code):

```
/plugin marketplace add brentmitchell25/mise-plugin
/plugin install mise@brentmitchell25
```

Or from the CLI:

```bash
claude plugin install mise@brentmitchell25
```

## Use Cases

### 1. Task definition with arguments

Ask Claude: *"Create a mise task that deploys to dev/staging/prod with a force flag"*

```toml
[tasks.deploy]
description = "Deploy application to environment"
depends = ["build", "test"]
usage = '''
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
flag "-f --force" help="Skip confirmation"
'''
confirm = "Deploy to {{usage.env}}?"
run = './scripts/deploy.sh ${usage_env?} ${usage_force:+--force}'
```

### 2. File-based tasks for migrations

Ask Claude: *"Set up file-based mise tasks for database migrations"*

```bash
#!/usr/bin/env bash
#MISE description="Run database migrations"
#MISE depends=["db:check"]
#USAGE arg "<direction>" choices "up" "down" help="Migration direction"
#USAGE flag "-n --count <n>" default="1" help="Number of migrations"
#USAGE flag "--dry-run" help="Preview SQL only"

set -euo pipefail
direction="${usage_direction?}"
count="${usage_count:-1}"

if [ -n "${usage_dry_run:-}" ]; then
  echo "Would run $count migration(s) $direction"
  exit 0
fi

migrate "$direction" -n "$count"
```

### 3. Build pipeline with caching

Ask Claude: *"Configure a build pipeline with lint, test, build using caching"*

```toml
[tasks.lint]
description = "Run linter"
sources = ["src/**/*.ts", ".eslintrc.*"]
run = "eslint src/"

[tasks.test]
description = "Run tests"
depends = ["lint"]
sources = ["src/**/*.ts", "tests/**/*.ts"]
run = "vitest run"

[tasks.build]
description = "Build for production"
depends = ["test"]
sources = ["src/**/*.ts", "tsconfig.json"]
outputs = ["dist/**/*"]
run = "tsc --build"
```

### 4. Dev tool setup

Ask Claude: *"Set up mise.toml with node, python, and some CLI tools"*

```toml
[tools]
node = "22"
python = { version = "3.12", postinstall = "pip install -r requirements.txt" }
"aqua:BurntSushi/ripgrep" = "latest"
"npm:prettier" = "latest"
"pypi:psf/black" = "latest"
"packslip:github.com/jdx/hk" = "latest"   # signed manifest + version-matched completions
terraform = { version = "1.9", lazy = true }   # installs on first `terraform` invocation
```

### 5. Environment configuration

Ask Claude: *"Configure mise environments with profiles for dev and staging"*

```toml
# mise.toml (base)
[env]
APP_NAME = "myapp"
DATABASE_URL = { required = "Set postgres connection string" }
API_KEY = { value = "{{env.API_KEY}}", redact = true }
_.path = ["./node_modules/.bin", "{{config_root}}/scripts"]
_.file = [".env", ".env.local"]
```

```toml
# mise.staging.toml (loaded when MISE_ENV=staging)
[env]
DATABASE_URL = "postgres://staging-host:5432/myapp"
LOG_LEVEL = "info"
```

### 6. Hooks and watchers

Ask Claude: *"Set up mise hooks for project enter and file watchers for auto-formatting"*

```toml
[hooks]
enter = "echo 'Welcome to the project — running setup...'"
postinstall = "echo 'Tools installed successfully'"

[[watch_files]]
patterns = ["src/**/*.rs"]
run = "cargo fmt"
```

### 7. New developer machine setup

Ask Claude: *"Set up mise bootstrap so a new developer can provision everything with one command"*

```toml
[bootstrap.packages]
"apt:build-essential" = "latest"
"brew-cask:visual-studio-code" = "latest"

[bootstrap.repos]
"~/src/dotfiles" = { url = "git@github.com:myorg/dotfiles.git", ref = "main" }

[bootstrap.mise_shell_activate]
zprofile = "shims"
zshrc = "activate"

[dotfiles]
"~/.gitconfig" = "dotfiles/gitconfig"
"~/.zshrc/activate" = { block = 'eval "$(mise activate zsh)"' }

[tools]
node = "lts"
python = "3.12"

[tasks.bootstrap]
run = "gh auth status || gh auth login"
```

Then run `mise bootstrap` (add `--dry-run` to preview, `--yes` to skip prompts, `--only`/`--skip` to run a subset). Stable since mise 2026.7.4 — no longer needs `MISE_EXPERIMENTAL`.

### 8. Sandboxing untrusted tasks

Ask Claude: *"Lock down this dependency-install task so it can only reach the npm registry"*

```toml
[tasks.fetch-deps]
description = "Install dependencies with no ambient access"
deny_all = true
allow_net = ["registry.npmjs.org"]
allow_read = ["{{config_root}}"]
allow_write = ["{{config_root}}/node_modules"]
run = "npm ci"
```

Or set a deny-by-default policy for the whole project and open access per task:

```toml
[settings.sandbox]
deny_net = true
deny_write = true
```

For configs you don't control (fork PRs, third-party repos), `MISE_SAFE=1` makes mise refuse to run hooks, tasks, `_.source`, and template `exec()` entirely — so it can read tool versions without executing anything.

### 9. Task output caching

Ask Claude: *"Make my build and lint tasks reuse cached results instead of re-running when nothing changed"*

```toml
[settings]
experimental = true

[task_config]
global_inputs = ["mise.toml", "@group:lockfiles"]

[task_config.input_groups]
lockfiles = ["package-lock.json", "pnpm-lock.yaml"]

[tasks.build]
run = "tsc --build"
sources = ["src/**/*.ts", "!src/**/*.test.ts", "tsconfig.json"]
outputs = ["dist/**"]
cache = { enabled = true, command_inputs = ["tsc --version"] }

[tasks.lint]
run = "eslint src/"
sources = ["src/**/*.ts", ".eslintrc.*"]
outputs = []                      # result-only: caches success + replayable logs, no archive
cache = { enabled = true, env = ["CI"] }
pass_through_env = ["GITHUB_TOKEN"]   # available to the command, not part of the cache key
```

Unlike `sources`/`outputs` freshness — which only *skips* a task — the cache restores declared outputs and replays stdout/stderr from a content-addressed archive. Inspect and control it:

```bash
mise run --task-cache-explain build   # what went into the key (never prints secret-derived hashes)
mise run --task-cache off build       # read-write | read-only | write-only | off | local-only
mise cache task build --json          # stored size, restorable bytes, time saved
mise cache clear --task build         # leaves working-tree outputs alone
```

### 10. Monorepo affected-task selection

Ask Claude: *"Only run tests for packages my branch actually touched"*

```toml
# Root mise.toml
monorepo_root = true

[settings]
experimental = true
task.auto_infer = ["node"]        # package.json scripts become tasks, no per-package mise.toml

[monorepo]
config_roots = ["packages/*", "services/*"]

[monorepo.task_defaults.build]
sources = ["src/**", "package.json"]
outputs = ["dist/**"]
cache = { enabled = true }
depends = ["^build"]              # build every upstream package first, transitively

[monorepo.task_defaults.test]
env = { NODE_ENV = "test" }
```

```bash
mise run --affected test                              # only projects touched by the diff
mise run --affected --affected-base origin/main test  # plus their reverse dependencies
mise run --affected --affected-explain --dry-run test # why each task was selected
mise tasks graph --explain                            # inferred projects, edges, and provenance
```

mise infers the project graph from Cargo, uv, Go, and Node workspaces **without the toolchain installed**, and auto-detects base/head revisions on GitHub Actions and GitLab.

### 11. Project services as task prerequisites

Ask Claude: *"My tests need Postgres and Redis running. Set that up so `mise run test` just works."*

```toml
[settings]
experimental = true

[daemons]
postgres = "18"
redis = "8"

[tasks.test]
daemons = ["postgres", "redis"]   # started and waited for before the task body
run = "npm test"
```

`mise run test` installs missing tools, starts both services, waits for pitchfork to report them ready, then runs the tests. Already-running daemons are reused, and they stay up between invocations. This replaces the usual "prerequisite task that launches a background process and polls for readiness" pattern.

The presets export connection variables (`DATABASE_URL`, `PGPORT`, `REDIS_URL`, …) and keep data between runs. For several git worktrees sharing one stack, `port = "auto"` derives a distinct port per checkout so they don't collide:

```toml
[daemons.postgres]
preset = "postgres"
version = "18"
port = "auto"        # primary checkout keeps 5432; linked worktrees get 5433+
```

```bash
mise daemons start          # or `mise daemons start <group>`
mise daemons ls --json      # resolved ports, including automatic allocation
mise daemons logs postgres
mise daemons stop
```

### 12. Project requirements that tool versions can't express

Ask Claude: *"Add a check that fails with a useful hint if OpenSSL headers or the database aren't available."*

```toml
[doctor.checks.openssl]
description = "OpenSSL development files are discoverable"
run = "pkg-config --exists openssl"
hint = "Run `mise bootstrap packages apply` to install the declared build dependencies."
timeout = "5s"
os = ["linux", "macos"]

[doctor.checks.database]
description = "PostgreSQL accepts connections"
run = "pg_isready --quiet"
hint = "Start the project's services with `mise daemons start postgres`."
```

```bash
mise doctor project          # PASS/FAIL per check, with the hint on failure
mise doctor project --json
```

Checks run concurrently, a failure never stops the others, and command output is captured and **discarded** — so a probe can't accidentally print a credential into the report. Explain the requirement with `description` and the remedy with `hint`. Ordinary `mise doctor` still diagnoses mise itself and never runs these.

## Advanced Examples

These examples demonstrate what Claude generates with the plugin installed — showcasing dynamic completions, safety confirmations, dependency chains, caching, file-based tasks, and real-world tooling integration.

### 1. Multi-environment Terraform deployment with dynamic completions

*"Set up mise tasks for deploying infrastructure with Terraform. I have services in `infrastructure/<service>/application/` and environment tfvars in `infrastructure/<service>/application/env/`. I need plan, apply, and destroy tasks with dynamic service/environment tab completion."*

```toml
[tasks."infra:plan"]
description = "Terraform plan for a service environment"
usage = '''
arg "<service>" help="Service to deploy"
complete "service" run="ls -d infrastructure/*/application 2>/dev/null | sed 's|infrastructure/||;s|/application||'"
arg "<environment>" help="Target environment"
complete "environment" run="ls infrastructure/${usage_service}/application/env/ 2>/dev/null | sed 's/.tfvars//'"
flag "--destroy" help="Plan a destroy operation"
flag "-l --lock" default="true" negate="--no-lock" help="Lock state during plan"
'''
env = { TF_IN_AUTOMATION = "1" }
run = '''
#!/usr/bin/env bash
set -euo pipefail
svc_dir="infrastructure/${usage_service?}/application"
cd "$svc_dir"
terraform init -backend-config="env/${usage_environment?}.backend.hcl" -reconfigure
terraform plan \
  -var-file="env/${usage_environment?}.tfvars" \
  ${usage_destroy:+-destroy} \
  ${usage_lock:+-lock=true} \
  -out="${usage_environment}.tfplan"
'''

[tasks."infra:apply"]
description = "Apply a Terraform plan"
usage = '''
arg "<service>" help="Service to deploy"
complete "service" run="ls -d infrastructure/*/application 2>/dev/null | sed 's|infrastructure/||;s|/application||'"
arg "<environment>" help="Target environment"
complete "environment" run="ls infrastructure/${usage_service}/application/env/ 2>/dev/null | sed 's/.tfvars//'"
'''
confirm = "Apply Terraform plan for {{usage.service}} in {{usage.environment}}?"
run = '''
#!/usr/bin/env bash
set -euo pipefail
cd "infrastructure/${usage_service?}/application"
terraform apply "${usage_environment?}.tfplan"
'''

[tasks."infra:destroy"]
description = "Destroy infrastructure for a service environment"
depends = ["infra:plan {{usage.service}} {{usage.environment}} --destroy"]
usage = '''
arg "<service>" help="Service to destroy"
complete "service" run="ls -d infrastructure/*/application 2>/dev/null | sed 's|infrastructure/||;s|/application||'"
arg "<environment>" help="Target environment"
complete "environment" run="ls infrastructure/${usage_service}/application/env/ 2>/dev/null | sed 's/.tfvars//'"
'''
confirm = "DESTROY all infrastructure for {{usage.service}} in {{usage.environment}}? This cannot be undone."
run = '''
#!/usr/bin/env bash
set -euo pipefail
cd "infrastructure/${usage_service?}/application"
terraform apply "${usage_environment?}.tfplan"
'''
```

### 2. Dockerized microservice build pipeline with caching

*"I have a monorepo with services in `services/`. Each has a Dockerfile. Set up build, test, and publish tasks with image tagging, caching, and a full CI pipeline task that runs everything."*

```toml
[vars]
registry = "ghcr.io/myorg"
git_sha = "{{exec(command='git rev-parse --short HEAD')}}"

[tasks."docker:build"]
description = "Build a service Docker image"
usage = '''
arg "<service>" help="Service to build"
complete "service" run="ls services/"
flag "--no-cache" help="Build without Docker cache"
flag "-t --tag <tag>" default="latest" help="Image tag"
'''
sources = ["services/{{usage.service}}/**/*"]
run = '''
#!/usr/bin/env bash
set -euo pipefail
image="{{vars.registry}}/${usage_service?}:${usage_tag:-latest}"
docker build \
  ${usage_no_cache:+--no-cache} \
  --label "git.sha={{vars.git_sha}}" \
  -t "$image" \
  -f "services/${usage_service?}/Dockerfile" \
  "services/${usage_service?}"
echo "Built $image"
'''

[tasks."docker:test"]
description = "Run tests for a service inside its container"
usage = '''
arg "<service>" help="Service to test"
complete "service" run="ls services/"
flag "-v --verbose" help="Verbose test output"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
image="{{vars.registry}}/${usage_service?}:latest"
docker run --rm \
  -e CI=true \
  ${usage_verbose:+-e VERBOSE=1} \
  "$image" npm test
'''

[tasks."docker:publish"]
description = "Push service image to registry"
usage = '''
arg "<service>" help="Service to publish"
complete "service" run="ls services/"
flag "-t --tag <tag>" default="latest" help="Image tag"
'''
confirm = "Push {{vars.registry}}/{{usage.service}}:{{usage.tag}} to registry?"
run = '''
#!/usr/bin/env bash
set -euo pipefail
image="{{vars.registry}}/${usage_service?}:${usage_tag:-latest}"
docker push "$image"
docker tag "$image" "{{vars.registry}}/${usage_service?}:{{vars.git_sha}}"
docker push "{{vars.registry}}/${usage_service?}:{{vars.git_sha}}"
'''

[tasks."ci:pipeline"]
description = "Full CI pipeline: lint, build all, test all, publish"
depends = ["lint"]
usage = '''
flag "--publish" help="Also publish images after tests pass"
flag "--services <list>" var=#true default="api gateway worker" help="Services to build"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
for svc in $usage_services; do
  echo "=== Building $svc ==="
  mise run docker:build "$svc"
  mise run docker:test "$svc"
  if [ -n "${usage_publish:-}" ]; then
    mise run docker:publish "$svc"
  fi
done
echo "Pipeline complete."
'''
```

### 3. Full release automation with changelog generation

*"Create a release task that bumps semver, generates a changelog from conventional commits, tags the release, and optionally publishes to npm."*

```bash
#!/usr/bin/env bash
#MISE description="Create a new release with changelog generation"
#MISE alias="rel"
#MISE depends=["test", "build"]
#MISE tools={node="22", git="latest"}
#USAGE arg "<bump>" choices "major" "minor" "patch" help="Semver bump type"
#USAGE flag "--dry-run" help="Preview release without making changes"
#USAGE flag "--publish" help="Publish to npm after release"
#USAGE flag "--pre <label>" help="Pre-release label (e.g., beta, rc)"

set -euo pipefail

bump="${usage_bump?}"
dry="${usage_dry_run:-}"
pre="${usage_pre:-}"

if [ -n "$(git status --porcelain)" ]; then
  echo "Error: Working tree is not clean."
  exit 1
fi

current=$(jq -r .version package.json)
IFS='.' read -r major minor patch <<< "${current%%-*}"
case "$bump" in
  major) major=$((major + 1)); minor=0; patch=0 ;;
  minor) minor=$((minor + 1)); patch=0 ;;
  patch) patch=$((patch + 1)) ;;
esac
new_version="${major}.${minor}.${patch}"
[ -n "$pre" ] && new_version="${new_version}-${pre}.0"

echo "Release: ${current} → ${new_version}"

last_tag=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
if [ -n "$last_tag" ]; then
  changelog=$(git log "${last_tag}..HEAD" --pretty=format:"- %s (%h)" --no-merges)
else
  changelog=$(git log --pretty=format:"- %s (%h)" --no-merges)
fi

if [ -n "$dry" ]; then
  echo "[dry-run] Would bump to ${new_version}, tag, and push."
  exit 0
fi

jq --arg v "$new_version" '.version = $v' package.json > package.json.tmp
mv package.json.tmp package.json

git add package.json
git commit -m "chore(release): ${new_version}"
git tag -a "v${new_version}" -m "Release ${new_version}"
git push && git push --tags

if [ -n "${usage_publish:-}" ]; then
  npm publish ${pre:+--tag "$pre"}
fi

echo "Released v${new_version}"
```

### 4. Database management suite with migration tracking

*"Set up a complete database management task suite with PostgreSQL."*

```toml
[env]
DATABASE_URL = { required = "Set PostgreSQL connection string" }

[tasks."db:migrate"]
description = "Run database migrations"
usage = '''
arg "[direction]" default="up" choices "up" "down" help="Migration direction"
flag "-n --steps <n>" default="0" help="Number of steps (0 = all)"
flag "--dry-run" help="Show SQL without executing"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
dir="${usage_direction:-up}"
steps="${usage_steps:-0}"
if [ -n "${usage_dry_run:-}" ]; then
  migrate -database "$DATABASE_URL" -path db/migrations "$dir" -v $steps 2>&1 | head -50
  echo "[dry-run] No changes applied."
  exit 0
fi
if [ "$steps" = "0" ]; then
  migrate -database "$DATABASE_URL" -path db/migrations "$dir"
else
  migrate -database "$DATABASE_URL" -path db/migrations "$dir" "$steps"
fi
'''

[tasks."db:reset"]
description = "Drop all tables and re-run migrations + seeds"
depends_post = ["db:migrate", "db:seed"]
confirm = "This will DROP ALL TABLES and re-run migrations. Continue?"
run = 'psql "$DATABASE_URL" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"'

[tasks."db:dump"]
description = "Dump database to SQL file"
usage = 'flag "-o --output <file>" default="db/dump.sql" help="Output file"'
run = 'pg_dump "$DATABASE_URL" > "${usage_output:-db/dump.sql}" && echo "Dumped to ${usage_output:-db/dump.sql}"'
```

### 5. Kubernetes deployment orchestration with namespace management

*"Create mise tasks for K8s deployments with context, deploy, rollback, and port-forward."*

```toml
[tasks."k8s:ctx"]
description = "Switch Kubernetes context and namespace"
usage = '''
arg "<cluster>" help="Cluster context"
complete "cluster" run="kubectl config get-contexts -o name"
flag "-n --namespace <ns>" default="default" help="Namespace"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
kubectl config use-context "${usage_cluster?}"
kubectl config set-context --current --namespace="${usage_namespace:-default}"
echo "Context: ${usage_cluster?} | Namespace: ${usage_namespace:-default}"
'''

[tasks."k8s:deploy"]
description = "Deploy a service to Kubernetes"
usage = '''
arg "<service>" help="Service to deploy"
complete "service" run="ls -d k8s/services/*/ 2>/dev/null | xargs -I{} basename {}"
arg "[tag]" default="latest" help="Image tag"
flag "-n --namespace <ns>" help="Override namespace"
flag "--dry-run" help="Render without applying"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
svc="${usage_service?}"
tag="${usage_tag:-latest}"
ns="${usage_namespace:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
kustomize build "k8s/services/${svc}" | \
  sed "s|:latest|:${tag}|g" | \
  kubectl apply ${usage_dry_run:+--dry-run=client} -n "${ns:-default}" -f -
'''

[tasks."k8s:rollback"]
description = "Rollback a deployment"
usage = '''
arg "<service>" help="Service to rollback"
complete "service" run="kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}'"
flag "-n --namespace <ns>" help="Override namespace"
'''
confirm = "Rollback {{usage.service}} to previous revision?"
run = '''
#!/usr/bin/env bash
set -euo pipefail
ns="${usage_namespace:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
kubectl rollout undo deployment/"${usage_service?}" -n "${ns:-default}"
'''

[tasks."k8s:forward"]
description = "Port-forward a service"
usage = '''
arg "<service>" help="Service to forward"
complete "service" run="kubectl get svc -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}'"
flag "-p --port <port>" default="8080" help="Local port"
flag "--remote <port>" default="80" help="Remote port"
'''
raw = true
run = '''
#!/usr/bin/env bash
set -euo pipefail
echo "Forwarding localhost:${usage_port:-8080} → ${usage_service?}:${usage_remote:-80}"
kubectl port-forward "svc/${usage_service?}" "${usage_port:-8080}:${usage_remote:-80}"
'''
```

## What's Covered

The skill provides Claude with comprehensive knowledge of:

| Area | Details |
|------|---------|
| **Tasks** | All fields (run, depends, sources, outputs, cache, usage, extends, timeout, output, pass_through_env, sandbox, etc.), structured run/depends, `[task_templates]`, file tasks, remote tasks |
| **Usage Spec** | Full arg/flag/cmd/complete reference with all attributes, `effect=`, env var access patterns, bash expansion — plus the attributes that are documented upstream but hard-error |
| **Task Caching** | `cache = {}`, result-only `outputs = []`, `input_groups`/`@group:` refs, `global_inputs`, `command_inputs`, `--task-cache` modes, `mise cache task`, remote cache settings |
| **Dev Tools** | 20 backends, per-tool options, `lazy`/`lazy_bins`, `version_order`, version formats, backend-specific config (packslip, github, http, s3, cargo, pypi, npm, pkgx, etc.), tool stubs, lockfile revision 2, provenance |
| **Daemons** | `[daemons]`, `[daemon_groups]`, `[daemons_settings]`, five service presets, custom `run`/`task` daemons, cross-project references, worktree-aware `port = "auto"`, task `daemons` prerequisites |
| **Diagnostics** | `[doctor.checks]` fields, concurrency and output limits, `mise doctor project --json` |
| **Environments** | env vars, `_.path`/`_.file`/`_.source`/`_.python.venv` directives, configuration environments, `.miserc.toml`, required/redacted vars, secrets (fnox/sops/age), Tera v2 templates |
| **Hooks** | cd/enter/leave/preinstall/postinstall hooks, exact per-hook env vars, cross-config precedence, file watchers |
| **Wrappers** | `[wrappers]` command interception and `activate_shims` |
| **Security** | `[settings.sandbox]`, per-task deny/allow fields, `MISE_SAFE=1`, paranoid mode, trust rules, supply-chain verification |
| **Machine Bootstrap** | `mise bootstrap` (17 steps: accounts, plugins, packages, files, services, firewall, compose, repos, shell activation, macOS defaults, services/timers, login shell), 14 package managers, `mise dot` dotfiles + Git-backed history, agent skills, OCI images |
| **Dependencies** | `[deps]` providers (npm, uv, poetry, go, bundler, …) and `mise deps` |
| **Monorepos** | `monorepo_root`, `[monorepo].config_roots`, `//path:task` syntax, workspace project graph (Cargo/uv/Go/Node), `[monorepo.task_defaults]`, `^task` upstream deps, `mise run --affected`, root lockfiles |
| **Configuration** | File hierarchy, ~324 settings across 36 namespaces, `conf.d` fragments, merge behavior, minimum version, deprecation calendar |
| **Best Practices** | DO/DON'T patterns, complete examples, common gotchas |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [mise](https://mise.jdx.dev/) installed in your development environment

## How It Works

This plugin provides a single skill file (`SKILL.md`) that Claude reads when it detects mise-related context in your conversation. The skill contains comprehensive reference material covering the entire mise ecosystem — tasks, dev tools, environments, hooks, and configuration.

The skill activates automatically when you mention mise, edit `mise.toml` files, or work with `mise-tasks/` directories.

## Updating the Skill

This repo includes a Claude Code slash command for updating the skill as mise docs evolve:

```
/update-skill    # Re-crawl mise docs and update SKILL.md
```

The command launches 6 parallel research agents — 5 documentation crawlers plus a release-notes auditor that diffs `jdx/mise` and `jdx/usage` releases since the last update — then consolidates findings into `skills/mise/SKILL.md`. Release notes take precedence over the doc pages for experimental/stable status, renamed keys, and changed defaults, since the docs lag behind releases. Additional mise tasks handle versioning and validation:

```bash
mise run lint              # Validate skill file structure
mise run update:version    # Bump plugin version
```

## License

[MIT](LICENSE)
