# Homelab docs style

Adapted from Infisical's docs style guide, cut to what applies to plain Markdown in a repo.

## Content

### Open with what it is and why it's here

The first sentence says what the component is. The next says why this cluster runs it. Then
the how. A reader who stops after two sentences knows whether the page is for them.

**Instead of:** `agent-vault-proxy` runs `infisical/cli` with `agent-vault proxy --port=17323`.

**Write:** Agent Vault keeps credentials out of agent sandboxes. The sandbox sends its traffic
through a proxy, and the proxy adds the real API keys on the way out.

### Leave out what the reader won't act on

Before you add a fact, ask what the reader would do differently after reading it. If nothing,
leave it out. Facts that usually fail:

- Step-by-step internals, when the reader only needs the result
- Warnings about mistakes the steps don't lead the reader into
- Facts another page covers (link to it)
- History: what the setup used to be, unless the reader will meet leftovers of it

If a fact matters only in one situation, put it in "Troubleshooting", next to the symptom.

**Instead of:** A paragraph on how the Infisical Operator polls the API, caches the response and
diffs it against the Secret before writing.

**Write:** A change in Infisical Cloud lands in the cluster within 60 seconds.

### Keep the why, drop the how-it-was-found

The repo shows what's configured. A doc earns its place by saying why: the reason for a
non-default setting, the gotcha no config file confesses. The debugging story belongs in an
incident page.

**Instead of:** We tried the default resolver first, saw 5-second lookups, and after some
digging found the search domains were the problem.

**Write:** k3s `resolv-conf` points at a file with only `1.1.1.1` and `8.8.8.8`. Without it, pods
inherit the Tailscale and router search domains, and in-cluster lookups take up to 5 seconds
([incident](incidents/2026-10-25-infisical-db-failover.md)).

### Link every concept the first time

Link another doc the first time the page mentions its component. Link to a section, not "above"
or "below".

**Instead of:** The operator logs in with the credentials described above.

**Write:** The operator logs in as `homelab-cluster` ([secrets](secrets.md#how-the-operator-connects)).

### Say how, whenever you say the reader can do something

If the page says something can be done, give the command or link the steps.

**Instead of:** The age key can be rotated if needed.

**Write:** To rotate the operator's credentials, follow [Rotate the operator's
credentials](secrets.md#rotate-the-operators-credentials).

### State prerequisites the reader can check

If a page needs something set up first, list it at the top, as something the reader can check
before starting.

**Instead of:** A working cluster.

**Write:** `flux get kustomizations` shows all three Ready.

## Pages

### One purpose per page

A page covers one component or one task. If a reader has to scroll past sections meant for
someone else, split the page.

### One topic per heading

Under one heading, keep one topic and at most two paragraphs. Turn parallel items (options,
secrets, files, cases) into a table or list. Turn a paragraph with several actions into a
numbered list. Don't add a heading over two lines of text; merge it into its neighbour.

### Don't pack separate facts into a paragraph

If you could shuffle a paragraph's sentences without changing its meaning, they're separate
facts. Lead with the main point, then list the rest.

**Instead of:** The proxy needs a token. It's in Infisical Cloud at `/agent-vault`. The operator
syncs it every 60 seconds. If it's missing, the proxy crash-loops.

**Write:** The proxy loads its settings with `INFISICAL_AGENT_VAULT_PROXY_TOKEN`:

- Stored in Infisical Cloud `/agent-vault`, synced into `agent-vault-secrets-infisical`
- If it's missing or revoked, the proxy crash-loops

### Order sections the way a reader looks for them

What it is, how it fits together, what's where, how to change it, how to check it, how to fix
it. Keep sections about the same topic next to each other. Put a note in the section it applies
to.

### Diagrams

Draw a diagram whenever two or more parts connect: request flows, sync paths, auth, which
namespace may reach which. Diagrams are ASCII in a bare fenced block:

- Arrows: `──▶` for a call or flow, `✕` for blocked, labels on the arrow
- Number the arrows when order matters, and follow with a numbered list using the same numbers
- Use real names: Service and port, namespace, Secret name, Infisical folder
- Keep it under 100 columns so it fits a GitHub page without scrolling
- One idea per diagram; draw two before you draw one crowded one

```
Infisical Cloud /agent-vault ──operator──▶ agent-vault-secrets-infisical ──envFrom──▶ proxy
```

### Headings name the topic in the reader's words

Sentence case. Name what the reader is looking for, not an internal term.

**Instead of:** `## TokenReview`

**Write:** `## Kubernetes Auth`

## Sentences

### Don't give human verbs to things that aren't human

A pod, a policy or a proxy doesn't know, want, decide or think. Name what does what.

**Instead of:** The proxy knows which credential to add.

**Write:** The proxy looks up the session's access bundle and adds that bundle's credential.

### Say what a thing is, in one clause

**Instead of:** `wait: true` is a setting that controls whether the Kustomization blocks.

**Write:** With `wait: true`, `infra-configs` isn't Ready until everything it applied is healthy.

### Say it once

If a sentence repeats the one before it, delete the weaker one.

**Instead of:** Secrets live in Infisical Cloud. Cloud is the source of truth. Nothing is stored
in the cluster first.

**Write:** Infisical Cloud is the single source of truth for secrets.

### Keep pronouns next to what they refer to

If the reader has to look back for what "it" means, repeat the noun.

**Instead of:** Rotate the Client Secret, re-encrypt the file, then revoke it.

**Write:** Rotate the Client Secret, re-encrypt the file, then revoke the old Client Secret.

### Give every transitive verb its object

Sync, apply, add, return and create need an object.

**Instead of:** The operator syncs every 60 seconds.

**Write:** The operator syncs each folder into its Secret every 60 seconds.

### Write conditions as "if" clauses

**Instead of:** A node without `runsc` can't run sandboxes.

**Write:** If a node has no `runsc`, sandboxes can't run on it.

### Name what actually happens

Say which system does what, with which input, and the result.

**Instead of:** Access is enforced at the network layer.

**Write:** The proxy's NetworkPolicy accepts connections only from pods in `agents`.

### Say exactly what you mean

Plain, literal English. No metaphors, no figurative verbs.

**Instead of:** Flux is the brain that drives the cluster.

**Write:** Flux applies this repo to the cluster every minute.

### Say what a qualifier refers to

"Default", "standard" and "built-in" need their object: default for what?

**Instead of:** k3s uses the default DNS settings.

**Write:** By default, k3s gives pods the node's `/etc/resolv.conf`.

### Make sentences unambiguous

Words like log, sync, apply, set, run, request, update and cache can be a noun or a verb. If both
readings fit, add a small word ("the", "that", "to") or reorder.

**Instead of:** Flux reconciles kustomizations from the repo every minute.

**Write:** Every minute, Flux applies the Kustomizations in this repo.

### Use contractions

"It's", "don't", "can't", "won't".

### Read it out loud

If you wouldn't say a sentence that way to someone next to you, rewrite it.

## Mechanics

| Rule | Check |
|---|---|
| No em dashes. Use a comma, colon, parentheses or period | `grep -n $'\xe2\x80\x94' <file>` prints nothing |
| Sentence case headings | Read every `#` line |
| Bold only for labels: list-item labels (`**Repo:**`), troubleshooting symptoms, UI labels, and at most one must-not-miss warning per page | Read every `**` |
| Code spans for anything typed or read in code: paths, names, flags, values | Read each line |
| No periods at the end of bullets | Read each `- ` line |
| "Select", not "click" | `grep -n -i "click" <file>` prints nothing |
| Wrap prose at 100 columns | `awk 'length > 100' <file>`; tables and code are exempt |
| Links are relative (`secrets.md#sops`), with working anchors | Each `#anchor` matches a heading |
| Commands in `sh` blocks, with a trailing `#` comment for the expected result | Read each block |
