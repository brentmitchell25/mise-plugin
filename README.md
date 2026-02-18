# mise Plugin for Claude Code

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Use Cases](#use-cases)
- [Advanced Examples](#advanced-examples)
  - [Multi-Environment Terraform Deployment](#1-multi-environment-terraform-deployment-with-dynamic-completions)
  - [Dockerized Microservice Pipeline](#2-dockerized-microservice-build-pipeline-with-caching)
  - [Release Automation](#3-full-release-automation-with-changelog-generation)
  - [Database Management Suite](#4-database-management-suite-with-migration-tracking)
  - [Kubernetes Deployment Orchestration](#5-kubernetes-deployment-orchestration-with-namespace-management)
- [Requirements](#requirements)
- [How It Works](#how-it-works)
- [License](#license)

---

## Overview

A Claude Code plugin that teaches Claude how to create and configure [mise](https://mise.jdx.dev/) tasks correctly. When installed, Claude will use the `usage` field for all task arguments (never shell-native `$1`/`$@` patterns), generate proper TOML-based and file-based tasks, configure dependencies and caching, and follow mise best practices.

This plugin is a pure skill - no commands, hooks, or MCP servers. It activates automatically whenever you ask Claude about mise task configuration.

## Features

- **Strict `usage` field enforcement** - all task arguments use mise's cross-platform `usage` spec, never shell-native `$1`/`$@`
- **TOML-based and file-based tasks** - supports both `mise.toml` inline tasks and executable script files with `#MISE`/`#USAGE` headers
- **Argument handling** - positional args, flags, choices validation, custom completions, variadic args, defaults
- **Task dependencies** - `depends`, `depends_post`, `wait_for` with parallel execution
- **Caching** - `sources`/`outputs` for last-modified checking and rebuild avoidance
- **Environment configuration** - env vars, `.env` file loading, PATH manipulation, required vars, redaction
- **Task namespacing** - subdirectory-based grouping (e.g., `test:unit`, `test:e2e`)
- **Remote tasks** - fetch tasks from URLs or git repos
- **Monorepo support** - experimental cross-project task orchestration
- **Tool requirements** - automatic tool installation before task execution
- **Global configuration** - `task_config`, `vars`, output modes, timeouts

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

### 1. Deploy with environment selection and flags

Ask Claude: *"Create a mise task that deploys to dev/staging/prod with a force flag"*

Claude generates tasks using the `usage` field with `choices` validation and proper `${usage_*}` variable access:

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

### 2. File-based tasks for database migrations

Ask Claude: *"Set up file-based mise tasks for database migrations"*

Claude creates executable scripts with `#MISE`/`#USAGE` headers and `${usage_*}` env vars:

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

### 3. Build pipeline with caching and dependencies

Ask Claude: *"Configure a build pipeline with lint, test, build using caching"*

Claude creates tasks with `depends` for ordering and `sources`/`outputs` for cache-based rebuild avoidance:

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
# Also tag with git SHA
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

*"Create a release task that bumps semver, generates a changelog from conventional commits, tags the release, and optionally publishes to npm. It should support major/minor/patch bumps and dry-run mode."*

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

# Ensure clean working tree
if [ -n "$(git status --porcelain)" ]; then
  echo "Error: Working tree is not clean. Commit or stash changes first."
  exit 1
fi

# Calculate new version
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

# Generate changelog from conventional commits
last_tag=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
if [ -n "$last_tag" ]; then
  changelog=$(git log "${last_tag}..HEAD" --pretty=format:"- %s (%h)" --no-merges)
else
  changelog=$(git log --pretty=format:"- %s (%h)" --no-merges)
fi

echo ""
echo "## ${new_version} ($(date +%Y-%m-%d))"
echo ""
echo "$changelog"
echo ""

if [ -n "$dry" ]; then
  echo "[dry-run] Would bump to ${new_version}, tag, and push."
  exit 0
fi

# Update package.json version
jq --arg v "$new_version" '.version = $v' package.json > package.json.tmp
mv package.json.tmp package.json

# Prepend to CHANGELOG.md
{
  echo "## ${new_version} ($(date +%Y-%m-%d))"
  echo ""
  echo "$changelog"
  echo ""
  [ -f CHANGELOG.md ] && cat CHANGELOG.md
} > CHANGELOG.md.tmp
mv CHANGELOG.md.tmp CHANGELOG.md

# Commit, tag, push
git add package.json CHANGELOG.md
git commit -m "chore(release): ${new_version}"
git tag -a "v${new_version}" -m "Release ${new_version}"
git push && git push --tags

if [ -n "${usage_publish:-}" ]; then
  echo "Publishing to npm..."
  npm publish ${pre:+--tag "$pre"}
fi

echo "Released v${new_version}"
```

### 4. Database management suite with migration tracking

*"Set up a complete database management task suite: migrate up/down, seed, reset, dump, and restore. PostgreSQL, with connection string from environment and safety confirmations on destructive ops."*

```toml
[env]
DATABASE_URL = { required = "Set PostgreSQL connection string (e.g., postgres://user:pass@localhost:5432/mydb)" }

[tasks."db:migrate"]
description = "Run database migrations"
usage = '''
arg "[direction]" default="up" choices "up" "down" help="Migration direction"
flag "-n --steps <n>" default="0" help="Number of steps (0 = all pending)"
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

[tasks."db:create-migration"]
description = "Create a new migration file pair"
usage = 'arg "<name>" help="Migration name (e.g., add_users_table)"'
run = '''
#!/usr/bin/env bash
set -euo pipefail
timestamp=$(date +%Y%m%d%H%M%S)
name="${usage_name?}"
mkdir -p db/migrations
touch "db/migrations/${timestamp}_${name}.up.sql"
touch "db/migrations/${timestamp}_${name}.down.sql"
echo "Created:"
echo "  db/migrations/${timestamp}_${name}.up.sql"
echo "  db/migrations/${timestamp}_${name}.down.sql"
'''

[tasks."db:seed"]
description = "Seed database with test data"
usage = '''
flag "--env <environment>" default="development" choices "development" "test" help="Seed data set"
flag "--truncate" help="Truncate tables before seeding"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
seed_file="db/seeds/${usage_env:-development}.sql"
[ ! -f "$seed_file" ] && echo "No seed file: $seed_file" && exit 1
if [ -n "${usage_truncate:-}" ]; then
  echo "Truncating tables..."
  psql "$DATABASE_URL" -f db/seeds/truncate.sql
fi
psql "$DATABASE_URL" -f "$seed_file"
echo "Seeded from $seed_file"
'''

[tasks."db:reset"]
description = "Drop all tables and re-run migrations + seeds"
depends_post = ["db:migrate", "db:seed"]
confirm = "This will DROP ALL TABLES and re-run migrations. Continue?"
run = 'psql "$DATABASE_URL" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"'

[tasks."db:dump"]
description = "Dump database to SQL file"
usage = '''
flag "-o --output <file>" default="db/dump.sql" help="Output file path"
flag "--data-only" help="Dump data only, no schema"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
pg_dump "$DATABASE_URL" \
  ${usage_data_only:+--data-only} \
  > "${usage_output:-db/dump.sql}"
echo "Dumped to ${usage_output:-db/dump.sql}"
'''

[tasks."db:restore"]
description = "Restore database from SQL dump"
usage = 'arg "<file>" help="SQL dump file to restore"'
confirm = "Restore database from {{usage.file}}? This will overwrite current data."
run = 'psql "$DATABASE_URL" -f "${usage_file?}"'
```

### 5. Kubernetes deployment orchestration with namespace management

*"Create mise tasks for Kubernetes deployments. I need tasks to switch context, deploy services, check rollout status, rollback, and port-forward — all with namespace and cluster awareness."*

```toml
[vars]
default_namespace = "default"

[tasks."k8s:ctx"]
description = "Switch Kubernetes context and namespace"
usage = '''
arg "<cluster>" help="Cluster context name"
complete "cluster" run="kubectl config get-contexts -o name"
flag "-n --namespace <ns>" default="default" help="Target namespace"
complete "namespace" run="kubectl get namespaces -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}'"
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
arg "[tag]" default="latest" help="Image tag to deploy"
flag "-n --namespace <ns>" help="Override namespace"
flag "--dry-run" help="Render manifests without applying"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
svc="${usage_service?}"
tag="${usage_tag:-latest}"
ns="${usage_namespace:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
ns="${ns:-default}"

echo "Deploying ${svc}:${tag} to namespace ${ns}"

if [ -n "${usage_dry_run:-}" ]; then
  kustomize build "k8s/services/${svc}" | \
    sed "s|:latest|:${tag}|g" | \
    kubectl apply --dry-run=client -n "$ns" -f -
  exit 0
fi

kustomize build "k8s/services/${svc}" | \
  sed "s|:latest|:${tag}|g" | \
  kubectl apply -n "$ns" -f -
mise run k8s:status "$svc" -n "$ns"
'''

[tasks."k8s:status"]
description = "Check rollout status for a deployment"
usage = '''
arg "<service>" help="Service to check"
complete "service" run="kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}'"
flag "-n --namespace <ns>" help="Override namespace"
flag "-w --watch" help="Watch until rollout completes"
'''
run = '''
#!/usr/bin/env bash
set -euo pipefail
ns="${usage_namespace:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
kubectl rollout status deployment/"${usage_service?}" \
  -n "${ns:-default}" \
  ${usage_watch:+--watch}
'''

[tasks."k8s:rollback"]
description = "Rollback a deployment to previous revision"
usage = '''
arg "<service>" help="Service to rollback"
complete "service" run="kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}'"
flag "-n --namespace <ns>" help="Override namespace"
flag "--revision <rev>" help="Specific revision number to rollback to"
'''
confirm = "Rollback {{usage.service}} to previous revision?"
run = '''
#!/usr/bin/env bash
set -euo pipefail
ns="${usage_namespace:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
kubectl rollout undo deployment/"${usage_service?}" \
  -n "${ns:-default}" \
  ${usage_revision:+--to-revision="$usage_revision"}
mise run k8s:status "${usage_service?}" -n "${ns:-default}" -w
'''

[tasks."k8s:logs"]
description = "Tail logs for a service"
usage = '''
arg "<service>" help="Service to tail logs for"
complete "service" run="kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}'"
flag "-n --namespace <ns>" help="Override namespace"
flag "-f --follow" help="Stream logs"
flag "--since <duration>" default="1h" help="Show logs since duration (e.g., 5m, 1h)"
flag "-c --container <name>" help="Specific container in pod"
'''
raw = true
run = '''
#!/usr/bin/env bash
set -euo pipefail
ns="${usage_namespace:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
kubectl logs -l "app=${usage_service?}" \
  -n "${ns:-default}" \
  --since="${usage_since:-1h}" \
  ${usage_follow:+-f} \
  ${usage_container:+-c "$usage_container"} \
  --tail=200
'''

[tasks."k8s:forward"]
description = "Port-forward a service locally"
usage = '''
arg "<service>" help="Service to forward"
complete "service" run="kubectl get svc -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}'"
flag "-p --port <port>" default="8080" help="Local port"
flag "--remote <port>" default="80" help="Remote service port"
flag "-n --namespace <ns>" help="Override namespace"
'''
raw = true
run = '''
#!/usr/bin/env bash
set -euo pipefail
ns="${usage_namespace:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
echo "Forwarding localhost:${usage_port:-8080} → ${usage_service?}:${usage_remote:-80}"
kubectl port-forward "svc/${usage_service?}" \
  "${usage_port:-8080}:${usage_remote:-80}" \
  -n "${ns:-default}"
'''
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [mise](https://mise.jdx.dev/) installed in your development environment

## How It Works

This plugin provides a single skill file (`SKILL.md`) that Claude reads when it detects mise-related context in your conversation. The skill contains:

1. **Strict enforcement rules** - Claude will never generate tasks with shell-native argument patterns
2. **Complete syntax reference** - all `usage` field syntax for args, flags, completions, and choices
3. **Configuration reference** - every task configuration key with types and descriptions
4. **Best practices** - DO/DON'T patterns for reliable task definitions
5. **Full examples** - complete TOML and file-based task examples ready to adapt

The skill activates automatically when you mention mise tasks, edit `mise.toml` files, or work with `mise-tasks/` directories.

## License

[MIT](LICENSE)
