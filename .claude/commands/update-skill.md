Update the mise plugin's skill (SKILL.md and its reference files) by crawling the latest mise documentation using parallel research agents.

## Process

1. Read `skills/mise/SKILL.md` and the file list in `skills/mise/references/` to understand the existing structure
2. Establish the "since last update" baseline: run `git log -1 --format=%cd --date=short -- skills/mise/` for the last-update date (and note the current `version` in `.claude-plugin/plugin.json`)
3. Launch 6 parallel research agents — 5 documentation crawlers plus 1 release-notes auditor
4. After all agents return, consolidate findings into the skill files
5. Run the `mise run lint` task to validate the result

## Research Agents

Launch ALL 6 of these agents in parallel using the Agent tool. Agents 1–5 use WebFetch to crawl the specified doc pages; Agent 6 uses the `gh` CLI to audit release notes. Each returns a structured summary of everything it finds.

### Agent 1: Tasks (TOML + File)

Crawl these pages and return all task-related documentation:
- https://mise.jdx.dev/tasks/toml-tasks.html
- https://mise.jdx.dev/tasks/file-tasks.html
- https://mise.jdx.dev/tasks/task-configuration.html

Focus on: all task fields (especially new ones), structured run/depends syntax, extends/inheritance, timeout, file task headers (#MISE/#USAGE), remote tasks, task namespacing, CLI flags for `mise run`, output modes. Return every field name, type, and default value you find.

### Agent 2: Dev Tools & Backends

Crawl these pages and return all dev tool documentation:
- https://mise.jdx.dev/dev-tools/
- Follow links to individual backend pages (aqua, github, npm, cargo, pipx, etc.)

Focus on: all backends with their specific options, per-tool options (version, os, postinstall, install_env), TOML syntax variants (simple string, array, table), version formats (exact, prefix, latest, ref, path, sub-N), shims vs PATH mode, aliases, registry, CLI commands (mise use, mise install, mise ls, mise ls-remote, mise which, mise x, mise reshim).

### Agent 3: Environments & Settings

Crawl these pages and return all environment/settings documentation:
- https://mise.jdx.dev/environments/
- https://mise.jdx.dev/configuration/settings.html
- https://mise.jdx.dev/configuration.html

Focus on: env section syntax, special directives (_.path, _.file, _.source with all variants), profiles/MISE_ENV, Tera templates (all functions, filters, tests, available variables), required/redacted vars, shell expansion (env_shell_expand), env CLI commands, file hierarchy and merge behavior, all settings with types/defaults/env var overrides.

### Agent 4: Hooks, Watchers, IDE, CI/CD

Crawl these pages and return all hooks/integration documentation:
- https://mise.jdx.dev/hooks.html
- https://mise.jdx.dev/ide-integration.html
- https://mise.jdx.dev/continuous-integration.html
- https://mise.jdx.dev/cli/ (browse the CLI reference)

Focus on: hook types (cd, enter, leave, preinstall, postinstall) with env vars available to each, watch_files syntax, shell hooks, IDE integration patterns, CI/CD setup, full CLI command listing with flags.

### Agent 5: Usage Spec Reference

Crawl these pages and return comprehensive usage spec documentation:
- https://usage.jdx.dev/spec/reference/
- Follow links to: arg, flag, cmd, complete, config sub-pages

Focus on: ALL arg attributes with types/defaults, ALL flag attributes with types/defaults, cmd structure, complete blocks (run command, Tera templates, descriptions), config block, the `usage_` env var naming convention, value types for boolean/count/variadic, shebang and comment syntax by language.

### Agent 6: Release Notes & Changelog (since last update)

This agent catches what the fixed documentation URLs above miss — brand-new features whose pages aren't in the crawl list, status changes buried in changelogs, renames, and changed defaults. Release notes are the authoritative record of what changed.

Use the `gh` CLI (NOT WebFetch — these live on GitHub):
- Baseline date: `git log -1 --format=%cd --date=short -- skills/mise/`
- List releases: `gh release list --repo jdx/mise --limit 40`
- For EVERY release newer than the baseline date, read its body: `gh release view <tag> --repo jdx/mise --json body -q .body`
- Also check the usage spec: `gh release list --repo jdx/usage --limit 10` (+ `gh release view` for new ones)

Report, grouped by impact, everything from the **Added / Changed / Deprecated / Removed** sections that affects how users author `mise.toml` / tasks / config. For each finding give: the release version, a one-line description, the exact config/CLI syntax if shown, and whether it needs a NEW SKILL section or an edit to an existing one. Prioritize and flag loudly:
- **Brand-new features** — config sections, backends, package managers, CLI commands, settings, tool/flag options — ESPECIALLY any with no doc page in the Agent 1–5 URL list (highest-value finds; include the doc URL if one now exists).
- **Experimental → stable** graduations (and anything newly marked experimental).
- **Renamed / aliased keys** (e.g. `experimental_monorepo_root` → `monorepo_root`).
- **Changed defaults** (e.g. `minimum_release_age`).
- **Deprecations & removals** WITH their version/date.

Ignore pure bug-fixes and internal/CI/dependency changes unless they change documented behavior.

## Consolidation

After all six agents return their findings, update the skill files under `skills/mise/`.

`SKILL.md` loads every time the skill triggers, so it holds only what applies to every mise request. Keep it under 500 lines, with these sections in this order:

1. **YAML Frontmatter** — `name: mise`; a description that names the kinds of mise work (config files, tasks, tools, environments, CLI) rather than listing individual features
2. **Table of Contents**
3. **STRICT ENFORCEMENT: Usage Field Required** — copy this section verbatim from the current file, do not modify it
4. **Overview** — what mise covers, the mise version the skill was verified against, the top-level `mise.toml` keys
5. **Reference Files** — one table row per reference file: link, when to read it, what it covers
6. **Best Practices** — DO/DON'T patterns, complete examples

Everything else lives in `skills/mise/references/`, one file per topic. Each file starts with its `##` section heading and a contents list, and links to other files by relative path (`dev-tools.md#lockfiles-miselock`, `../SKILL.md#best-practices`):

| File | Section |
|------|---------|
| `task-definition.md` | Task Definition Methods — TOML tasks, file tasks, namespaces, remote tasks |
| `usage-spec.md` | Task Arguments - Usage Spec Reference — arg/flag/complete attributes, env var access patterns |
| `task-configuration.md` | Task Configuration Reference — all fields with types, structured run/depends, extends, timeout, global task config, vars |
| `running-tasks.md` | Running Tasks (CLI) — commands and flags |
| `task-dependencies.md` | Task Dependencies and Freshness — depends/depends_post/wait_for, sources/outputs, skip_if, redactions |
| `task-caching.md` | Task Output Caching — local artifact cache, cache inputs, remote task cache |
| `task-tracing.md` | Task Tracing with OpenTelemetry |
| `dev-tools.md` | Dev Tools Management — backends, TOML syntax, per-tool options, backend-specific config, shims, aliases, lockfiles |
| `environments.md` | Environment Configuration — basic vars, special directives, profiles, required/redacted, secrets, Tera templates |
| `hooks.md` | Hooks and Watchers — hook types with env vars, watch_files, shell hooks |
| `daemons.md` | Project Daemons (`[daemons]`) — daemons, providers, presets, ports |
| `doctor.md` | Project Diagnostics (`[doctor]`) |
| `wrappers.md` | Command Wrappers (`[wrappers]`) |
| `sandboxing.md` | Sandboxing and Safe Mode |
| `bootstrap.md` | Machine Bootstrap (Developer Setup) — `[bootstrap]`, dotfiles, dotfile groups, dotfiles history |
| `oci.md` | OCI Container Images |
| `settings.md` | Configuration and Settings — file hierarchy, key settings reference, min_version, automatic env vars |
| `deps.md` | Dependency Preparation (`[deps]`) |
| `monorepo.md` | Monorepo Tasks and Workspace Graph |
| `deprecations.md` | Deprecation Calendar — removal versions taken from mise's source |

Suffix a section title with "(Experimental)" while its feature is experimental, and drop the suffix when the feature stabilizes.

> **Reconcile release notes against the doc crawl.** When Agent 6's findings conflict with Agents 1–5 on experimental/stable status, renamed keys, or changed defaults, the **release notes win** (doc pages lag behind releases). If a release introduced a major feature with no home in the structure above (e.g. Machine Bootstrap / dotfiles), ADD a new reference file for it plus a matching row in SKILL.md's Reference Files table. Verify any contested default or status with a direct `WebFetch`/`gh` check before writing.

## Rules

- **NEVER modify the STRICT ENFORCEMENT section** — copy it exactly from the current file
- Include ALL configuration keys/fields with their types and defaults
- Include practical code examples for each section
- Note any deprecations with dates
- Use tables for reference material, code blocks for examples
- The file must be a practical reference that enables Claude to generate correct mise configurations
- Start every file with a table of contents (SKILL.md) or contents list (reference files)
- Before keeping a negative claim (the Still Not Implemented table, anything described as hard-erroring, a no-op, or ignored), re-check it against the current mise binary; a claim that has since become false sends Claude around a feature that works
- If any crawled page is unreachable, note what couldn't be fetched and use the existing content for that section
- Reconcile Agent 6 (release notes) against Agents 1–5 (docs); release notes are authoritative for status changes (experimental↔stable), renames, and changed defaults
- Add a new reference file (and a Reference Files row) when a release introduced a major feature not covered by the files above
- Write each change into the section it belongs to, as a current fact with the mise version that introduced it. Leave out "new since the last update" summaries and notes about what earlier versions of this skill said — the model reading the skill never saw those versions

## Validation

After writing the file, run `mise run lint` to validate the structure. Fix any issues it reports.

$ARGUMENTS
