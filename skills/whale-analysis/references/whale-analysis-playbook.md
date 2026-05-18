# whale Whale Analysis Playbook

## Purpose

This document is the shared read-only playbook for Whale analysis skills.
Hermes adapters should derive their workflow from this file and from the
canonical gateway contract surfaced through `/cli/v1/debug/schema` and
`/cli/v1/debug/explain`.

Metadata host:
- [`whale-analysis-playbook.json`](./whale-analysis-playbook.json)

## Rules

- Use `whale` semantic commands only: `list`, `show`, `search`, `status`, `debug`.
- Keep the workflow read-only.
- Default to `quick-triage`: produce a first-pass handling recommendation within 90 seconds.
- Quick triage is not root-cause confirmation. It must use "suspected", "likely", or "needs confirmation" unless deep evidence confirms the cause.
- Quick triage has a strict command budget: normally 4 commands, maximum 5 commands.
- Quick triage output must include: conclusion grade, confidence, evidence, what is not confirmed, and the next best read-only action.
- Do not run logs, SLS, trace, topology, appinstance template/runtime, release approval, or broad debug output in quick triage unless the user explicitly asks for that evidence.
- Use exactly one targeted metric in alert quick triage. Do not fetch `traffic`, `latency`, and `error-logs` by default.
- For application health quick triage, use status plus the two highest-signal metrics: `error-logs` and `latency`.
- Treat metrics as corroborating evidence, not decoration. Without at least two evidence classes, report "suspected" rather than "confirmed".
- Do not invent compatibility, deprecation, or alias semantics outside gateway debug output.
- Organize reasoning by scenario, not by resource inventory.

## Execution Modes

### Quick Triage

Target:

- finish the first pass within 90 seconds
- decide whether the user should treat the case as active incident, recovered alert, stale/ignored alert, or needs targeted deepening
- return one concrete next step, not a full investigation tree

Default behavior:

- if the input is `当前告警`, start with `list alert --status alarm --page-size 10`
- if the input is `最近告警`, start with `list alert --page-size 10`
- if multiple alerts are returned, summarize up to 10 and quick-triage only the newest actionable alert with an application name
- if an alert is old, `ignore=true`, `status=recover`, or `status=recoverbyfix`, do not present it as an active incident
- for application health, do not fetch appinstance or pod by default unless `status application` is abnormal or the user asks for instance-level analysis
- defer logs, SLS, topology, trace, release, template, and approval to deep mode

Output contract:

```text
结论等级: active | recovered | stale_or_ignored | healthy | inconclusive
置信度: high | medium | low
证据:
- ...
不能确认:
- ...
下一步:
- ...
```

### Deepen On Demand

Trigger only when:

- the user says "继续", "深入", "确认根因", "查日志", "查 SLS", "查 trace", "查拓扑", or equivalent
- quick triage is inconclusive and one extra evidence class can materially change the recommendation
- release/config/root-cause confirmation is required

Deep mode can use appinstance, pod, logs, SLS, trace, topology, release, task, and approval reads, but it must still stay read-only.

## Targeted Metric Selection

- ERROR logs, error count, exception logs: `error-logs`
- latency, timeout, slow call, Dubbo method duration: `latency`
- traffic, QPS, request volume, no traffic: `traffic`
- CPU: `cpu`
- memory, heap: `heap`
- FullGC, major GC: `major-gc-count`
- minor GC: `minor-gc-count`
- fallback when no signal is obvious: `traffic`

## Canonical Command Patterns

### Application

- `list application --keyword <text>`
- `search application --keyword <text>`
- `show application --name <app-name>`
- `show application <id>`
- `status application --name <app-name> --env <env>`
- `search application --view metric --name <app-name> --env <env> --type <metric>`
- `show application --view topology --name <app-name>`

### App Instance and Pod

- `list appinstance --of application=<app-name>`
- `list appinstance --of application=<application-id>`
- `show appinstance <id>`
- `status appinstance <id>`
- `list pod --of appinstance=<id>`
- `search pod --name <app-name> --namespace <ns> --status <status>`
- `show appinstance <id> --view runtime`
- `show appinstance <id> --view template`

### Observability

- `list alert --keyword <text>`
- `list alert --status alarm --page-size 10`
- `show alert <id>`
- `show alert <id> --view timeline`
- `list event --keyword <text>`
- `show event <id>`
- `search log --of application=<app-name> --env <env> --contains <text> --start <ts> --end <ts>`
- `search log --view sls --id <console-id> --query <expr> --start <ts> --end <ts> --page 1 --limit 100`
- `search log --view sls --id <console-id> --query '<filter> | select count(1) as cnt' --start <ts> --end <ts>`
- `search trace --name <app-name> --page 1 --page-size 20`

SLS count semantics:

- Use `--limit` only for sample rows, not for totals.
- If the user asks how many logs match a condition, run a bounded `count(1)` query and report the `cnt` field.
- If a normal SLS search response includes `total`, treat it as response shape only unless the query itself is a count aggregation.
- Backend supports auto-segmenting bounded normal-log and single-row `count(1)` queries up to 24h. Skills must not emulate time slicing.
- After count, fetch rows with `--page N --limit 100`; stop after the counted page range is covered or `has_more=false`.

### Release, Task, Approval

- `list release --keyword <text>`
- `show release <id>`
- `list job --of release=<id>`
- `list deploy-record --keyword <app-name>`
- `search log --of release=<id> --keyword <text>`
- `show approval --of release=<id>`
- `list task --keyword <text>`
- `show task <id>`
- `show approval --of task=<id>`

## Playbooks

### Alert Quick Triage

For a specific alert:

1. `show alert <id>`
2. `show alert <id> --view timeline`
3. `status application --name <app-name> --env <env>`
4. `search application --view metric --name <app-name> --env <env> --type <targeted-metric>`

Handling guidance:

- Active alert + abnormal app status or metric anomaly: recommend immediate owner follow-up and one targeted deep check.
- Recovered alert + normal app status: recommend review in the alert time window only if recurrence or impact is reported.
- Old or ignored alert: state that it should not be treated as a current incident without fresh evidence.
- Missing app name or env: summarize the alert and ask for the missing selector or recommend a bounded `list/search` path.

### Current Alerts

1. `list alert --status alarm --page-size 10`
2. Summarize up to 10 alerts by id, app, env, status, age, ignore flag, and message.
3. Quick-triage only the newest actionable alert with app and env.
4. If all alerts are stale, ignored, or missing app context, say so and do not deep dive automatically.

### Recent Alerts

1. `list alert --page-size 10`
2. Summarize up to 10 alerts by id, app, env, status, age, and message.
3. Quick-triage only the newest alert with app and env.
4. If the newest alert is recovered, output a recovery/review recommendation, not an active incident recommendation.

### Application Health Quick Triage

For `meta-api` or any app health request:

1. `status application --name <app-name> --env <env>`
2. `search application --view metric --name <app-name> --env <env> --type error-logs`
3. `search application --view metric --name <app-name> --env <env> --type latency`

Handling guidance:

- Normal status + no metric series or no anomaly: report healthy or no obvious app-level issue, confidence medium.
- Abnormal status or metric anomaly: recommend deepening into appinstance/pod or bounded logs depending on the symptom.
- Do not claim "fully healthy" without appinstance/pod/log evidence.

### Deepen On Demand Command Set

Use only after quick triage or explicit user request:

- instance scope: `list appinstance --of application=<app-name>`, then `status appinstance <id>`, then `list pod --of appinstance=<id>`
- resource/JVM metrics: `cpu`, `heap`, `minor-gc-count`, `major-gc-count`
- logs: `search log --of application=<app-name> --env <env> --contains <text> --start <ts> --end <ts>`
- SLS: count first when the user asks for totals, then page sample logs
- topology/trace: use only to confirm dependency or traffic-path suspicion
- release impact: `list release --keyword <app-name> --page-size 5`, `list job --of release=<id>`, `search log --of release=<id> --keyword failed`

## Constraints

- `search log --of pod=...` is not enabled.
- `show --name` returning `409` requires one clarification question with candidates.
- `show approval --of release=<id>` is conditional on release state and must not block the rest of the analysis.
- `search log --view sls ...` is conditional on log-console visibility and must not block the rest of the analysis.
- Metric query failure is a visibility or upstream-data gap, not proof that the metric is normal. Record the gap and continue.
- A confirmed root cause requires at least two evidence classes, for example status/pod plus metric, metric plus log/trace, or release plus metric.
- Quick triage must not perform write actions, retries, restarts, or approval operations.
