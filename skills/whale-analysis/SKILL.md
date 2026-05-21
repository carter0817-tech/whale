---
name: whale-analysis
description: >
  Analyze Whale applications, alerts, releases, tasks, logs, metrics,
  topology, traces, and cost using read-mostly Whale semantic commands. Use
  for SRE first-pass triage, current alerts, application health, release/task
  context, bounded log analysis, and evidence-backed operational diagnosis.
version: 2.0.0
platforms: [macos, linux]
metadata:
  hermes:
    category: sre
    tags: [whale, sre, observability, incident-response, triage]
    requires_toolsets: [terminal]
---

# Whale Analysis

Use this skill when the user asks for Whale operational analysis such as
application health, current alerts, specific alerts, release impact, task
context, log evidence, trace or topology context, or cost context.

## Role

This skill is a scenario adapter, not a second resource kernel.

- Gateway or resource kernel owns resource semantics, evidence bundles,
  permissions, pagination, query rules, and compatibility.
- `whale` CLI owns command grammar, auth and config, installation, version,
  help, and rendering such as `--format summary`.
- This skill owns natural-language routing, first-pass evidence selection,
  stop or deepen decisions, and concise conclusions.

When a deterministic rule is missing, prefer a gateway or CLI evidence-bundle
improvement over long prompt-only fallback behavior.

## Allowed Surface

Use only Whale semantic commands:

- `list`
- `show`
- `search`
- `count`
- `status`
- `snapshot`
- `correlate`
- `debug`

Default to read-mostly analysis. Do not perform approvals, retries, scaling,
stop, rollback, release application, ticket creation, or raw K8S or CMDB
writes.

The only controlled write surface in this phase is `whale workload restart`.
Use it only when the user explicitly asks for restart. First run it without
`--yes` to obtain policy and audit context. Execute with `--yes` only if the
policy returns `direct_allowed` and the user explicitly confirms execution.

## First Update Model

Default to a useful first update, then deepen only when needed.

First-pass target:

- usually within 60 seconds
- answer current judgment, confidence, handling suggestion, material evidence,
  and material gaps
- do not try to prove root cause in the first response unless evidence already
  supports it

Prefer gateway or CLI evidence bundles:

- application health:
  `whale snapshot application --name <app> --env <env> --profile triage --format summary`
- alert: `whale snapshot alert <id> --format summary`
- current alerts: `whale snapshot alert current --format summary`
- release: `whale snapshot release <id> --format summary`
- task: `whale snapshot task <id> --format summary`
- time-window relationship:
  `whale correlate application --name <app> --env <env> --window <duration> --format summary`

For simple status lookup, run only
`whale status application --name <app> --env <env>` first. If no environment is
provided, use `production-b2b-cn` and say so briefly.

For `当前告警`, phase 1 is triage direction, not full inventory or root-cause
proof.

Allowed first-pass command:

- `whale snapshot alert current --format summary`

Representative alert priority from the first page: active application or
workload availability alerts first, then resource saturation or capacity
alerts, then other infrastructure alerts; break ties by severity and recency.

Do not expand pagination, run multiple alert snapshots, run `correlate`,
inspect logs, inspect pod or appinstance, or inspect release impact before the
first answer.

## Deepen Triggers

Automatically deepen only when:

- bundle `quality` is insufficient
- an anomaly is clear and the allowed first-pass evidence lacks enough
  owner, resource, time, or severity context to give a useful handling
  suggestion
- evidence classes conflict
- the user explicitly asks to continue, confirm root cause, check logs, check
  trace, check instances, or inspect release impact

Do not fetch appinstance, pod, logs, SLS, VictoriaLogs, trace, topology,
approval, runtime/template, or broad debug output in the first pass unless the
user asks for that evidence.

For `当前告警`, do not treat "there is an obvious next read" as a deepen trigger
by itself; put it in `下一步` unless the user asks to continue.

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

For status-only requests, answer shorter: one conclusion line, up to 4 evidence
bullets, and one optional next action.

## Required Routing Rules

- Treat `当前告警` as `whale snapshot alert current --format summary`.
- For `当前告警`, do not expand later pages before the first answer.
- For `当前告警`, use the snapshot-bundled representative alert before
  considering any second alert read.
- For `当前告警`, pick the representative alert by impact class first, not only
  by recency.
- For `当前告警`, run `correlate` only after the first answer or when the user
  explicitly asks for release or time-window correlation.
- Do not present stale, ignored, recovered, or resource-less alerts as active
  incidents.
- App-less RDS, VPN, cluster, node, pod, deployment, or container alerts may
  be active resource incidents; label them as resource or workload incidents
  and state the missing application mapping.
- Prefer exact `id` when provided.
- If only a `name` is provided, follow canonical resource paths. For
  `appinstance`, discover by `application` first, then switch to instance `id`.
- If `show --name` returns `409`, ask one clarification question with
  candidates.
- Treat missing metric series as an evidence gap, not healthy evidence.
- Treat QPS or traffic warning alone as observe-level when alerts, restarts,
  errors, and latency are normal.
- Use canonical pressure reads only after the first answer or when the user
  asks for resource deepening:
  - `whale snapshot cluster --name <cluster> --format summary`
  - `whale snapshot cluster --zone <zone> --format summary`
  - `whale snapshot node <id> --window <duration> --format summary`
- When the user asks for instance-level proof, prefer the canonical pod
  evidence chain:
  - `whale list pod --of appinstance=<id>`
  - `whale show pod <pod-id>`
  - `whale list event --of pod=<pod-id>`
  - `whale search/count log --of pod=<pod-id> ...`
- Treat `search/count log --of pod=<pod-id>` as bounded historical Pod log
  evidence over a time window, not live tail or websocket stream.
- If the user explicitly asks for real-time Pod log tail, state that the
  current Whale analysis surface does not expose live stream Pod logs in this
  phase.
- For SLS count intent, prefer `whale count log --view sls --query '<expr>'`.
  The legacy `search ... '| select count(1) as cnt'` form remains compatible.
- If the user wants SLS but does not know the console id or name yet, enumerate
  accessible consoles first with
  `whale list log-console --keyword <text> --page-size 100 --format summary`.
- For VictoriaLogs totals, use
  `whale count log --view vlogs --query '<LogsQL>'`.
- When reading VictoriaLogs, prefer explicit `--start/--end`; if omitted,
  Whale defaults to the latest 15m.
- For heavier exact-token or `_msg` reads, `--timeout 60s` is available, but
  narrowing with app, namespace, or pod filters still takes priority.

## References

- Load
  [references/whale-analysis-playbook.md](./references/whale-analysis-playbook.md)
  for deeper scenario orchestration, exact command patterns, SLS and
  VictoriaLogs details, and runtime caveats.
- Load
  [references/whale-analysis-playbook.json](./references/whale-analysis-playbook.json)
  for machine-readable version, policy, and compatibility metadata.
