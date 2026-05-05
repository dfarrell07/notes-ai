---
date: 2026-05-04T18:00:00-04:00
topic: slides
tags: [outline, slide-deck, talk-prep, skill-design-patterns]
---

# Slide Outline: Skill Design Patterns

General-audience talk about patterns for building Claude skills and
agent automation. Examples drawn from release management automation
(~15 skills) but patterns are universally applicable.

<!-- Slide content = what's on screen (short). [Notes] = what you say. -->

---

## Part 1: Introduction

### Slide 1: Title

- Skill Design Patterns

### Slide 2: Topic

- Patterns that emerged while creating skills for project automations
- Lessons from improving agent reliability, skill repeatability

### Slide 3: Level-Setting

- "Skill" = Claude skill
- Claude interactive -> automation that runs Claude non-interactively
- 19 skills for release management, goal: handoff-ready

### Slide 4: What a Skill Ecosystem Looks Like

- 19 skills | 27 make targets | 28 scripts (9,772 lines)
- 41 workflow docs (context for both humans and agents)
- 7+ repos, 20-step lifecycle, previously entirely manual

### Slide 5: Demo Start

- Starting a skill in the background — check results at end
- [Note: use /cve-fix. Don't explain what it does; the point is
  it's running autonomously.]

---

## Part 2: Core Lessons

### Slide 6: Core Lesson: New Domains of Automation

- Complex automation that wasn't practical before — now feasible
- Still build on shared, quality tools
- Processes should be Written-Down-as-Automation
- Shared skills and custom skills both valuable

### Slide 7: Core Lesson: Design Around Indeterminism

- Extract as much work to deterministic logic as possible
- Agent call = function call for "fuzzy" logic
- [Note: deterministic part collects data and attempts fixes; agent
  reviews, judges, handles ambiguity. Example:
  [fix-all.sh](https://github.com/submariner-io/shipyard/pull/2383)
  tries every CVE fix deterministically, exits with "needs review"
  for ambiguous cases.
  [bundle-image-update.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/bundle-image-update.sh)
  is 536 lines of deterministic SHA work — agent only for release
  notes. Salesforce calls this "guided determinism."]

### Slide 8: Core Lesson: Design the Context

- Interactive: more context is typically better
- Non-interactive: unnecessary context causes inconsistencies
- [Note: Anthropic calls this "context engineering." Non-interactive
  agents are single-shot (claude -p --print) — principle of least
  privilege for context. Example:
  [release-status.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-status.sh)
  (1,474 lines) crafts focused context for both humans and agents.]

---

## Part 3: Design Patterns

Each pattern: concept slide then example slide.

### Slide 9: Design Pattern: Pseudocode as Prototype

- Skills start as English in Markdown, but run like code
- Recommended way to get started
- Four phases — most skills converge at Phase 3

### Slide 10: Phase 0 — Mostly md

- English process doc — Claude follows it like a runbook
- [submariner/.agents/workflows/cve-fix.md](https://github.com/submariner-io/submariner/blob/devel/.agents/workflows/cve-fix.md)
- [submariner-release-management/.agents/workflows/scan-cves.md](https://github.com/stolostron/submariner-release-management/blob/main/.agents/workflows/scan-cves.md)

### Slide 11: Phase 1 — Mixed md/sh

- Markdown structure + embedded bash for precision
- Hard to maintain at ~1,000 lines
- [skills/cve-fix/SKILL.md](https://github.com/submariner-io/shipyard/pull/2383/files)

### Slide 12: Phase 2 — Mostly sh

- Scripts do the heavy lifting, skill wraps them
- /add-release-notes ->
  [scripts/release-notes/auto-apply.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/auto-apply.sh)
- /cve-fix ->
  [shipyard/pull/2383](https://github.com/submariner-io/shipyard/pull/2383)

### Slide 13: Phase 3 — All sh, optional agent

- Script runs standalone — three entry points:
  - Script (CI) | Make (humans) | Skill (Claude)
- [scripts/create-component-release.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/create-component-release.sh)
  \+ [SKILL.md](https://github.com/stolostron/submariner-release-management/blob/main/skills/create-component-release/SKILL.md)
  \+ [Makefile](https://github.com/stolostron/submariner-release-management/blob/main/Makefile#L93)
- [scripts/rpm-lockfile-update.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/rpm-lockfile-update.sh)
  \+ [SKILL.md](https://github.com/stolostron/submariner-release-management/blob/main/skills/rpm-lockfile-update/SKILL.md)
  \+ [Makefile](https://github.com/stolostron/submariner-release-management/blob/main/Makefile#L101)
- (many more end in this state)
- [Note: entry points are thin aliases — value is meeting users
  where they are. Four skills refactored from Phase 1 to Phase 3
  in a single day.]

### Slide 14: Design Pattern: Pulse-Agnostic Docs

- Context for agents that's also docs for humans
- Write once, serve both audiences

### Slide 15: Example: Pulse-Agnostic Docs

- When | Process | Done When
- Same doc: trigger/precondition, steps/instructions,
  checklist/verification
- [submariner-release-management/.agents/workflows/](https://github.com/stolostron/submariner-release-management/tree/main/.agents/workflows)
- [submariner-operator/.agents/workflows/](https://github.com/submariner-io/submariner-operator/tree/devel/.agents/workflows)
- [Note: 22 docs in orchestration repo + 5 upstream repos — same
  format everywhere. Zero duplication.]

### Slide 16: Design Pattern: Small World, Many Agents

- Pre-fetch evidence deterministically, create focused context
- Many agents in parallel, each on a discrete problem

### Slide 17: Example: Small World, Many Agents

- /add-release-notes:
  [collect](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/collect.sh)
  -> [prepare](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/prepare.sh)
  -> [review](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/review.sh)
  \+ [prompt](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/review-prompt.md)
- /cve-fix:
  detect -> fix-all -> review + prompt
  ([PR](https://github.com/submariner-io/shipyard/pull/2383))
- Agent evaluates, never searches — per-unit revertable commits

### Slide 18: Design Pattern: Proper Plans

- Plans for agents deserve more rigor, not less
- Ambiguity a human navigates = failure mode for agents

### Slide 19: Example: Proper Plans

- [SEP-0031](https://github.com/submariner-io/enhancements/blob/devel/seps/SEP-0031-modernize-enhancements.md)
  -> [enhancements/pull/267](https://github.com/submariner-io/enhancements/pull/267)
- [SEP-0032](https://github.com/submariner-io/enhancements/pull/268)
  -> [shipyard/pull/2383](https://github.com/submariner-io/shipyard/pull/2383)
- Enhancement proposals as agent-consumable specs
- configure-downstream.sh: 49 files, 3 commits
- [Note: like plan mode extracted to a shared repo for review.
  Planning phase was longer than implementation.]

---

## Part 4: Wrap Up

### Slide 20: Lessons Learned

- Inline skills unmaintainable at ~1,000 lines -> extract to scripts
- `set -e` silently swallows errors -> use `&&`/`||`
- Real usage is the only reliable test
- Human-in-the-loop at danger points

### Slide 21: Demo End

- Check on skill results

### Slide 22: What's Next

- Conductor: meta-skill, stateless re-evaluation, no state machine
- "Quality bar shifts to the state checker's thoroughness"
- Open: hosting with many secrets? Top-level UI?
- [Note: IaC idempotent convergence (Ansible/Terraform). Hosting:
  ephemeral containers + credential vaults. UI: Slack + MCP.]

---

## Possible Additions

- **Live demo of release-status.sh** — concretize "Design the Context"
- **Audience interaction** — "what processes could be
  written-down-as-automation?"
- **Timing data** — Y-stream: 17-50h, Z-stream: 13-42h
- **Plugin distribution** — skills as installable packages
