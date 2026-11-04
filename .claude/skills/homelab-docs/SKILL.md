---
name: homelab-docs
description: Write, edit, and review the homelab docs (docs/, README.md, PLAN.md) to the house style. Use when creating or changing any doc in this repo, documenting a component, or reviewing a docs diff.
---

# Homelab docs

The readers are Jake, months from now, and other people rebuilding this setup on their own
machines ([reproduce](../../../docs/reproduce.md)). They know Kubernetes basics. They don't know
why this cluster is built the way it is, or which piece lives where.

[STYLE.md](STYLE.md) holds the rules. [PAGES.md](PAGES.md) holds the page shapes. Read both in
full before you write or review. Each rule carries an "Instead of" and "Write" pair; the pair
shows what the rule means better than the rule's sentence does.

## 1. Gather facts from the repo

Take every fact from the source of truth, never from another doc, because other docs go stale:

- **Manifests:** names, namespaces, ports, versions, flags, and labels come from the YAML under
  `apps/`, `infrastructure/` and `clusters/`
- **Live state:** if a fact only exists in the cluster, give Jake the `kubectl` or `flux` command
  to run and ask for the output
- **Outside git:** Infisical Cloud folders, Cloudflare settings, UI steps, and anything done by
  hand on a node. Ask Jake, and write only what Jake confirms

Done when every name, path, port and version on the page traces to a file or to Jake.

## 2. Draft

Pick the page shape from [PAGES.md](PAGES.md). Reach for a diagram whenever two or more parts
connect.

## 3. Review

Review your own draft as you'd review anyone's. Work in order: cut, then structure, then
sentences. Rewriting a sentence that should be cut is wasted work.

1. **Cut.** Read one paragraph at a time. For each fact, ask what the reader would do differently
   after reading it. If the answer is nothing, cut it, even if it's true
2. **Structure.** Check the page against its shape in [PAGES.md](PAGES.md) and against the
   "Pages" rules in [STYLE.md](STYLE.md)
3. **Sentences.** Take the "Sentences" rules in [STYLE.md](STYLE.md) one heading at a time, and
   check every sentence on the page against that heading's rule
4. **Mechanics.** Run the checks in the "Mechanics" section of [STYLE.md](STYLE.md)
5. **Re-review.** Run steps 1 to 4 again on the edited page. Edits create new problems

Done when a full pass of steps 1 to 4 finds nothing. Before you hand the page back, also check it
for every correction Jake has made earlier in the conversation.

If Jake wrote prose in this conversation, report findings on those sentences instead of
rewriting them, and ask once whether to apply them.

## 4. Wire it in and commit

- Add a new page to the "Docs" list in `README.md`, with a one-line summary after the colon
- Link the page from the docs that mention its component
- Commit each change on its own, lowercase message, and push
