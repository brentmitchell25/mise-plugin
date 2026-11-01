## Task Tracing with OpenTelemetry (Experimental)

**Contents**

- [Task Tracing with OpenTelemetry (Experimental)](#task-tracing-with-opentelemetry-experimental)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

Added **2026.9.13** (docs: `https://mise.jdx.dev/tasks/opentelemetry.html`). `mise run` can export one trace per run — a root span, setup spans (fetch remote tasks, resolve tasks, install tools, deps, start daemons), monorepo group spans, and a span per task — plus, optionally, every task output line as an OTLP log record.

```toml
[settings]
experimental = true   # required for export since 2026.10.5
otel.enabled = true   # traces
otel.logs = true      # optional: task stdout (INFO) / stderr (WARN) as log records
```
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
mise run build ::: test
```

| Setting | Env | Default | Exports only when… |
|---------|-----|---------|--------------------|
| `otel.enabled` | `MISE_OTEL_ENABLED` | `false` | `OTEL_EXPORTER_OTLP_ENDPOINT` or `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` is also set |
| `otel.logs` | `MISE_OTEL_LOGS` | `false` | `OTEL_EXPORTER_OTLP_ENDPOINT` or `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` is also set |

The setting gate exists so mise doesn't emit spans just because other tools set `OTEL_EXPORTER_OTLP_*`. 🔴 **Since 2026.10.5 export also requires `experimental = true`** — `mise run` errors with `OpenTelemetry export is experimental` when it would export (enabling `otel.*` with no endpoint is harmless).

- **Standard env honored:** `OTEL_EXPORTER_OTLP_{ENDPOINT,TRACES_ENDPOINT,LOGS_ENDPOINT,HEADERS,TRACES_HEADERS,LOGS_HEADERS,TIMEOUT,…}`, `OTEL_EXPORTER_OTLP_PROTOCOL` / `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` / `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` (`http/protobuf` default, or `http/json` — **no gRPC**), `OTEL_SERVICE_NAME` (default `mise`), `OTEL_RESOURCE_ATTRIBUTES`.
- **Task span attributes:** `mise.task.name`, `mise.task.display_name`, `mise.task.args`, `mise.task.source`, `mise.task.config_root`, `mise.task.skipped`, `mise.task.cancelled`, `process.command_args`, `process.exit.code`. Non-task spans carry `mise.span_type` (`run` \| `setup` \| `monorepo_group`; group spans add `mise.config_root`); setup spans include `fetch tasks`. Log records carry `output.stream`. Only the task that actually failed is marked `Error`; siblings mise stopped get `mise.task.cancelled = true`.
- **Propagation:** `TRACEPARENT` / `TRACESTATE` are passed to tasks, so nested `mise run` calls and OTel-instrumented tools join the same trace. Nested runs coordinate log export via `MISE_TASK_OTEL_LOG_CLAIM` (each line exported once, by the innermost run).
- **Redactions** apply to args in span names/attributes, error messages, and exported log lines.
- **Failure handling:** 3s export timeout; failures are logged at debug and never fail the run. Offline mode disables export. On `--timeout` expiry the root span ends as `Error`; still-running tasks aren't exported.

> ⚠️ **`otel.logs` changes how tasks see their terminal.** In `interleave`/`quiet` output modes the child's stdio becomes a **pipe, not a TTY** — affecting `isatty()`, colours, progress bars, and prompts. Use `--raw` to keep a TTY (that output is then not exported). It is also a separate trust boundary: anything a task prints, including secrets that escape redaction, leaves the machine.
