## Dependency Preparation (`[deps]`)

**Contents**

- [Dependency Preparation ([deps])](#dependency-preparation-deps)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

**Experimental.** Ensures project dependencies (npm packages, Python venvs, Go modules, …) are installed before task execution. This replaces the old `[prepare]` section — **there is no `prepare` key in the schema.**

```bash
mise deps            # install project deps
mise deps --list     # show active/inactive providers with a reason
mise deps <provider> --explain
mise deps add <pkg>  # -D/--dev for dev dependencies
mise deps remove <pkg>
mise deps install --force
mise deps --skip npm -n                 # skip a provider; -n = dry run
mise deps --monorepo                    # across monorepo config roots
mise deps --monorepo --only //apps/api:uv
```

```toml
[deps.npm]
auto = true

[deps.custom]
sources = ["schema/*.graphql"]
outputs = ["src/generated/"]
run = "npm run codegen"
```

**Built-in providers:** `npm`, `yarn`, `pnpm`, `bun`, `deno`, `aube`, `go`, `pip`, `poetry`, `uv`, `bundler`, `composer`, `dart`, `flutter`, `git-submodule`.

Built-in providers activate only when **explicitly configured in `mise.toml` AND their lockfile exists** — except `go` (needs `go.mod`), `pip` (`requirements.txt`), `dart`/`flutter` (`pubspec.yaml`), and `git-submodule` (a non-empty `.gitmodules`). Setting `sources`/`outputs` on a provider **replaces** its defaults; each provider also takes a `timeout`. The `status.show_deps_stale` setting surfaces stale deps in the prompt status. Safe mode ignores `[deps]`. `mise deps --list` reports inactive providers with the reason (e.g. `inactive (missing package-lock.json)`) instead of silently omitting them. Providers with `auto = true` run automatically before `mise x` and `mise run`; disable for one invocation with `--no-deps`.

In monorepos, provider IDs are qualified by config root (`//apps/api:uv`) so repeated provider names don't collide. Deps **task providers** are gated behind `experimental = true` (2026.7.12+).
