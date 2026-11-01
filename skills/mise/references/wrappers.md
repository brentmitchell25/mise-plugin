## Command Wrappers (`[wrappers]`)

**Contents**

- [Command Wrappers ([wrappers])](#command-wrappers-wrappers)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

Added **2026.8.16**. Intercept an ordinary command name with a different command, arguments, and environment. Wrappers take precedence over mise-managed tools, and mise **strips its own dispatch directories before delegating**, so the underlying tool still resolves from mise or the system.

```toml
[wrappers]
# string shorthand
rg = "rg-wrapper"

# table form
[wrappers.cargo]
command = "mbx"
args = ["--shim"]                        # inserted BEFORE the intercepted command's arguments
env = { MBX_CARGO_SHIM_MODE = "1" }
```

| Field | Type | Description |
|-------|------|-------------|
| `command` | string | *Required* in table form. The command to run instead. |
| `args` | string[] | Arguments inserted **before** the intercepted command's own arguments |
| `env` | table | Environment variables set for the wrapper |

Works with normal activation, `mise activate --shims`, and `mise exec`, and (since 2026.9.15) through Windows exe/file shims. Managed wrapper shims are refreshed by `mise reshim`. Since **2026.9.13** a wrapper **installs a missing provider tool before dispatching** (lazy providers always; others when `not_found_auto_install` is on, the default). Shim and wrapper dirs are removed from PATH while the wrapper runs. A remote [`include`](settings.md#remote-config-includes-include) fragment may carry `[wrappers]`; the including file's entry wins. Safe mode ignores project `[wrappers]`. Docs: `https://mise.jdx.dev/dev-tools/shims.html#command-wrappers`.

> 🔴 **`command`, `args`, and `env` are Tera templates (2026.10.13, breaking).** A literal `{{` must be wrapped in `{% raw %}…{% endraw %}`. `command` sees only `vars`, `config_root`, `config_source` (2026.10.14), and the process env.

> Related: `activate_shims = false` (`MISE_ACTIVATE_SHIMS`, default `true`, added 2026.9.2) keeps tool shim directories off PATH during activation and hooks without changing auto-install or lazy-tool settings — which lets external command wrappers keep working.
