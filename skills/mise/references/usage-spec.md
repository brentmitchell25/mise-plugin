## Task Arguments - Usage Spec Reference

**Contents**

- [Positional Arguments (arg)](#positional-arguments-arg)
- [Flags (flag)](#flags-flag)
- [Custom Completions (complete)](#custom-completions-complete)
- [Accessing Arguments in Scripts](#accessing-arguments-in-scripts)
- [File Task Headers](#file-task-headers)
- [Subcommands (cmd block)](#subcommands-cmd-block)
- [Command Effects (effect=)](#command-effects-effect)
- [Spec-Level Metadata](#spec-level-metadata)
- [Validation and Env-Backed Choices](#validation-and-env-backed-choices)
- [Still Not Implemented (do not use)](#still-not-implemented-do-not-use)
- [Flag Groups, Flagsets, and Outputs (v6)](#flag-groups-flagsets-and-outputs-v6)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

The `usage` field uses [KDL-inspired syntax](https://usage.jdx.dev/) to define arguments, flags, and completions.

> 🔴 **Version context (verified 2026-11-01).** mise 2026.10.25 bundles **usage-lib 7.1.4** (2026.10.4 → 6.12.1, 2026.10.6 → 7.0.0, 2026.10.12 → 7.0.1, 2026.10.13 → 7.0.2, 2026.10.16 → 7.1.0, 2026.10.18 → 7.1.1, 2026.10.20 → 7.1.2, 2026.10.21 → 7.1.3, 2026.10.25 → 7.1.4; usage 7.1.5 is not in any mise release yet; mise's own CLI spec still declares `min_usage_version "6.11"`). **The usage 6 → 7 major bump changes nothing a mise task author writes** — its breaking changes (derive macros no longer read `Cargo.toml`, `usage generate fig` removed, `usage-argv` internals) are Rust-library/CLI concerns; no spec syntax was added, removed, or renamed. In **2026.8.11** mise migrated its *own* CLI parser, help output, and shell completions from clap to usage-rs.
>
> **Nothing is compiled out.** Since **2026.8.13** mise depends on `usage-cli`, whose manifest enables usage-lib's `validation` and `unstable_choices_env` features; Cargo unifies features across the build, so mise's single usage-lib gets them. `validate=` and `choices env=` **work and are enforced** (see [Validation and Env-Backed Choices](#validation-and-env-backed-choices)).
>
> **usage 6.11–6.12 change how specs are written:** `choices run="cmd"` (command-backed choices that validate), `complete` nested inside an `arg`/`flag`, `complete … delegate="tool"`, `$usage_cmd` for the chosen subcommand, `default_subcommand_on_empty`, and `logo`. All re-verified on mise 2026.10.25 below.
>
> 🔴 **`choices` MUST go in a child block — the inline form is silently ignored.** `arg "<env>" choices "dev" "prod"` parses (KDL reads the bare `choices` as just another positional string) but **validates nothing**: `mise run t bogus` runs with `usage_env=bogus`, `--help` lists no possible values, and `mise tasks validate` passes. Always write `arg "<env>" { choices "dev" "prod" }` — then `bogus` is rejected with `Invalid choice for arg env: bogus, expected one of dev, prod`. (Verified on 2026.10.25.)
>
> 🔒 **Dynamic completion needs trust (2026.10.19).** `complete run=`, `complete delegate=`, `choices run=`, and `mount` run **only for trusted configs**; untrusted projects fall back to file completion (static `choices`, task names, and help still complete). Safe mode keeps them off even when trusted. A project with only a `mise-tasks/` directory needs `mise trust .`. Candidates containing control characters are dropped, and all candidates are treated as **plain text** (never shell-evaluated).
>
> 🔴 **TOML `usage` strings are Tera-rendered by mise before usage parses them.** Any `{{ … }}` meant for usage (e.g. `{{ words[PREV] }}` in `complete run=`) must be wrapped in `{% raw %}…{% endraw %}` — see [Tera Rendering of TOML usage Strings](#tera-rendering-of-toml-usage-strings). File-task `#USAGE` headers are not rendered.

### Positional Arguments (`arg`)

Every attribute is accepted both as a prop (`arg "<f>" key=value`) and as a child node (`arg "<f>" { key value }`), except `choices` which is child-node only.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (name) | string | *required* | `"<name>"` = required, `"[name]"` = optional. `"<file>"`/`"<path>"` triggers file completion, `"<dir>"` triggers dir completion. |
| `help` | string | none | Short help text shown with `-h` |
| `long_help` / `help_long` | string | none | Extended help text shown with `--help` |
| `help_md` | string | none | Markdown-only help (docs generation) |
| `help_heading` | string | none | Group the arg under a heading in help (`help_heading="Inputs"` → `Inputs:` section; verified) |
| `required` | boolean | `#true` for `<x>`, `#false` for `[x]` | **Forced to `#false` whenever `default` is set** |
| `default` | string (prop) / string-or-list (child) | none | Default value if not provided. `default=""` sets to empty string (different from unset). A multi-value default (`default { "a"; "b" }`) only takes effect on **variadic** args — on a single-value arg only the first value is used. |
| `env` | string | none | Environment variable that can provide this arg's value. Priority: CLI > env > default. |
| `var` | boolean | `#false` | Variadic mode (accept multiple values). Shorthand: `"<name>..."` |
| `var_min` | integer | none | Minimum values when variadic |
| `var_max` | integer | none | Maximum values when variadic |
| `choices` | **child node only** | none | Restrict to enumerated set — `arg "<x>" { choices "a" "b" }`. **The inline `arg "<x>" choices "a" "b"` form is silently ignored.** `choices strict=#false "a" "b"` **suggests** without rejecting other values (v6); `ignore_case=#true` accepts any case but does **not** normalise the value. `choices env="VAR"` reads the set from an env var; `choices run="cmd"` from a command's output (6.12) — see [Dynamic Choices](#dynamic-choices-env-and-run). |
| `validate` / `validate_error` | string (expr) | none | Expression checked against `value` (e.g. `int(value) >= 1`, `value matches '^[a-z]+$'`, plus `date(value) > date('2030-01-01')`, `fromJSON(value).a == 1`, `fromBase64(value) == 'hi'` — all verified); `validate_error` is the message. **Works in mise** (see [Validation and Env-Backed Choices](#validation-and-env-backed-choices)). |
| `complete` | child node | none | **6.12.** Completion attached directly to this arg — `arg "<x>" { complete run="…" }`. Beats a same-named top-level `complete`. |
| `effect` | enum / child node | none | `read` \| `write` \| `destructive` — raises the command's effect |
| `double_dash` | enum | `optional` | `"required"`, `"optional"`, `"automatic"`, `"preserve"`. **Now enforced** under v6. |
| `hide` | boolean | `#false` | Exclude from help output |
| `display_order` | integer | none | **v6.** Explicit ordering in help; parse order is unchanged |
| `delimiter` | string | none | **v6.** Split one word into several values (`--tags a,b,c`). Variadic args only; applied *before* choices/validation. |
| `allow_negative_numbers` | boolean | `#false` | **v6.** Accept a leading-minus number as a value; `--force` stays flag-like |
| `value_terminator` | string | none | **v6.** Token that ends a variadic arg without being stored |
| `hide_default_value`, `hide_env`, `hide_env_values`, `hide_possible_values`, `hide_short_help`, `hide_long_help` | boolean | `#false` | **v6.** Presentation-only; defaults, env fallback, and validation stay active |
| `value_names` | child node | none | **v6.** Relabel values / declare fixed arity |
| `env_fallback` | child node | none | **v6.** Additional env vars in precedence order |
| `deprecated_env` | child node | none | **v6.** Compatibility env aliases, consulted last (mise prints no deprecation warning for them) |
| `note` / `warning` | child node | none | **v6.** Semantic callouts in long help and generated Markdown |

**Shorthands:** `<f>` required · `[f]` optional · `<f>...` ⇒ `var=#true` · `<-- f>` / `[-- f]` ⇒ `double_dash="required"`.

**Relationship child nodes (v6, valid on `arg` and `flag`)** — bare selectors name positionals, dashed selectors name flags:
`conflicts`, `requires`, `requires_if`, `required_if`, `required_if_eq`, `required_if_eq_all`, `required_unless`, `required_unless_all`.

```
arg "[request]" { requires "--mode" "--scope" }
arg "[token]"   { required_if_eq "--mode" "remote" }
arg "[sum]"     { required_unless_all "--stdin" "--file" }
```

> 🔴 **One target per property; several targets → child node.** `flag "--a" conflicts="--b --c"` (a property naming several targets) or a repeated property (`conflicts="--b" conflicts="--c"`) passes `mise tasks validate` on mise 2026.10.25 (usage-lib 7.1.4) but misbehaves: multi-target `conflicts` enforces nothing, a repeated one enforces only the last, and multi-target `requires`/`required_unless` can never be satisfied. usage 7.1.5 makes both forms load-time errors. Write `flag "--a" { conflicts "--b" "--c" }` — correct on every version. Applies to `conflicts`, `overrides`, `requires`, `required_if`, `required_unless`, `required_unless_all`.

**Environment precedence:** CLI argv → `env` → `env_fallback` (left to right) → `deprecated_env` → `default`.

**Examples:**

```
arg "<file>" help="Input file to process"
arg "[output]" default="out.txt" help="Output file"
arg "<files>" var=#true var_min=1 help="One or more files"
arg "<files>..." help="Shorthand variadic syntax"
arg "<env>" help="Target environment" { choices "dev" "staging" "prod" }   # braces REQUIRED
arg "<args>..." double_dash="automatic" help="Pass-through arguments"
arg "<file>" env="MY_FILE" help="Input file (or set MY_FILE)"
arg "<modes>..." { default { "fast"; "safe" } }     # multi-value default — variadic args only
arg "<port>" validate="int(value) >= 1 && int(value) <= 65535" validate_error="must be a valid port"
```

#### Dynamic Choices (`env=` and `run=`)

Both forms **validate** (unlike `complete`, which only suggests) and both feed tab-completion. Verified on mise 2026.10.25:

```
arg "<env>" { choices env="DEPLOY_ENVS" }                    # DEPLOY_ENVS="dev,staging prod"
arg "<svc>" { choices "all" run="ls services" }              # literal values may sit beside run= (6.12)
flag "--svc <svc>" { choices run="printf 'app\ndb\n'" }      # also on a flag's value
```

| Situation | Result |
|-----------|--------|
| `choices env=` — value in the list | accepted; the var is split on **commas and whitespace** |
| `choices env=` — value not in the list | `Invalid choice for arg env: qa, expected one of dev, staging, prod` |
| `choices env=` — var unset | `Invalid choice for arg env: dev, no choices resolved from env DEPLOY_ENVS` |
| `choices run=` — value not in output | `Invalid choice for arg svc: nope, expected one of all, app, db` |
| `choices run=` — command fails | `Could not check arg svc: x against its choices: exited with code 3`, then the command (`sh -c exit 3`) on a second line (task fails) |
| `--help` | does **not** run the command; prints ``[possible values: output of `ls services`]`` (with a literal value alongside, two lines: `[possible values: all]` and the `output of` line) |

> Prefer `choices run=` over `complete run=` whenever the set is derivable *and* invalid values should be rejected. Inside a TOML `usage` string, any `{{ }}` in the command must be `{% raw %}`-wrapped.

> ✅ **`double_dash="required"` is now enforced** (usage v6). A value offered before `--` errors with ``Argument <args> can only be set after a `--` separator``. Specs that "happened to work" under the old unenforced behavior will now error — re-verified on mise 2026.10.25.

**Variadic args in bash:**
```bash
# Values are a shell-escaped string in usage_files
eval "files=($usage_files)"
for f in "${files[@]}"; do
  echo "Processing: $f"
done
```

### Flags (`flag`)

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (definition) | string | *required* | `"-s --long"` (boolean) or `"-s --long <value>"` (with value). Short flag optional. Short flags must be exactly 1 char. |
| `help` | string | none | Short help text |
| `long_help` / `help_long` | string | none | Extended help for `--help` |
| `help_md` | string | none | Markdown-only help |
| `required` | boolean | `#false` | **Forced `#false` when `default` set** |
| `default` | string/bool | none | Default value. Boolean flags use `#true`/`#false`. Multi-value block form also accepted. |
| `env` | string | none | Environment variable backing the flag |
| `global` | boolean | `#false` | Available on all subcommands; also passed to `mount` |
| `count` | boolean | `#false` | Value = number of times used (e.g., `-vvv` = 3) |
| `var` | boolean | `#false` | Flag repeatable, collecting values |
| `var_min` / `var_max` | integer | none | Min/max values when `var=#true` |
| `negate` | string | none | Negative form (e.g., `"--no-color"`). Sets env var to the string **`false`** (still *set*). Without a `default`, an absent flag leaves the var **unset** — only `--no-x` yields `false`. |
| `effect` | enum | none | `read` \| `write` \| `destructive` |
| `allow_hyphen_values` | boolean | `#false` | Let a value-taking flag consume a following `-…` token as its value. Errors if the flag takes no value. |
| `deprecated` | bool \| string | none | `#true` → literal `"deprecated"`; a string is used as the message |
| `arg` | child node | none | Names the flag's value: `flag "--user" { arg "<user>" }` |
| `choices` | child node | none | Enumerated set. `choices strict=#false` suggests without rejecting. `env=` / `run=` forms work (see [Dynamic Choices](#dynamic-choices-env-and-run)). |
| `validate` / `validate_error` | string | none | Expression check on the flag's value — put it on the value's `arg` child: `flag "--port <p>" { arg "<p>" validate="…" }` |
| `complete` | child node | none | **6.12.** `flag "--out <path>" { complete type="dir" }` |
| `hide` | boolean | `#false` | Exclude from docs/completions |
| `alias` | child node | none | **v6 — now works.** Short/extra form: `flag "--user" { alias "-u" }`; supports `hide=#true` |
| `required_if` / `required_unless` | string | none | **v6 — now works and is enforced** |
| `required_if_eq` / `required_if_eq_all` / `required_unless_all` | child node | none | **v6.** Value-conditional requirements |
| `conflicts` / `requires` / `requires_if` | string \| child node | none | **v6.** Mutual exclusion / co-requirements. A string names **one** target — use the child node for several (see the relationship note above) |
| `overrides` | string | none | **v6.** Two flags override each other; last one wins |
| `exclusive` | boolean | `#false` | **v6.** Flag must be given on its own |
| `require_equals` | boolean | `#false` | **v6.** `--inspect=9229` accepted, `--inspect 9229` rejected |
| `default_missing` | string | none | **v6.** Value used for a bare `--color` vs an explicit `--color=never` |
| `value_optional` | boolean | `#false` | **v6.** Absent / bare / valued become three distinct states |
| `bool_value` | boolean | `#false` | **v6.** Allow explicit `--color=false` |
| `default_if` | child node | none | **v6.** Conditional default: `default_if "--json" "true"`; 3-argument equality form `default_if "--output" "json" "pretty"` (`--output json` ⇒ this flag = `pretty`; verified) |
| `delimiter` | string | none | **v6.** `--tags a,b,c` → three values |
| `allow_negative_numbers` | boolean | `#false` | **v6.** `--jobs -1` binds `-1` |
| `value_terminator` | string | none | **v6.** Token ending one variadic occurrence |
| `display_order` | integer | none | **v6.** Explicit order within its help section |
| `help_heading` | string | none | **v6.** Group this flag under a heading in help |
| `action` | enum | none | **v6.** `help` \| `help_short` \| `help_long` \| `help_all` \| `version`. In mise tasks every `help*` action (with or without `builtin=#true`) prints help through `mise ERROR`, exit 1 — rely on the built-in `-h`/`--help` |
| `env_fallback` / `deprecated_env` | child node | none | **v6.** Extra / legacy env sources |
| `note` / `warning` | child node | none | **v6.** Semantic callouts |
| `hide_default_value`, `hide_env`, `hide_env_values`, `hide_possible_values`, `hide_short_help`, `hide_long_help` | boolean | `#false` | **v6.** Presentation-only |

**Examples:**

```
flag "-v --verbose" help="Enable verbose output"
flag "-f --force" help="Skip confirmation prompts"
flag "--port <port>" default="8080" help="Server port"
flag "--color" negate="--no-color" default=#true help="Enable colors"
flag "-d --debug" count=#true help="Debug level (-ddd for max)"
flag "--include <pattern>" var=#true help="Include patterns (repeatable)"
flag "--include... <pattern>" help="Ellipsis notation: repeatable flag"
flag "--files <f>..." help="Several values in ONE occurrence: --files a b c ⇒ usage_files='a b c'"
flag "--range <start> <end>" help="Exactly two values per occurrence ⇒ usage_range='1 2'"
flag "--shell <shell>" { choices "bash" "zsh" "fish" }
flag "--color" env="MYCLI_COLOR" help="Backed by env var"
flag "-a --args <ARGS>" allow_hyphen_values=#true help="Pass-through args (e.g. -a -destroy)"
flag "--old-flag" deprecated="use --new-flag instead"
flag "--clear" effect="destructive" help="Delete stored logs"
```

**Count flags:** `-vvv` sets `$usage_verbose` to `3`. Short flags chain: `-abc` = `-a -b -c`.

> ⚠️ **Never put `default` on a `count` flag.** A bare integer (`default 0`) fails to parse (`expected string`), and `default="0"` is coerced through the boolean path, yielding `usage_verbose=false` when the flag is absent. Use `${usage_verbose:-0}` in the script instead. (Still broken on 2026.10.25.)

**Negate flags:** `flag "--color" negate="--no-color" default=#true` — `--no-color` sets `$usage_color` to `false`. Help renders these as `--color / --no-color`.

**v6 examples (all re-verified working in mise 2026.10.25):**

```
flag "--user" { alias "-u" }                      # short form as a child node
flag "--file <f>" required_unless="--stdin"       # enforced
flag "--dump" exclusive=#true                     # must be given alone
flag "--inspect <p>" require_equals=#true         # --inspect=9229 only
flag "--color <w>" default_missing="always"       # bare --color ⇒ "always"
flag "--tags <t>" var=#true delimiter=","         # --tags a,b,c ⇒ 3 values
flag "--backend <b>" { choices strict=#false "core" "git" }   # suggest, don't reject
```

### Custom Completions (`complete`)

Provide **dynamic tab-completion** for arguments. `complete` only *suggests*; to also *reject* bad values use [`choices run=`](#dynamic-choices-env-and-run).

```
arg "<svc>" { complete run="ls services" }        # 6.12: nested inside the arg (preferred in mise tasks)
flag "--out <path>" { complete type="dir" }       # 6.12: nested on a flag
complete "<arg_name>" run="<cmd, one value per line>"   # top-level, matched by arg name
complete "<arg_name>" type="dir"
complete "<arg_name>" delegate="terraform"       # 6.12: hand off to another tool's completion
complete "<arg_name>" delegate="git --no-pager"  # fixed leading args are allowed
```

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `run` | string (Tera) | none | Shell command; one candidate per line |
| `type` | string | none | Built-in completer (all verified on 2026.10.25) — see table below |
| `delegate` | string | none | **6.12.** Complete the remaining words with *that* command's own shell completion; may carry fixed args (`"git --no-pager"`). zsh and bash verified; the delegated shell gets **3 s** to answer; the wrapped tool's completion must be in bash-completion's loader dir (zsh: default `fpath`) — `.bashrc`/`.zshrc` setup isn't seen. PowerShell/Nushell unsupported. With `double_dash="automatic"` every word after the first passes straight through. Completion-only — the task runs normally. |
| `descriptions` | boolean | `#false` | Parse `value:description` output |

| `type` | Result |
|--------|--------|
| `path` / `file` | Files (identical) |
| `path:toml,yaml` / `file:toml` | Extension filter |
| `dir` | Directories |
| `command` | Commands known to the shell (incl. PATH) |
| `executable` | Executables plus directories |
| `command_args` | A command first, then paths |
| `hostname` | Hosts from the environment and `/etc/hosts` |
| `username` | Users from `/etc/passwd` (on macOS the file's `#` comment lines also appear until you type a prefix) |
| `url`, `email` | No candidates; file fallback suppressed |
| `none` | No candidates; no file fallback |
| `unknown` | No candidates; file fallback kept |
| `config_keys`, `config_values` | `config_keys` gives no candidates in mise tasks; `config_values` falls back to files — don't use either |

Short flags show their help in completion menus (usage 7.0.2: `-v  Verbose output` in zsh/fish).

`run`, `type`, and `delegate` are **mutually exclusive** — combining them errors with `can set only one of run, type or delegate`.

**Key rules (verified on mise 2026.10.25):**
- **Nest `complete` inside the `arg` in mise tasks.** A nested `complete` always wins over a same-named top-level one and keeps the completer next to its arg. Since mise 2026.10.20 (usage 7.1.2) a top-level `complete "<name>"` is no longer shadowed by mise's own root completers — but an arg named like one of them (`arg "<plugin>"`, `arg "<tool>"`) with **no** completer now falls back to file completion rather than mise's plugin/tool list.
- A top-level `complete` may now appear before or after its `arg`; the node name is lowercased and must match the arg name.
- **`usage_*` variables are NOT set during completion** — `${usage_service}` in a `complete run=` is empty. Read earlier words with `{{ words[PREV] }}` instead.
- When no completer is given, the arg's own name is used as the type: `arg "<file>"`, `<dir>`, `<path>`, `<command>`, `<executable>`, `<hostname>`, `<username>` complete with no block.

**Resolution order per arg:** `choices` (literal/`env=`/`run=`) → nested `complete` → top-level named `complete` → arg name as built-in type → file fallback. (Older versions let the `<file>` name win over `choices`; it no longer does.)

> 🔴 **One broken spec breaks completion for EVERY task.** If any task in scope has an invalid `usage` (or uses `clause`, or a TOML `usage` with an un-escaped `{{ }}`), `mise run <TAB>` falls back to file names for all tasks. Running the other tasks still works. Run `mise tasks validate` (which fails on usage parse errors since 2026.9.15) to find it — but note validate does **not** catch `clause` (it reports every task valid) nor an inline `choices` that is silently dropped.

**Example — values derived from the filesystem, the second depending on the first:**

```toml
[tasks.deploy]
usage = '''
arg "<service>" help="Service name" {
  complete run="ls -d infrastructure/*/application 2>/dev/null | sed 's|infrastructure/||;s|/application||'"
}
arg "<environment>" help="Target environment" {
  complete run="ls infrastructure/{% raw %}{{ words[PREV] | shell_quote }}{% endraw %}/application/env/ 2>/dev/null | sed 's/.tfvars//'"
}
'''
run = 'terraform -chdir=infrastructure/${usage_service?}/application apply -var-file=env/${usage_environment?}.tfvars'
```

**With descriptions:**
```
arg "<plug>" { complete run="printf 'alpha:First one\nbeta:Second one\n'" descriptions=#true }
```
Output format: `value:description` per line — split on the **first unescaped colon** (a line *starting* with `:` is not split); escape a literal colon with `\:`; both sides are trimmed.

**Tera variables and filters in `run`** (usage's own Tera, evaluated at completion time):
- `words` — array of all prompt words including the one being typed; access via `words[index]`. In a mise task this is the **whole line** — `["mise", "run", "<task>", …]`.
- `CURRENT` — index of word being typed
- `PREV` — index of previous word (`CURRENT-1`). **Only defined when `CURRENT > 0`.**
- Filters `shell_quote` (one value) and `shell_join` (array → space-separated, each quoted) (6.12) — quote words before splicing them into the command
- ⚠️ `slice` is **not** available in usage's completion Tera — a `{{ words | slice(…) }}` template fails and the arg silently falls back to file completion. Index with `words[…]` instead.

```
# file-task #USAGE header (not mise-rendered) — write the template directly
#USAGE complete "controller" run="ls modules/{{ words[PREV] | shell_quote }}/controllers"
# TOML usage string — wrap it so mise leaves it for usage
arg "<b>" { complete run="printf '%s\n' {% raw %}{{ words | shell_join }}{% endraw %}" }
```

Execution: `sh -c` (`cmd /c` on Windows when `sh` is absent), stdin closed, `__USAGE` set to the usage version. On Windows `cmd` cannot run pipelines or builtins — keep `run` to a single command.

**`choices` vs `complete`:**

| Feature | `choices` (literal / `env=` / `run=`) | `complete` |
|---------|-----------|------------|
| Values | Literal, env var, or command output | Command output, built-in type, or delegated |
| Validation | **Rejects** invalid input | Tab-completion only |
| `--help` | Lists literal values; shows ``output of `cmd` `` for `run=` | Not shown |
| Best for | Any closed set (static or derivable) | Open-ended values, paths, cascading lookups |

#### Tera Rendering of TOML `usage` Strings

mise renders a TOML task's `usage` string with **its own Tera context** (`config_root`, `cwd`, `env`, `vars`, `tools`, `mise_bin`, `xdg_*` …) **before** usage parses it. Consequences, verified on 2026.10.25:

- `{{ words[PREV] }}` / `{{ usage }}` meant for usage fail when the task is parsed — `[bad] ERROR Failed to render task usage … Cannot index into an undefined value` — breaking `mise run <task>`, its `--help`, **and completion for every other task** (other tasks still run; `mise tasks validate` catches it).
- Fix: wrap usage-time templates in `{% raw %}…{% endraw %}`.
- Upside: mise values work in defaults — `arg "[dir]" default="{{ config_root }}/sub"`, `flag "--home <h>" default="{{ env.HOME }}"`.
- File-task `#USAGE` lines are **not** rendered by mise; write `{{ words[PREV] }}` directly there.

### Accessing Arguments in Scripts

**Environment variables** (prefixed with `usage_`):

| Definition | Environment Variable |
|------------|---------------------|
| `arg "<file>"` | `$usage_file` |
| `arg "[output]"` | `$usage_output` |
| `flag "-v --verbose"` | `$usage_verbose` |
| `flag "--dry-run"` | `$usage_dry_run` |
| `flag "-o --output <file>"` | `$usage_output` |
| `cmd "migrate" { … }` (chosen subcommand) | `$usage_cmd` → `migrate`; nested → `db migrate` (6.12) |

**Naming rules:** `usage_` prefix + snake_case of the long name (hyphens → underscores, lowercased — `--Mixed-Case` → `usage_mixed_case`). An arg with `value_names "START" "END"` exports `$usage_start` (named after the **first** value name), not `$usage_<argname>`.

**`$usage_cmd` (usage 6.12, verified):** holds the subcommand path actually chosen, with aliases canonicalised (`dep` → `deploy`). When a spec has `cmd` children but none was chosen, mise sets it **to an empty string** (usage's own docs say "unset"). Branch on it instead of re-parsing words:

```bash
case "${usage_cmd:-}" in
  migrate) ./db migrate "${usage_dir?}" ;;
  seed)    ./db seed ;;
  *)       echo "pick a subcommand: migrate | seed" >&2; exit 1 ;;
esac
```

**Bash variable expansion patterns:**

| Pattern | Use Case |
|---------|----------|
| `${var?}` | Required args — fail if missing |
| `${var:-default}` | Optional with fallback |
| `${var:?msg}` | Required with custom error |
| `${var:+value}` | Conditional (if set, use value) |

**Values by type:**

| Type | Present | Absent |
|------|---------|--------|
| Boolean flag | `"true"` | **variable absent entirely** — but the string **`"false"`** (SET) with `default=#false`, after `--no-x` on a `negate` flag, or when an `env=`-backed var holds anything other than `true`/`1` (e.g. `VERB=yes` ⇒ `false`) |
| Count flag | `"1"`, `"2"`, etc. | **absent** |
| Value flag/arg | the string value (an explicit `--opt ''` is SET-empty) | absent unless `default=` or `env=` supplied one |
| `value_optional` flag | bare `--bump` → SET-empty; `--bump=major` → `major` | **absent** |
| Variadic | shell-escaped space-separated (`'y z' w`) | **absent** |
| `select`-driven output flag | the chosen output | **absent** — the `default=#true` output does not fill it |

**Critical distinction:** absent means **UNSET**, not empty string. `default=""` makes the variable SET to an empty string.

> 🔴 **Test boolean flags against `"true"`: `[ "${usage_x:-false}" = "true" ]`.** A `[ -n "${usage_x:-}" ]` test (or `${usage_x:+…}`) treats the string `"false"` as ON — wrong whenever the flag has `default=#false`, a `negate` form, or an env backing (verified on 2026.10.25; mise's own task-arguments doc now says the same). `-n` is only safe for value args/flags.

```bash
# Required arg — error if not provided
echo "Deploying to ${usage_environment?}"

# Optional with default
clean="${usage_clean:-false}"

# Conditional flag forwarding (boolean) — compare to "true", don't rely on :+
[ "${usage_verbose:-false}" = "true" ] && verbose_flag="--verbose"
docker build ${verbose_flag:-} .

# Boolean flag check
if [ "${usage_dry_run:-false}" = "true" ]; then
  echo "Dry run mode"
fi

# Count flag
level="${usage_verbose:-0}"

# Variadic to array
eval "files=($usage_files)"
```

**Precedence:** CLI args > env vars > defaults. This applies to **both TOML tasks and file tasks** (documented for file tasks since 2026.7.12). The usage docs describe a four-tier chain including config files, but the config tier is unimplemented (see [`config` block](#spec-level-metadata)).

> **Breaking change (2026.7.6):** `usage_*` variables are **invocation-local** — they no longer leak into nested tasks as implicit inputs. Values are cleared for normal task execution; `raw_args = true` tasks retain them. If a dependency needs a value, declare it with `env=` (under a different name) or pass it via structured `depends`.

Arguments are also readable as Tera values in TOML tasks: `{{ usage.env }}`, `{{ usage["dry-run"] }}`. Variadics arrive as arrays (usable with `for` / `length`). Values are **not** shell-escaped or quoted. Valid in `depends`, `depends_post`, and `wait_for` too.

### File Task Headers

```bash
#!/usr/bin/env bash
#MISE description="Deploy application"
#USAGE arg "<environment>" help="Environment" {
#USAGE   choices "dev" "prod"
#USAGE }
#USAGE flag "-f --force" help="Skip confirmation"

echo "Deploying to ${usage_environment?}"
```

**Supported shebangs:** `bash`, `node`, `python`, `deno`, `pwsh`, `fish`, `zsh`, `ruby`, `bun`.

`#MISE key=value` lines are task **config**, not spec — mise routes anything matching `^[a-z0-9_.-]+\s*=` away from the spec parser.

- **`.<script>.usage.kdl` sidecars** (usage 7.0.2) are **not** read by mise file tasks — args pass through raw. Use `#USAGE include file="…"` instead.
- An **undefined variable in a `#USAGE include` path** prints a WARN and the task runs with **no argument parsing** (args raw). `$$` writes a literal `$`; `MISE_TASK_FILE` is also available; task/project `env` cannot supply include variables.
- A file task with `#MISE extends="tpl"` gets the template's `usage` first, then its own `#USAGE` lines (2026.10.19).

### Subcommands (`cmd` block)

The `usage` spec supports nested subcommands via `cmd` blocks. Each subcommand can declare its own args/flags/completions:

```toml
[tasks.deploy]
usage = '''
flag "-v --verbose" global=#true help="Verbose output"

cmd "staging" help="Deploy to staging" {
  arg "<service>"
  flag "--force"
}

cmd "production" effect="destructive" help="Deploy to production" {
  arg "<service>"
  flag "--canary"
  before_help "Production deploys require approval."
}
'''
```

| `cmd` attribute | Type | Notes |
|-----------------|------|-------|
| `help`, `long_help`, `before_help`, `after_help`, `before_long_help`, `after_long_help` | string | Help text (valid as props *or* child nodes) |
| `help_md`, `before_help_md`, `after_help_md` | string | Also valid as child nodes since usage 3.6.0 |
| `subcommand_required` | boolean | Require a subcommand |
| `hide` | boolean | Hide from help |
| `effect` | enum | `read` \| `write` \| `destructive` |
| `restart_token` | string | Resets arg parsing for repeated invocations (this is how `mise run a ::: b` works) |
| `deprecated` | bool \| string | Mark deprecated |
| `alias "x"` | child node | Repeatable; supports `hide=#true` |
| `example "code"` | child node | **Exactly one** positional arg; use `header=` / `help=` / `lang=` props |
| `mount run="<cmd>"` | child node | Dynamic subcommand mounting; `run` is required. **Inside a `cmd` in a TOML `usage` it works at run time** (`usage_cmd="run inner"`, mounted args parsed); `synopsis="[TASK] [ARGS]…"` shows in that cmd's help |
| `disable_help_subcommand` | boolean | Every task with `cmd` children lists a fake `help` subcommand (`mise run t help` → `unexpected word: help`); set `#true` to remove it |
| `subcommand_help_heading`, `subcommand_value_name`, `next_line_help`, `term_width`/`max_term_width`, `display_order`, `help_heading`, `disable_version_flag`, `args_conflicts_with_subcommands` | various | All verified working on 2026.10.25. `display_order` orders whole help **sections** (2026.10.21, usage 7.1.3): each section sits at the lowest `display_order` of its commands, and a command without one counts as 999 (`args_conflicts_with_subcommands` → `subcommand 'sub' cannot be used with arguments on its parent command`). `disable_help_flag` hides the `-h` row but mise still prints help on `--help` |
| `subcommand_precedence_over_arg` | boolean | Works for a **multi-value** flag `flag "--items <i>..."` (`--items a b done` ⇒ `cmd=done`), **not** for a repeatable `var=#true` flag (`unexpected word: b`) or variadic positionals |
| `surface`, `available_if` | — | Metadata only — no effect in mise. (`view` and `about_md` are **top-level** nodes, not `cmd` attributes: `cmd "a" view="z"` is a hard error `unsupported cmd prop view`; at the top level they parse and do nothing) |

```
example "mycli list --all" header="Basic usage" help="Lists everything"
```

**Child-node-only** (never props): `flag`, `arg`, `mount`, `cmd`, `alias`, `example`, `complete`. Every prop also has a child-node form.

Mounted commands do **not** inherit the mounting CLI's global flags (usage 3.5.7+), so `mise run mytask --<TAB>` no longer offers mise's own `--cd`/`--jobs`/etc., and a task flag that collides with a global is no longer shadowed.

### Command Effects (`effect=`)

Declares what a command does to the world. Values, ordered: `read` < `write` < `destructive`.

| Effect | Meaning |
|--------|---------|
| `read` | Only inspects state. Idempotent. |
| `write` | Creates/modifies state; removes nothing the user can't recreate. |
| `destructive` | May delete or irreversibly overwrite. Deserves a confirmation prompt. |

- Available on `cmd` (usage 3.6.0+) and on `flag`/`arg` (usage 4.0+).
- The effect of an invocation is the **maximum** of the command's effect and every flag/arg actually supplied. Effects only ever **raise**, never lower.
- **Not inherited by subcommands.** Absent means *unknown*, not safe — consumers should treat it as "ask".
- **Most flags should declare nothing** — reserve it for the few that change what a command does.
- `--dry-run` lowering is deliberately unsupported.

```
cmd "settings" effect="read" {
  arg "[setting]"
  arg "[value]" effect="write"   # `settings foo` reads, `settings foo=bar` writes
}
```

Consumed by `mise mcp`'s `list_commands` tool and `usage mcp` (4.1.0+), which tag every command with its declared effect.

### Spec-Level Metadata

Top-level usage keywords (rarely needed for mise tasks but supported):

```
name "..."              # display name
bin "..."               # binary name (defaults to filename)
version "..."           # version string — ⚠️ BROKEN in mise tasks: -V/--version prints `mise ERROR 1.0` and exits 1
min_usage_version "..." # minimum usage spec version required (put first; warns, does not fail)
author "..."            # CLI author
license "..."           # SPDX license identifier
repository "..."        # plain project URL (usage 4.1.0+)
about "..."             # short help
long_about "..."        # long help
before_help "..."       # text before help body
after_help "..."        # text after help body
before_long_help "..."  # before-help shown with --help
after_long_help "..."   # after-help shown with --help
usage "..."             # override the generated usage line
disable_help #true      # suppress the automatic help flag
default_subcommand "..." # target for an unmatched word (`task src` → `task run src`)
default_subcommand_on_empty #true  # 6.11: a BARE invocation also runs the default subcommand
logo #"""…ascii art…"""# style="cyan+bold"   # 6.11: shown on the root -h/--help page only
source_code_link_template "..."  # Tera template with {{path}} / {{cmd}}
example "code" header="..." help="..." lang="..."
include file="./other.usage.kdl"   # merge another spec (file= is required)
                                   # In FILE TASKS (2026.9.12+) the path may be relative to the
                                   # task file, or use $MISE_CONFIG_ROOT / $MISE_TASK_DIR /
                                   # $MISE_PROJECT_ROOT — so shared flagsets need no absolute path.
                                   # In a TOML `usage` string a relative path FAILS
                                   # ("relative includes require a source file") — write
                                   # include file="{{ config_root }}/shared.usage.kdl"
```

`repository` (a plain URL, like `Cargo.toml`'s) is distinct from `source_code_link_template` (a per-command deep link) — neither implies the other. `bin "x"` does **not** rename the usage line in mise — help always uses the task name.

**`default_subcommand` vs `default_subcommand_on_empty` (verified):** with only `default_subcommand "run"`, a bare `mise run t` runs the *root* (`$usage_cmd` empty) while `mise run t src` routes to `run`. Adding `default_subcommand_on_empty #true` makes the bare call run `run` too (its arg defaults apply). `--help` marks the default `(default)`.

**`config` block** — declares a CLI's settings for docs, JSON-schema, and SDK generation:

```
config {
  file "~/.config/mycli/config.toml" scope="global"
  file "mycli.toml" findup=#true
  prop "jobs" type="uint" default=0 help="Number of parallel jobs" {
    cli "--jobs" "-j"
    env "MYCLI_JOBS"
  }
}
```

`prop` keys are dotted paths (`prop "status.missing_tools"`; props do not nest) with a `type=` (scalars plus list/set/map/option/union forms), `default`, `help`, and child `cli` / `env` / `deprecated_env` / `source` bindings. `source "git" name="git config"` declares an external kind for docs. The older `data_type=` / `default_note` spellings still parse.

> **mise resolves nothing from a `config` block.** Verified: with `prop "jobs" … { env "MYCLI_JOBS" }`, `MYCLI_JOBS=5 mise run t` leaves the value unset. It is metadata only — the "CLI > env > config file > default" chain does not apply to tasks. Use `env=` on the `arg`/`flag` itself for env backing.

### Validation and Env-Backed Choices

`validate=` and `choices env=` work and are enforced on mise 2026.8.13 and later — mise depends on `usage-cli`, whose manifest turns on usage-lib's `validation` and `unstable_choices_env` features. Verified on 2026.10.25:

| Syntax | Valid value | Invalid value |
|--------|-------------|---------------|
| `flag "--port <p>" { arg "<p>" validate="int(value) >= 1 && int(value) <= 65535" validate_error="port must be 1-65535" }` | `--port 8080` → runs | `--port 99999` → `Invalid value for port: 99999: port must be 1-65535`; `--port abc` → `validation expression failed: invalid operation: int(abc)` |
| `arg "<name>" validate="value matches '^[a-z]+$'" validate_error="lowercase only"` | `abc` → runs | `ABC` → `Invalid value for name: ABC: lowercase only` |
| `arg "<env>" { choices env="DEPLOY_ENVS" }` | in the list → runs | `Invalid choice for arg env: qa, expected one of dev, prod` |

A `validate` rule runs **after defaults and env fallbacks are applied** — it is skipped only when no CLI, env, or default value exists. So `arg "[n]" default="0" validate="int(value) >= 1"` fails a bare invocation (`Invalid value for n: 0: …`); keep defaults inside the valid range. Prefer `validate=` / `choices` over hand-rolled checks in the script body — mise rejects bad input before any dependency runs.

### Still Not Implemented (do not use)

These appear in usage docs but hard-error (or silently do nothing) in mise 2026.10.25. Errors now carry a precise span, e.g. `unsupported arg key parse`:

| Syntax | Result in mise | Use instead |
|--------|----------------|-------------|
| `arg "<f>" parse="cmd {}"` | `unsupported arg key parse` | — (no equivalent) |
| `flag "--color" config="ui.color"` | `unsupported flag key config` | `env="..."` |
| `config_alias "a" "b"` | `unsupported spec key config_alias` | — |
| `cmd "x" { example "Header" "code" }` | `expected 1..=1 arguments, got 2` | `example "code" header="Header"` |
| a spec with only `mount` (no `arg`, `flag`, or `cmd`) — TOML `usage` **or** file-task `#USAGE` | mise runs the usage parser only when a spec has at least one `arg`, `flag`, or `cmd`, so a mount-only spec is ignored at run time: the task reports no arguments and extra args pass through raw; only completion runs the mount | Add any `arg`/`flag` beside the root-level `mount` (it then works in TOML and file tasks, though `--help` lists only your own args), or put the mount inside a `cmd` block |
| `clause "x" separator=":::"` | values never reach the script (the words are appended raw to the command); `:::` collides with mise's own task separator; **breaks completion for every task**, and `mise tasks validate` doesn't catch it (a body-less `clause` is a parse error it does catch) | Separate tasks, or `arg "<args>..."` |
| `external_subcommand #true` | runs, but nothing is exported | `raw_args = true` |
| `multicall #true` | `unexpected word` | — |
| `dont_delimit_trailing_values #true` | ignored | — |
| `version "…"` / `long_version "…"` | `-V`/`--version` print the version as `mise ERROR 1.0` and exit 1. A spec with only `version` and no args is treated as having no arguments (args pass raw) | Print the version from the script (e.g. a `--version` boolean flag) |
| `flag … action="help"` / `"help_short"` / `"help_long"` / `"help_all"` | prints help through `mise ERROR failed to validate task`, exit 1 (also with `builtin=#true`) | Rely on the built-in `-h`/`--help` |
| `inline choices` — `arg "<x>" choices "a" "b"` | **parses, validates nothing** | `arg "<x>" { choices "a" "b" }` |

> **Working v6 policies** (verified): `unknown_flags "error"`, `subcommand_negates_reqs #true`, `allow_missing_positional #true`, `args_override_self #false`, `arg_required_else_help #true` (bare call prints usage and exits 1), `default_subcommand_flags`/`default_subcommand_help`, `help_template` (`{% raw %}`-wrapped in TOML), `heading` + `help_heading`, `flatten_help`, and `deprecated` / `deprecated_warn_at` / `deprecated_remove_at` (rendered in `--help` only — mise prints **no runtime warning**). Rich `choice "always" { alias "yes" }` entries and `choices ignore_case=#true` validate, but the value is **not normalised** (`ALWAYS`/`yes` reach the script as typed).

**Sigil args (usage 6.5, verified)** — a variadic positional whose items start with a sigil, ahead of the normal args:

```
arg "[tool]..." sigil="+" { choices "node@22" "node@24" "python@3.14" }
arg "<command>"
arg "[args]..."
```
`mise run t +node@22 +python@3.14 python x y` → `usage_tool="node@22 python@3.14"`, `usage_command=python`, `usage_args="x y"`; `+n<TAB>` completes with the sigil restored.

### Flag Groups, Flagsets, and Outputs (v6)

**`group`** — declare how a set of flags relate. Fully enforced in mise:

```toml
[tasks.fetch]
usage = '''
flag "--file <f>"
flag "--url <u>"
flag "--stdin"
group "input" "--file" "--url" "--stdin" required=#true
'''
```

| `required` | `multiple` | Meaning |
|-----------|-----------|---------|
| — | — | At most one |
| `#true` | — | Exactly one |
| — | `#true` | Nothing enforced |
| `#true` | `#true` | At least one |

Errors read `Missing one of the required flags in group input: --file, --url` and `Invalid flag '--url': cannot be used with --file in group input`. Members may also be **positional names** (`group "input" "--file" "--url" "pos" required=#true` — enforced). A block form exists too: `group "input" required=#true { flag "--file" "--url"; flag "--stdin" }`.

**`flagset` + `use`** — a named set of flags any `cmd` can pull in. Declared at spec top level only; resolved while the spec is read, so nothing downstream sees a new concept:

```toml
usage = '''
flagset "output" {
  flag "-v --verbose"
  flag "--json"
}
cmd "build" { use "output" }
cmd "test"  { use "output" }
'''
```

**`output` / `exit_code` / `select`** — declare what a command writes and what its statuses mean. `output`/`exit_code` are metadata for docs and MCP consumers, **but `select` is enforced**: it turns the named flag into a choice over the declared outputs (`--format bogus` → `Invalid choice for option format: bogus, expected one of human, json`), and `--help` lists them as possible values. A boolean flag can select one output with `output "json" select="--json"`. The selected flag must be declared in that `cmd` or above — `select` naming an undeclared flag is a hard spec error (`… names no flag here or above it`) that also breaks completion for every task.

```
cmd "check" {
  flag "--format <f>"          // select needs this flag declared here or above
  output "human" default=#true help="Human-readable report"
  output "json" media_type="application/json" framing="json"
  select "--format"
  exit_code 0 "all checks passed"
  exit_code 1 "a check failed"
}
```

`framing` is `text` (default), `json` (one document), or `jsonl` (one per line).

> ⚠️ **Multi-placeholder fixed arity does not map to separate env vars in mise.** `arg "<start> <end>" var_min=2 var_max=2` parses, but both values land in `$usage_start` (`"1 2"`) and `$usage_end` is empty. Use two separate `arg` nodes when you need two variables.
