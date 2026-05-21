# Whale Analysis Playbook

## Purpose

This is the shared read-only playbook for the `whale-analysis` skill.
Hermes adapters should derive from this file and from the canonical gateway
contract surfaced through `/cli/v1/debug/schema`, `/cli/v1/debug/explain`,
`/cli/v1/snapshot`, and `/cli/v1/correlate`.

Metadata host:

- [`whale-analysis-playbook.json`](./whale-analysis-playbook.json)

## Ownership Boundary

The skill is not a second resource kernel.

- Gateway or resource kernel owns resource semantics, API composition, evidence
  bundles, permission behavior, pagination, query semantics, and compatibility.
- `whale` CLI owns command grammar, auth and config, help and version, install
  surfaces, and local rendering such as `--format summary`.
- `whale-analysis` owns natural-language intent routing, first-pass evidence
  choice, stop or deepen decisions, and human-readable conclusions.

When a rule is deterministic and reusable, prefer moving it into gateway or
CLI. Short prompt-level fallbacks are allowed only as transitional behavior.

## Non-Goals

- No write operations such as release application, ticket creation, approval,
  restart, scaling, retry, or mutation workflow by default.
- No split into multiple analysis skills in this phase.
- No duplicated resource schema, permission, pagination, SLS, or VictoriaLogs
  rules in adapter prompts.
- No requirement that the first response confirms root cause.

## Analysis Model

### Phase 1: First Update

Default to a useful first update, not an exhaustive investigation.

Target:

- usually within 60 seconds
- answer whether this currently looks healthy, active, recovered,
  stale/ignored, suspected, or inconclusive
- include one handling suggestion and only material evidence gaps

Preferred first reads:

- application health:
  `snapshot application --name <app> --env <env> --profile triage --format summary`
- alert analysis: `snapshot alert <id> --format summary`
- current-alert triage: `snapshot alert current --format summary`
- release context: `snapshot release <id> --format summary`
- task context: `snapshot task <id> --format summary`
- time-window relationship:
  `correlate application --name <app> --env <env> --window <duration> --format summary`

Status-only requests are simpler:

- For `看状态`, `查状态`, or equivalent lookup, run
  `status application --name <app> --env <env>` first.
- If no environment is given, use `production-b2b-cn` and say so briefly.
- Stop when alerts are 0, restart is 0, latency is normal, and the only
  warning is QPS or traffic.

Do not fetch appinstance, pod, logs, SLS, VictoriaLogs, trace, topology,
approval, runtime/template, or broad debug output in phase 1 unless the user
explicitly asks for that evidence.

### Phase 2: Deepen When Needed

Automatically deepen only when one of these is true:

- evidence bundle quality is insufficient
- an anomaly is clear and the allowed first-pass evidence lacks enough owner,
  resource, time, or severity context to give a useful handling suggestion
- evidence classes conflict
- the user explicitly asks to continue, confirm root cause, inspect logs,
  inspect trace, inspect instances, or inspect release impact

Deepen with the smallest evidence class that can change the recommendation.
Do not continue collecting evidence just to make the answer look complete.
For `当前告警`, do not treat "there is an obvious next read" as a reason to
deepen before the first answer; put it in `下一步` unless the user asks to
continue.

### Controlled Action Boundary

Analysis remains read-mostly by default.

- Do not call raw K8S, CMDB restart/scale/stop, rollback, release, approval,
  or ticket write APIs.
- The only controlled write surface in this phase is `whale workload restart`.
- For restart requests, first run
  `whale workload restart ... --reason <reason>` without `--yes`; this records
  an audit event and returns CMDB policy.
- Execute with `--yes` only when policy is `direct_allowed` and the user
  explicitly confirms the execution.
- If policy is `approval_required`, recommend a modify-appinstance ticket.

## Output Contract

Default first-pass output:

```text
结论: <healthy | active | recovered | stale_or_ignored | suspected | inconclusive>
置信度: <high | medium | low>
处理建议: <one actionable recommendation>
证据:
- <material evidence>
未确认:
- <only material gaps>
下一步:
- <one best read-only next step>
```

Status-only output:

```text
结论: <one sentence>
证据:
- <up to 4 short evidence bullets>
下一步: <optional>
```

Rules:

- Separate evidence from inference.
- Do not claim a confirmed root cause from a first-pass bundle unless at least
  two evidence classes support it.
- Missing metric series is a visibility or data gap, not proof of normality.
- Empty active-alert, release, or task lists may be absence evidence when the
  source status is complete.
- A QPS or traffic warning alone is not an incident when alerts, restarts,
  errors, and latency are normal.

## Intent Router

### Application Status Lookup

Use when the user asks to look up current status without diagnosis language.

First read:

```bash
whale status application --name <app> --env <env>
```

Stop after this read unless status is materially abnormal or the user asks for
analysis.

### Application Health Analysis

Use when the user asks whether an application is healthy, abnormal, or worth
handling.

First read:

```bash
whale snapshot application --name <app> --env <env> --profile triage --format summary
```

Fallback only when snapshot is unavailable, denied, or incomplete:

```bash
whale status application --name <app> --env <env>
whale search application --view metric --name <app> --env <env> --type error-logs
whale search application --view metric --name <app> --env <env> --type latency
whale search application --view metric --name <app> --env <env> --type traffic
```

Deepen candidates:

- instance distribution:
  `list appinstance --of application=<app>`, then `status appinstance <id>`
- pod visibility:
  `list pod --of appinstance=<id>`, `search pod --name <app> --namespace <ns> --status <status>`, or `show pod <pod-id>`
- pod runtime evidence:
  `list event --of pod=<pod-id>`,
  `search log --of pod=<pod-id> --contains <text> --start <ts> --end <ts>`,
  `count log --of pod=<pod-id> --contains <text> --start <ts> --end <ts>`
- cluster pressure:
  `snapshot cluster --name <cluster> --format summary` or
  `snapshot cluster --zone <zone> --format summary`
- node pressure:
  `list node --cluster-id <cluster-id> --status fail`, then
  `snapshot node <node-id> --window <duration> --format summary`
- logs: bounded application logs, SLS, or VictoriaLogs
- topology or trace: only when dependency or path suspicion is material
- release or task context: when timing or deployment evidence points there

### Current Alerts

Use when the user says `当前告警`.

Phase-1 purpose:

- give a triage direction, not full alert inventory
- prioritize one current handling lane
- state first-page scope and material gaps

Wave 1:

```bash
whale snapshot alert current --format summary
```

Use `quality`, `sources`, `signal_keys`, `alert_context`,
`application_context`, and `next_reads` for the first update. This snapshot
returns the first active-alert page and one representative actionable alert path
in a single bundle.

Representative priority remains: active application or workload availability
alerts first, then resource saturation or capacity alerts, then other
infrastructure alerts. Break ties by severity and recency.
Do not treat stale, ignored, recovered, or resource-less alerts as active
incidents.
App-less infrastructure alerts such as RDS, VPN, cluster, node, pod,
deployment, or container alerts may still be active resource incidents; label
them as resource or workload incidents and state the missing application
mapping.
Do not expand later pages before the first answer.

Do not run `correlate`, second alert snapshots, appinstance/pod reads, logs,
SLS, VictoriaLogs, trace, topology, or release-impact reads before the first
answer.

### Specific Alert

First read:

```bash
whale snapshot alert <id> --format summary
```

Fallback only when the snapshot is unavailable or lacks app or env:

```bash
whale show alert <id>
whale show alert <id> --view timeline
whale status application --name <app> --env <env>
whale search application --view metric --name <app> --env <env> --type <targeted-metric>
```

Metric choice:

- error logs or exception: `error-logs`
- latency or timeout: `latency`
- traffic or QPS: `traffic`
- CPU: `cpu`
- memory or heap: `heap`
- FullGC or major GC: `major-gc-count`
- minor GC: `minor-gc-count`
- fallback: `traffic`

### Release Impact

First read:

```bash
whale snapshot release <id> --format summary
```

Deepen only when needed:

```bash
whale show release <id>
whale list job --of release=<id>
whale search log --of release=<id> --keyword failed
whale show approval --of release=<id>
whale list deploy-record --keyword <app>
```

### Task Context

First read:

```bash
whale snapshot task <id> --format summary
```

### Correlation Requests

Use when the user asks whether alerts, releases, tasks, logs, or symptoms are
related in one time window.

First read:

```bash
whale correlate application --name <app> --env <env> --window <duration> --format summary
```

## Log Rules

### SLS

- For totals, prefer `whale count log --view sls --query '<expr>'`.
- Legacy `search ... '| select count(1) as cnt'` remains compatible.
- Never infer true totals from `--limit` or normal response `total`.
- Page normal SLS results with `--page` and `--limit 100`.
- If the console id or name is unknown, discover it first:
  `whale list log-console --keyword <text> --page-size 100 --format summary`

### VictoriaLogs

- For totals, use `whale count log --view vlogs --query '<LogsQL>'`.
- Keep filters inside LogsQL.
- Prefer explicit `--start` and `--end`; Whale otherwise defaults to latest
  15m.
- For syntax uncertainty, consult the official VictoriaLogs docs instead of
  inventing syntax:
  - `https://docs.victoriametrics.com/victorialogs/logsql/`
  - `https://docs.victoriametrics.com/victorialogs/querying/`

## Constraints

- If `show --name` returns `409`, ask one clarification question with
  candidates.
- Metric query failure is a visibility gap, not proof that a metric is normal.
- Confirmed root cause still requires at least two evidence classes.
