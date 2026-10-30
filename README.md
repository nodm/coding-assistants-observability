# AI Coding Assistant Observability Stack

A self-hosted OpenTelemetry pipeline + Grafana dashboards for monitoring your own usage of **Claude Code**, **GitHub Copilot CLI**, and **GitHub Copilot Chat (VS Code)** — cost, token usage, tool calls, latency, errors, and more.

Everything runs locally via containers (Podman or Docker). No data leaves your machine.

## Architecture

```
Claude Code ──┐
Copilot CLI ──┼──OTLP──▶ otel-collector ──┬──▶ Prometheus (metrics)
Copilot Chat ─┘        (:4317 grpc,       ├──▶ Loki       (logs)
                        :4318 http)       └──▶ Tempo      (traces)
                                                   │
                                                   ▼
                                               Grafana (:3000)
                                          (dashboards, pre-provisioned)
```

- **otel-collector** — the only ingestion point. Receives OTLP over gRPC (4317) and HTTP (4318) from all three tools, fans out to the three backends.
- **Prometheus** (`:9090`) — scrapes the collector's `/metrics` endpoint (exposed internally on :8889) every 15s. Stores all numeric metrics (cost, tokens, durations, counts).
- **Loki** (`:3100`) — receives logs/events natively over OTLP (`/otlp` endpoint). Only Claude Code and Copilot Chat emit this signal; **Copilot CLI does not** (see Known Limitations).
- **Tempo** (`:3200`) — receives traces natively over OTLP. All three tools emit traces; Claude Code's are beta and require `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` plus `OTEL_TRACES_EXPORTER=otlp`.
- **Grafana** (`:3000`) — three dashboards, auto-provisioned from the JSON files in this repo. Anonymous access enabled (Editor role) since everything is bound to `127.0.0.1` only.

All ports are bound to `127.0.0.1` — nothing is reachable from outside your machine.

## Dashboards

| Dashboard | Covers |
|---|---|
| **Claude Code Usage** | Cost, sessions, commits, PRs, token usage, lines of code, tool-edit decisions (accept/reject, by source/language), active time, MCP connection health, API errors, hook/plugin activity, live event log, recent traces |
| **Claude Code - Session Analysis** | Per-session drill-down: a leaderboard table of all sessions (project, cost/tokens/active time/LOC/commits/PRs) to pick a session from, then a `$session_id`-scoped view — working directory/repo identity, token/cost/active-time breakdowns, tool-decision mix, tool latency (p95) and call volume by tool, that session's traces, and its raw event log |
| **GitHub Copilot Usage** | Chat calls, agent invocations, tool calls, token usage, latency (avg + p50/p95), lines of code (CLI only), tool outcomes, MCP connections, CLI traces table, Chat event log |
| **AI Assistant Overview** | Side-by-side comparison across all three tools: cost, input/output/cache/total tokens, sessions/invocations |

## Repo contents

```
compose.yaml              # the whole stack
otel-collector.yaml       # collector config (receivers/exporters/pipelines)
prometheus.yml            # scrape config
tempo.yaml                # Tempo single-binary config
grafana/provisioning/
  datasources/datasources.yaml   # Prometheus, Loki, Tempo datasources
  dashboards/dashboards.yaml     # tells Grafana where to find dashboard JSON
  dashboards/*.json              # the 4 dashboards themselves
```

Telemetry **data** (Prometheus/Loki/Tempo/Grafana state) lives in named Podman/Docker volumes, **outside** this repo — nothing sensitive (no captured prompts, emails, tokens) is ever written into these files.

---

## Setting this up on a new machine

### 1. Prerequisites

- Podman (with `podman-compose`) or Docker (with `docker compose`) — the `compose.yaml` works with either; just substitute `docker compose` for `podman-compose` below if using Docker.
- Claude Code CLI, GitHub Copilot CLI, and/or VS Code with the GitHub Copilot Chat extension — whichever of the three you actually use.

### 2. Clone and start the stack

```bash
git clone <your-repo-url> coding-assistants-observability
cd coding-assistants-observability
podman-compose up -d        # or: docker compose up -d
```

> **Note:** `compose.yaml` pins an explicit `name: claude-graphana` (a leftover from this project's original folder name). This is deliberate, not a bug — Podman/Docker Compose derives container and volume names from this project name, so changing it would create brand-new, empty volumes and orphan all historical data. Leave it as-is unless you're deliberately starting fresh.

First run pulls 5 images (~1–2 GB total). Verify everything is healthy:

```bash
podman ps --format "{{.Names}}\t{{.Status}}"
```

You should see `claude-otel-local`, `claude-prometheus`, `claude-loki`, `claude-tempo`, `claude-grafana` all `Up`.

Open **http://localhost:3000** — dashboards should already be there (no login needed).

### 3. Point each tool at the collector

None of this config lives in this repo — it's global per-tool config on each machine, so you need to set it up again on every new computer.

#### Claude Code

Add to `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "CLAUDE_CODE_ENHANCED_TELEMETRY_BETA": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_TRACES_EXPORTER": "otlp",
    "OTEL_METRICS_INCLUDE_REPOSITORY": "true",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317"
  }
}
```

`CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` + `OTEL_TRACES_EXPORTER` turn on traces (beta) — per-interaction spans (`claude_code.interaction` → `llm_request`/`tool`/`hook`) with latency, token counts, and model per step, visible in the "Recent Claude Code traces" panel on the Claude Code Usage dashboard.

`OTEL_METRICS_INCLUDE_REPOSITORY` tags every metric (and event) with `vcs.repository.name`/`vcs.owner.name` from the session's git remote, so a `session.id` isn't the only way to identify a session — the Session Analysis dashboard's leaderboard shows the project name next to it, and raw session IDs stay purely as a stable row key for filtering.

Takes effect on the next new session (not the one you're editing it from).

#### GitHub Copilot CLI

Just one env var, set it anywhere that reaches the `copilot` process (e.g. `~/.bashrc` or `~/.zshrc`):

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

That's the HTTP port (4318), not the gRPC one Claude Code uses — the CLI only speaks OTLP/HTTP.

#### GitHub Copilot Chat (VS Code)

Add to VS Code's settings.json:

```json
{
  "github.copilot.chat.otel.enabled": true,
  "github.copilot.chat.otel.exporterType": "otlp-http",
  "github.copilot.chat.otel.otlpEndpoint": "http://localhost:4318"
}
```

**Which settings.json depends on how VS Code runs:**
- **Plain local VS Code**: your normal user `settings.json` (`Ctrl+Shift+P` → *Preferences: Open User Settings (JSON)*).
- **Remote-WSL / Remote-SSH / Dev Containers**: the **Machine**-scoped settings for that remote (e.g. `~/.vscode-server/data/Machine/settings.json` inside the WSL distro) — your *local* Windows settings.json won't apply to the remote extension host.

Requires a window reload (`Developer: Reload Window`) after changing.

### 4. Verify end-to-end

Run each tool once, then check:

```bash
# metrics reached Prometheus?
curl -s 'http://localhost:9090/api/v1/query?query=up' | grep '"value"'

# logs reached Loki?
curl -sG 'http://localhost:3100/loki/api/v1/label/service_name/values'
# expect: ["claude-code", "copilot-chat"]  (copilot CLI never appears here — see below)
```

---

## Known limitations (found the hard way — worth knowing before you debug these again)

- **GitHub Copilot CLI emits no logs/events signal.** Only traces + metrics. Confirmed via `copilot help monitoring`. The "Recent Copilot CLI traces" panel on the Copilot dashboard is the closest substitute.
- **Fixed: the "Chat calls", "Avg chat latency by model", "Chat calls by source", and "Chat latency percentiles" panels were only matching `gen_ai_client_operation_duration`, which per `copilot help monitoring` is the CLI's *non-inference* operation metric.** The actual LLM call duration for Copilot CLI is `gen_ai_client_inference_duration` — a separate histogram. These panels now match `gen_ai_client_(operation|inference)_duration(_seconds)?` so they cover both Copilot Chat (which uses `operation.duration` per its own semconv) and CLI (which uses `inference.duration`). Before this fix, CLI chat latency/volume may have been silently undercounted or missing from these four panels.
- **Short-lived processes "disappear" from Prometheus after ~5 minutes of inactivity.** Copilot CLI runs are brief; the collector's Prometheus exporter stops re-exposing a metric ~5 min after the last update (`metric_expiration`), and Prometheus then can't extrapolate `increase()`/`rate()` for it in an instant query. This is normal, not a bug — the historical data is still there if your time range covers it.
- **Prometheus relabels the OTel "job" resource attribute to `exported_job`.** Its own scrape config already claims the `job` label, so every PromQL query that wants to distinguish tools must filter/group on `exported_job`, not `job`.
- **Use `$__rate_interval`, not `$__interval`, inside `rate()`/`increase()` in Prometheus queries.** `$__interval` is just the display resolution; when it gets close to the 15s scrape interval, Prometheus often can't find 2 samples and silently returns nothing. `$__rate_interval` is purpose-built to avoid this.
- **The Loki datasource does not support `$__rate_interval` at all** — it errors with "not a valid duration string." Use `$__interval` for Loki's `count_over_time()` queries (no staleness problem there since it counts raw log lines, not polled samples).
- **The "Recent Copilot CLI traces" table panel (Tempo, TraceQL) was never fully confirmed to render.** I hit an unresolved "unsupported query type" error testing it via Grafana's API directly; the underlying Tempo data is solid (verified via Tempo's native API), but if that specific panel shows an error in your browser, it needs further debugging.
- **Claude Code's "sessions" and the other tools' "invocations" aren't quite the same concept** — the Overview dashboard's session/invocation comparison is directional, not a precise apples-to-apples count.
- **Token totals differ in what they include per tool** unless you specifically use the Overview dashboard's Input/Output/Cache/Total breakdown — a naive "total tokens" sum can silently include or exclude cache tokens depending on the source.
- **The "Claude Code - Session Analysis" dashboard was built without a live stack to test against** (no containers were running at the time). It assumes: the collector's Prometheus exporter sanitizes the `session.id` resource attribute to a `session_id` label (standard OTel Collector behavior, same pattern as the existing `exported_job` rename); Loki's OTLP receiver exposes `session_id` and `duration_ms` as structured metadata queryable via `| session_id="..."` and `| unwrap duration_ms` (same style as the existing `event_name` filters); and Tempo's TraceQL accepts the unscoped `.session.id` attribute selector. Verify the leaderboard table's `merge` transform renders correctly and the Loki `unwrap` latency panel returns data before relying on either. The leaderboard's "Project" column and the "Session identity" panel carry the same assumption for `vcs.repository.name` → `vcs_repository_name` (metrics) and `workspace.host_paths`/`vcs_repository_name` as Loki structured-metadata fields reachable via `| keep`.
- **Claude Code's content-logging flags (`OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_ASSISTANT_RESPONSES`, `OTEL_LOG_TOOL_DETAILS`, `OTEL_LOG_TOOL_CONTENT`, `OTEL_LOG_RAW_API_BODIES`) are deliberately left unset.** They'd put real prompt/response text, shell commands, and file contents into Loki/Tempo — strictly more useful for debugging, but a step up in what's stored even though it never leaves this machine. Set them in `~/.claude/settings.json` if you want that detail; the "Recent Claude Code traces" panel will still show spans without them, just without content attributes.

## Security notes

- Grafana has **anonymous access enabled with Editor role** (`GF_AUTH_ANONYMOUS_ENABLED=true`) and the **default `admin`/`admin` credentials were never changed**. This is fine *only* because every port in `compose.yaml` is bound to `127.0.0.1` — nothing is reachable off-box. **If you ever change a port binding to `0.0.0.0` or expose this to a network, disable anonymous auth and set `GF_SECURITY_ADMIN_PASSWORD` first.**
- Telemetry data includes your email, account/org IDs, session IDs, and (depending on config) tool arguments — it's all PII-adjacent. It stays in local Podman/Docker volumes and is never part of this git repo, but don't casually `curl` these endpoints onto a shared machine.

## Data persistence

Metrics/logs/traces/dashboards survive `podman-compose down`, container removal, and full laptop reboots — they live in named volumes (`prometheus-data`, `loki-data`, `tempo-data`, `grafana-data`) on disk, independent of container lifecycle. They are **only** destroyed by `podman-compose down -v` or an explicit `podman volume rm`. Containers themselves do **not** auto-start after a reboot — run `podman-compose up -d` again each time (or set up a systemd/Quadlet unit if you want that automated).

## Maintenance

```bash
podman-compose down           # stop everything, keep data
podman-compose up -d          # start it back up, same data
podman-compose down -v        # ⚠️ stop AND wipe all historical data
podman logs -f claude-otel-local   # debug the collector
podman logs -f claude-grafana      # debug Grafana
```

Image versions are pinned in `compose.yaml` (otelcol 0.160.0, Prometheus v3.0.1, Loki 3.3.2, Tempo 2.6.1, Grafana 11.4.0) for reproducibility — bump them deliberately, not via `:latest`.
