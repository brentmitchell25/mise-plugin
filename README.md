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
- Configure dev tools across 18 backends (aqua, github, npm, cargo, pipx, etc.)
- Set up environment variables, profiles, dotenv loading, and secrets
- Create hooks and file watchers
- Follow mise best practices throughout

This plugin is a pure skill — no commands, hooks, or MCP servers. It activates automatically whenever you work with mise.

## Features

### Tasks
- **Strict `usage` field enforcement** — all arguments use mise's cross-platform usage spec
- **TOML and file-based tasks** — inline tasks and executable scripts with `#MISE`/`#USAGE` headers
- **Complete usage spec** — positional args, flags, choices, custom completions, variadic args, defaults, env binding
- **Task dependencies** — `depends`, `depends_post`, `wait_for` with parallel execution
- **Caching** — `sources`/`outputs` for rebuild avoidance, including `auto` mode
- **New features** — `extends` (task inheritance), `timeout`, structured `run`/`depends` with args/env

### Dev Tools
- **18 backends** — aqua, github, gitlab, forgejo, http, s3, pipx, npm, go, cargo, gem, dotnet, conda, spm, vfox, asdf
- **Per-tool options** — version, OS restriction, postinstall commands
- **Shims and aliases** — shell integration, tool aliasing, version aliasing

### Environments
- **Environment variables** — basic, required, redacted, lazy evaluation
- **Special directives** — `_.path`, `_.file`, `_.source` for PATH, dotenv, and script sourcing
- **Profiles** — `MISE_ENV`-based environment-specific configuration
- **Templates** — Tera templating with functions, filters, and path operations

### Configuration
- **Hooks** — cd, enter, leave, preinstall, postinstall triggers
- **File watchers** — watch patterns with automatic re-execution
- **Settings** — 100+ configuration options with env var overrides
- **Hierarchical config** — file precedence, merge behavior, profiles

## Installation

Install from the official Claude Code plugin directory:

```bash
claude plugin install mise
```

Or install directly from this repository:

```bash
claude plugin install --from https://github.com/brentmitchell25/mise-plugin.git
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
"pipx:psf/black" = "latest"
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

[settings]
env_shell_expand = true
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
| **Tasks** | All fields (run, depends, sources, outputs, usage, extends, timeout, etc.), structured run/depends, file tasks, remote tasks |
| **Usage Spec** | Full arg/flag/complete reference with all attributes, env var access patterns, bash expansion |
| **Dev Tools** | 18 backends, per-tool options, version formats, backend-specific config (github, http, cargo, pipx, etc.) |
| **Environments** | env vars, _.path/_.file/_.source directives, profiles, required/redacted vars, templates |
| **Hooks** | cd/enter/leave/preinstall/postinstall hooks, file watchers |
| **Configuration** | File hierarchy, 100+ settings, merge behavior, minimum version |
| **Best Practices** | DO/DON'T patterns, complete examples, common gotchas |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [mise](https://mise.jdx.dev/) installed in your development environment

## How It Works

This plugin provides a single skill file (`SKILL.md`) that Claude reads when it detects mise-related context in your conversation. The skill contains comprehensive reference material covering the entire mise ecosystem — tasks, dev tools, environments, hooks, and configuration.

The skill activates automatically when you mention mise, edit `mise.toml` files, or work with `mise-tasks/` directories.

## Updating the Skill

This repo includes a `mise.toml` with tasks for updating the skill as mise docs evolve:

```bash
mise run update:skill    # Re-crawl mise docs and update SKILL.md
```

See the [mise.toml](mise.toml) for the full update workflow.

## License

[MIT](LICENSE)
