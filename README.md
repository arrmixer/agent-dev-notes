# agent-dev-notes

A working knowledge base for building AI agents — distilled from real projects, refined as the journey
continues. The goal: a **springboard** for future projects, and the source material for Claude Code
**rules** (always-on principles) and **skills** (task playbooks).

## Start here

Building a new agent system? Read in this order:
1. **`principles/`** — the provider-agnostic rules that should hold on *any* agent project. Read these first; they're short.
2. **`playbooks/`** — concrete how-tos for a given platform (currently AWS Bedrock AgentCore). Grab the one you need.
3. **`retrospectives/`** — full worked systems to copy structure from. `capstone-cd14764.md` is the reference blueprint.

## Layout (tiered by portability)

| Dir | What | Half-life | Feeds |
|---|---|---|---|
| `principles/` | provider-agnostic agent-dev rules (one file per rule: terse statement + evidence) | years | Claude Code **rules** |
| `playbooks/` | AWS/AgentCore mechanics (deploy, guardrails, KBs, memory, observability) | until APIs change | Claude Code **skills** |
| `language/` | Python-via-Swift/Kotlin notes (personal onboarding) | personal | reference |
| `retrospectives/` | per-project blueprints + "if I built this again" | per project | springboard |
| `_archive/` | **journey log** — chronological story, snags, dead-ends, aha moments | narrative | **blog post / presentation source** |

## The distillation convention

Every principle doc leads with a **one-line rule** (the thing an agent/you should always do), then the
**evidence** (the war story that earned it). The rule is what becomes a Claude Code rule; the evidence is
what makes it stick. Example: *"A declared guardrail is not an enforced one — verify from raw assessments,
not config."* → rule. Below it: the capstone run where mild profanity slipped the managed list.

**Where the "why" lives:** decision *provenance* (the proof behind a rule) stays in that rule's evidence
section — never archived, so it can't rot unread. `_archive/` is for the *narrative* — the chronological
story written to be mined later for a blog post or presentation (including dead-ends, which are the thing
everyone loses). Two different kinds of "why," two different homes.

## Install / go live

Two different mechanisms, because **skills are plugin-distributable but rules are not**:

**Skills — via the plugin** (`agentcore-playbooks`, listed in the `agent-dev-notes` marketplace here):
```bash
claude plugin marketplace add ./          # from this repo dir (or: <git-url> / owner/repo)
claude plugin install agentcore-playbooks@agent-dev-notes
```
Installed skills are namespaced `/agentcore-playbooks:<skill>`. One-command install on any machine; update
with `claude plugin update agentcore-playbooks@agent-dev-notes`.

**Rules — by symlink** (Claude Code cannot package rules in a plugin — they must be project-local or in
`~/.claude/rules/`). To make the principles always-on globally:
```bash
for f in .claude/rules/*.md; do ln -sf "$PWD/$f" "$HOME/.claude/rules/$(basename "$f")"; done
```
(`~/.claude/rules/` requires Claude Code 2.1.200+. Or fold them into `~/.claude/CLAUDE.md`.)

The repo stays the **canonical source** for both; "going live" is the deliberate step above.

## Status

Seeded from `~/Developer/agentcore-reference/` after passing Udacity cd14764 (AgentCore multi-agent
capstone), 2026-10-11. Content migration **complete**: 8 principles + 8 rules + 6 skills + language +
retrospective + archive. Packaged as a plugin/marketplace. See `MIGRATION.md` for the tier map.
