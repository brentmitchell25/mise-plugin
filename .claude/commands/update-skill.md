Update the mise plugin's SKILL.md by crawling the latest mise documentation using parallel research agents.

## Process

1. Read the current `skills/mise/SKILL.md` to understand the existing structure
2. Establish the "since last update" baseline: run `git log -1 --format=%cd --date=short -- skills/mise/SKILL.md` for the last-update date (and note the current `version` in `.claude-plugin/plugin.json`)
3. Launch 6 parallel research agents — 5 documentation crawlers plus 1 release-notes auditor
4. After all agents return, consolidate findings into an updated SKILL.md
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
- Baseline date: `git log -1 --format=%cd --date=short -- skills/mise/SKILL.md`
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

After all 5 agents return their findings, write the updated `skills/mise/SKILL.md` with this exact section structure:

1. **YAML Frontmatter** — `name: mise`, description covering all areas
2. **Table of Contents** — linking to all sections below
3. **STRICT ENFORCEMENT: Usage Field Required** — copy this section verbatim from the current file, do not modify it
4. **Overview** — brief description of what mise covers
5. **Task Definition Methods** — TOML tasks, file tasks, namespaces, remote tasks
6. **Task Arguments - Usage Spec Reference** — comprehensive arg/flag/complete reference with all attributes, env var access patterns
7. **Task Configuration Reference** — ALL fields with types, structured run/depends, extends, timeout, global task config, vars
8. **Running Tasks (CLI)** — all commands and flags
9. **Task Dependencies and Caching** — depends/depends_post/wait_for, sources/outputs, redactions
10. **Dev Tools Management** — backends, TOML syntax, per-tool options, backend-specific config, shims, aliases
11. **Environment Configuration** — basic vars, special directives, profiles, required/redacted, Tera templates
12. **Hooks and Watchers** — hook types with env vars, watch_files, shell hooks
13. **Configuration and Settings** — file hierarchy, key settings reference, min_version, automatic env vars
14. **Prepare Feature (Experimental)** — if still experimental
15. **Monorepo Tasks (Experimental)** — if still experimental
16. **Best Practices** — DO/DON'T patterns, complete examples

> **Reconcile release notes against the doc crawl.** When Agent 6's findings conflict with Agents 1–5 on experimental/stable status, renamed keys, or changed defaults, the **release notes win** (doc pages lag behind releases). If a release introduced a major feature with no home in the structure above (e.g. Machine Bootstrap / dotfiles), ADD a new top-level section for it plus a matching Table of Contents entry. Verify any contested default or status with a direct `WebFetch`/`gh` check before writing.

## Rules

- **NEVER modify the STRICT ENFORCEMENT section** — copy it exactly from the current file
- Include ALL configuration keys/fields with their types and defaults
- Include practical code examples for each section
- Note any deprecations with dates
- Use tables for reference material, code blocks for examples
- The file must be a practical reference that enables Claude to generate correct mise configurations
- Always include a Table of Contents at the top of the file
- If any crawled page is unreachable, note what couldn't be fetched and use the existing content for that section
- Reconcile Agent 6 (release notes) against Agents 1–5 (docs); release notes are authoritative for status changes (experimental↔stable), renames, and changed defaults
- Add a new top-level section (and TOC entry) when a release introduced a major feature not covered by the structure above

## Validation

After writing the file, run `mise run lint` to validate the structure. Fix any issues it reports.

$ARGUMENTS
