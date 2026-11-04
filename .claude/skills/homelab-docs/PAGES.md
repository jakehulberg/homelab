# Page shapes

Pick the shape that matches the page's purpose. Drop sections the page has nothing for; keep the
order of the rest.

## Component page

One per thing that runs on the cluster. Examples: [airlock](../../../docs/airlock.md),
[agent-vault](../../../docs/agent-vault.md), [certificates](../../../docs/certificates.md).

```
# <Component>

<What it is. Why this cluster runs it. Who uses it, with links.>

## How it fits together
<ASCII diagram, then a numbered list matching the diagram's numbers>

## What's where
| Path | What |
<every file or folder in git, plus a row for what lives outside git (Infisical, Cloudflare, a node)>

## Secrets
| Secret | What | Used by |
<Infisical Cloud folder, and how each value is generated, e.g. `openssl rand -hex 32`>

## <One section per non-obvious decision or mechanism>
<Why it's set this way. A diagram if parts connect.>

## Checking it
<sh block: one command per line, trailing comment with the healthy result>

## Troubleshooting
- **<Symptom as the reader sees it>:** cause, then the command or fix
```

## Task page

How to do one thing, start to finish. Examples: [bootstrap](../../../docs/bootstrap.md),
[reproduce](../../../docs/reproduce.md), [nodes](../../../docs/nodes.md#taking-a-node-down).

```
# <Task>

<What the reader ends up with, in one or two sentences.>

## Prerequisites
<Checkable items, or a table: Need | Notes>

## 1. <Step>
<Where to run it (laptop, a mini, Infisical UI), the command, and how to tell it worked>
```

Number the steps in the headings so other pages can link them (`bootstrap.md#9-sops-key`).

## Incident page

`docs/incidents/YYYY-MM-DD-<slug>.md`. Example:
[2026-10-25](../../../docs/incidents/2026-10-25-infisical-db-failover.md).

```
# YYYY-MM-DD: <What broke>

**Impact:** <what didn't work, for how long, data lost or not>

## What happened
<numbered, in order>

## Root causes
## Fixes
<each linked to the doc section that now describes the fixed setup>

## Lessons
```

## Cheatsheet

[cheatsheet](../../../docs/cheatsheet.md): one `##` per tool or area, an `sh` block of everyday
commands with trailing comments, and a one-line link to the full doc. No explanation beyond that.

## README docs list

One line per page: `- [Title](docs/<page>.md): <what the reader finds there>`, in reading order
for someone new.
