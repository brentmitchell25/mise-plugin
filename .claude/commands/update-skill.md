Update the mise plugin's SKILL.md by crawling the latest mise documentation using parallel research agents.

## Process

1. Read the current `skills/mise/SKILL.md` to understand the existing structure
2. Launch 5 parallel research agents to crawl the latest mise documentation
3. After all agents return, consolidate findings into an updated SKILL.md
4. Run the `mise run lint` task to validate the result

## Research Agents

Launch ALL 5 of these agents in parallel using the Agent tool. Each agent should use WebFetch to crawl the specified pages and return a structured summary of everything it finds.

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

## Rules

- **NEVER modify the STRICT ENFORCEMENT section** — copy it exactly from the current file
- Include ALL configuration keys/fields with their types and defaults
- Include practical code examples for each section
- Note any deprecations with dates
- Use tables for reference material, code blocks for examples
- The file must be a practical reference that enables Claude to generate correct mise configurations
- Always include a Table of Contents at the top of the file
- If any crawled page is unreachable, note what couldn't be fetched and use the existing content for that section

## Validation

After writing the file, run `mise run lint` to validate the structure. Fix any issues it reports.

$ARGUMENTS
