---
name: smart-<problem-domain>
description: <Problem domain in one sentence. Main libraries / packages as keywords. Use when ... (trigger situations). Snapshot of the upstream skills; escalate to the upstream repositories for library development or deep troubleshooting.>
metadata:
  snapshot-date: "YYYY-MM-DD"
---

# <Problem Domain> (Snapshot)

<!-- Layer 1 - Domain Overview -->

<What the domain is about, in 2-4 sentences.>

- Libraries / repositories in scope: <repo A>, <repo B>
- Use for: <typical situations>
- Out of scope: <exclusions> - see `smart-<neighbor-domain>`

<!-- Layer 2 - Cached Essentials -->

## Snapshot Status

- Snapshot date: YYYY-MM-DD
- Synchronized against: <repo A> @ <tag | version | commit>, <repo B> @ <...>
- Staleness thresholds: fresh < 90 days, aging 90-180 days, stale > 180 days
- Volatility: low | medium | high

Apply the alarm logic from `smart-snapshot-skill-guidelines` before relying on this snapshot.

## Core Concepts

- <concept>: <one-line explanation>

## Typical Usage

<Minimal setup and the most important conventions. Short code snippets only where they save explanation.>

## Quick Troubleshooting

| Symptom | Likely cause | Quick check |
| --- | --- | --- |
| <...> | <...> | <...> |

<!-- Layer 3 - Upstream Escalation -->

## Upstream Sources

| Repository | URL | Skills | Docs | Packages |
| --- | --- | --- | --- | --- |
| <repo A> | <https://github.com/...> | `.agents/skills/` | `doc/` | <package ids> |

### Escalate to upstream when

- developing, extending or fixing the library itself;
- deep troubleshooting needs implementation details not covered above;
- this snapshot is stale or its alarm fires and the task depends on the affected details;
- observed behavior or the consumed package version contradicts this snapshot.

Then load the upstream skills (local clone preferred, otherwise online) and treat them as authoritative.
