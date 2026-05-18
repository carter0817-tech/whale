---
name: whale-analysis
description: >
  Analyze Whale alerts, applications, app instances, releases, logs, traces,
  tasks, and cost with read-only Whale semantic commands. Use for SRE triage,
  current-alert review, app health checks, release impact analysis, and
  evidence-backed operational diagnosis.
version: 1.0.0
platforms: [macos, linux]
metadata:
  hermes:
    category: sre
    tags: [whale, sre, observability, incident-response, triage]
    requires_toolsets: [terminal]
---

# Whale Analysis

Use this skill when the user wants read-only operational analysis against the
Whale CLI, including alert triage, application health checks, release impact
review, bounded log checks, and instance-level deepening.

## Before You Start

- Verify the `whale` CLI is installed and already authenticated.
- Keep the workflow read-only.
- Use only these Whale semantic commands: `list`, `show`, `search`, `status`,
  `debug`.
- Prefer exact IDs when the user gives one.
- Separate collected evidence from your own inference.

If `whale` is missing or unauthenticated, stop and tell the user what is
blocked instead of inventing data.

## Core Workflow

1. Classify the request as one of:
   - current alerts
   - recent alerts
   - specific alert
   - application health
   - deepen on demand
   - release or task analysis
2. Start with quick triage unless the user explicitly asks for deeper evidence.
3. Stay within the quick-triage command budget:
   - normal budget: 4 commands
   - hard budget: 5 commands
4. Use the canonical command patterns from
   [references/whale-analysis-playbook.md](./references/whale-analysis-playbook.md).
5. If quick triage is inconclusive, stop and recommend the single best next
   read-only command instead of silently continuing.

## Quick-Triage Rules

- Treat `当前告警` as `list alert --status alarm --page-size 10`.
- Treat stale, ignored, recovered, or app-less alerts as non-active unless
  fresh evidence says otherwise.
- For alert triage, use exactly one targeted metric chosen from alert content.
- For application health, use `status application` plus `error-logs` and
  `latency`.
- Do not pull logs, SLS, topology, trace, appinstance runtime/template, or
  release approvals during quick triage unless the user explicitly asks for
  depth.
- Do not present a root cause as confirmed unless at least two evidence classes
  support it.

## Canonical Scenarios

### Current Alerts

1. `list alert --status alarm --page-size 10`
2. Summarize up to 10 alerts.
3. Quick-triage only the newest actionable alert with app and env context.

### Recent Alerts

1. `list alert --page-size 10`
2. Summarize up to 10 alerts.
3. Quick-triage only the newest alert with app and env context.

### Specific Alert

1. `show alert <id>`
2. `show alert <id> --view timeline`
3. `status application --name <app-name> --env <env>`
4. `search application --view metric --name <app-name> --env <env> --type <targeted-metric>`

### Application Health

1. `status application --name <app-name> --env <env>`
2. `search application --view metric --name <app-name> --env <env> --type error-logs`
3. `search application --view metric --name <app-name> --env <env> --type latency`

## Deepen On Demand

Only deepen when the user explicitly asks, or when one extra evidence class can
materially change the recommendation.

Use these bounded paths:

- Instance scope: `list appinstance --of application=<app-name>` then
  `status appinstance <id>` then `list pod --of appinstance=<id>`
- Logs: `search log --of application=<app-name> --env <env> --contains <text> --start <ts> --end <ts>`
- Trace: `search trace --name <app-name> --page 1 --page-size 20`
- Topology: `show application --view topology --name <app-name>`
- Release: `list release --keyword <app-name> --page-size 5`, then
  `list job --of release=<id>`, then `search log --of release=<id> --keyword failed`

For SLS totals, follow the exact count rule in
[references/whale-analysis-playbook.md](./references/whale-analysis-playbook.md):
use `| select count(1) as cnt` first, then page normal logs with `--page` and
`--limit 100`.

## Resource-Specific Constraints

- `search log --of pod=...` is not enabled.
- `status appinstance <id>` is the canonical instance health summary.
- If `show --name` returns `409`, ask one clarification question with
  candidates.
- If metrics fail or return no usable series, record an evidence gap. Do not
  treat that as proof of normality.
- Conditional capabilities such as `show approval --of release=<id>` or
  `search log --view sls ...` must not block the rest of the analysis.

## Output Contract

Quick triage should end with:

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

Always label evidence separately from inference. Use words like `suspected`,
`likely`, or `needs confirmation` when evidence is incomplete.

## References

- Load
  [references/whale-analysis-playbook.md](./references/whale-analysis-playbook.md)
  for the full command patterns, scenario playbooks, SLS policies, and deep
  analysis paths.
- Load
  [references/whale-analysis-playbook.json](./references/whale-analysis-playbook.json)
  when you need machine-readable version/support metadata.
