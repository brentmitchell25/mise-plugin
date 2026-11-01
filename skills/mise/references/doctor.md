## Project Diagnostics (`[doctor]`)

**Contents**

- [Project Diagnostics ([doctor])](#project-diagnostics-doctor)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

`mise doctor project` runs named checks from the project's configuration — for requirements tool versions alone cannot establish, like whether a compiler can find a native library or a database accepts connections.

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

# 2026.10.18: ordering — a check whose dependency didn't pass reports `blocked`
[doctor.checks."examplectl:auth"]
run = "examplectl auth status"

[doctor.checks."examplectl:project"]
depends = ["examplectl:auth"]
run = 'test -n "$(examplectl config get project)"'
hint = "Run `examplectl config set project {{ vars.project }}`"   # 2026.10.17: run/description/hint are Tera
```

```bash
mise doctor project
mise doctor project --json
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `run` | string | *required* | Diagnostic command. **Exit zero passes**; any other code fails. Tera-rendered (2026.10.17). |
| `description` | string | the check name | Human-readable requirement. Tera-rendered. |
| `hint` | string | — | Remediation shown on failure. **Never executed.** Tera-rendered. |
| `depends` | string \| string[] | — | **2026.10.18.** Checks that must pass first; otherwise this one reports `blocked`. Unknown names or cycles are errors. |
| `timeout` | duration | `10s` | Positive duration (`500ms`, `5s`) |
| `dir` | string | the declaring config's root | Relative paths resolve from that root; `~/` and absolute used as given |
| `shell` | string | mise's inline task shell | Includes the command flag, e.g. `"bash -c"`, `"pwsh -Command"` |
| `os` | string \| string[] | run everywhere | Same selectors/aliases as tool filters (`darwin`, `win`, `amd64`, `linux/arm64`). **`os = []` is rejected.** |

- Ordinary `mise doctor` diagnoses mise itself and does **not** run these. Checks never run on directory entry or before a task — there are no implicit checks.
- Checks run **concurrently**, bounded by `jobs` (`MISE_JOBS`); set `jobs = 1` for sequential. Results stay in name order and a failure doesn't stop the others.
- A higher-precedence definition **replaces the entire check** of the same name, including its hint and dir.
- Commands get the project's environment and tool paths. Diagnostics do **not** install tools or run task dependencies/hooks. Checks are **not sandboxed**.
- Each command has a deadline and a combined **64 KiB** stdout/stderr cap; output is captured and **discarded** so a probe can't print a credential into the report. Use `description`/`hint` to explain; run the command directly for detail.
- Templates are **not rendered** in `dir`. Checks that apply to the current platform report errors in safe mode. Empty or fully platform-excluded check lists still succeed.
- Unknown `[doctor]` container options **warn** (forward compatibility); **unknown check fields are rejected** to catch typos.
- Statuses: `pass`, `fail`, `error`, `blocked`, `skipped`. Safe mode refuses `[doctor.checks]`.
- JSON report has `checks` and `errors` arrays; config-loading errors populate top-level `errors` with `checks` empty.

> Plain `mise doctor` gained checks too: it warns when an installed plugin differs from its `[plugins]` URL/ref (2026.9.15) and when a dotfiles-history watcher is stale or outdated (2026.9.18 / 2026.10.0), with recovery advice.
