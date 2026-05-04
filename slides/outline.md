---
date: 2026-05-04T17:30:00-04:00
topic: slides
tags: [outline, slide-deck, talk-prep]
---

# Slide Outline: Skill Design Learnings (Expanded Talk)

Longer version of the short talk given 2026-05-04.
Includes original content plus release process automation depth.

<!-- Each slide: title, bullets for content, [notes] for speaker context -->

---

## Part 1: Introduction

### Slide 1: Title

- /CVE-Fix Skill
- And learnings about skill designs

### Slide 2: Framing

- Was asked to talk about /cve-fix
- Real value is in applying skill design lessons to your context
- Will use /cve-fix and release automation as examples for bigger ideas

### Slide 3: Level-Setting

- "Skill" -> "Claude skill"
- Claude interactive to create automation that runs Claude
  non-interactively
- Agents in the context of release management automation, ~20 examples
- All early lessons, would love to hear feedback from others

### Slide 4: Demo Start

- Start /cve-fix skill running in background
- [Note: kick off live, check results at end of talk]

---

## Part 2: Core Lessons

### Slide 5: Core Lesson — New Domains of Automation

- Custom, complex automation that would not have been practical before
  is now possible to produce quickly and maintain
- Still good to build on shared, quality tools
- Processes should be Written-Down-as-Automation
- Shared skills and custom skills are both valuable

### Slide 6: Core Lesson — Design Around Indeterminism

- Extract as much work to deterministic logic as possible
- Treat agent call like a function call for "fuzzy" logic that would
  be overly complex to automate with traditional automation
- [Example: bundle-image-update.sh — 536 lines of deterministic SHA
  extraction/validation, agent only for fuzzy parts]
- [Example: release notes — deterministic filtering reduces 27 issues
  to 6, then per-issue agent review with pre-fetched evidence. The
  agent judges, it does not search.]

### Slide 7: Core Lesson — Design the Context

- In your Claude interactive sessions, more context is typically better
- In Claude instances that are meant to solve a narrower problem
  repeatedly, unnecessary context leads to unwanted inconsistencies
- [Example: release-status.sh (1,474 lines) — crafts focused context
  for both humans and agents to understand release state]

---

## Part 3: What We Built

Context for the design patterns that follow. Shows the scale,
complexity, and growth of the system the patterns were applied to.

### Slide 8: The Release Automation Landscape

- Submariner release lifecycle: 20 steps, previously manual
- 15 Claude Code skills as slash commands
- 27 Make targets (release creation + validation)
- 28 shell scripts (9,776 lines of automation)
- 22 agent workflow docs as detailed runbooks
- 88 release YAML configs across versions 0.20-0.24
- Eliminated the midstream repo entirely
- [Note: these numbers give weight to the patterns — this isn't a
  toy example, it's a production system]

### Slide 9: How the System Evolved

- Phase 1 (early Feb): Validation scripts and make targets. No skills.
- Phase 2 (late Feb): First skills — large, self-contained, inline
  bash in SKILL.md (e.g. /configure-downstream)
- Phase 3 (Mar): Skill explosion — 10 new skills in 3 weeks, covering
  full 20-step workflow
- Phase 4 (Mar-Apr): Real releases (0.22.1, 0.23.1) expose bugs.
  Fix-and-retry cycles drive hardening.
- Phase 5 (Apr 15): Big refactoring — 4 PRs in 1 day extract inline
  logic to standalone scripts, establishing three-entry-point pattern
- Phase 6 (Apr): Agent sophistication — per-issue review agents with
  pre-fetched evidence (PR #49)
- Phase 7 (Apr 30): Conductor vision — meta-skill orchestrating all
  others via stateless re-evaluation (Issue #64)
- [Note: The system itself went through the Pseudocode as Prototype
  phases. Built in ~2.5 months. This slide foreshadows the patterns.]

### Slide 10: Snapshot Pipeline Overview (optional)

- 5 upstream repos (operator, submariner, lighthouse, shipyard, subctl)
- Konflux builds 8 components -> snapshots with image SHAs
- bundle-image-update -> component releases -> FBC catalog
- FBC catalog covers 6 OCP versions (4.16-4.21)
- QE gets catalog URLs via `make get-fbc-urls`
- End-to-end SHA traceability across 10 sources
- [Note: include if audience benefits from understanding the pipeline.
  Skip if time is tight — slide 8 already conveys the complexity]

---

## Part 4: Design Patterns

### Slide 11: Design Pattern — Pseudocode as Prototype

- Skills can start as English descriptions of processes in Markdown,
  but they can run like code
- Recommended way to get started
- Four phases of evolution (following slides)

### Slide 12: Phase 0 — Mostly md

- submariner/.agents/workflows/cve-fix.md
- submariner-release-management/.agents/workflows/scan-cves.md

### Slide 13: Phase 1 — Mixed md/sh

- skills/cve-fix/SKILL.md
- [Note: Many release management skills started here — e.g.
  konflux-component-setup was 1,484 lines of inline bash in SKILL.md.
  This works to get started but becomes unmaintainable.]

### Slide 14: Phase 2 — Mostly sh

- /add-release-notes -> scripts/release-notes/auto-apply.sh
- /cve-fix -> WIP PR: shipyard/pull/2383

### Slide 15: Phase 3 — All sh, optional agent

- Three entry points to the same logic:
  - Script (direct execution, CI pipelines)
  - Make target (parameter validation — for humans)
  - Skill (thin wrapper — for Claude)
- Example: scripts/bundle-image-update.sh is called by both
  `make bundle-image-update` and `/bundle-image-update`
- The April 15 refactoring: 4 PRs in 1 day extracted inline SKILL.md
  logic from 4 skills to standalone scripts (configure-downstream,
  konflux-component-setup, konflux-bundle-setup, bundle-image-update)
- (many more end in this state)
- [Note: Phase 3 is where most skills converge. The refactoring was
  driven by real pain — 1,000+ line SKILL.md files were hard to test,
  hard to debug, and couldn't be used outside Claude.]

### Slide 16: Design Pattern — Pulse-Agnostic Docs

- Create context for agents that's also docs for humans
- Massively increases productivity of docs
- Example: /context

### Slide 17: Pulse-Agnostic Docs — Examples

- submariner-release-management/.agents/workflows
- submariner-operator/.agents/workflows

### Slide 18: Pulse-Agnostic Docs — At Scale

- 22 workflow docs serve as both agent context and team runbooks
- Consistent structure: Title, When, Process, code blocks, Done When
- CLAUDE.md encodes the full 20-step release workflow
  (Y-stream and Z-stream paths)
- Each step references a skill (`@/skill-name`) or a workflow doc
  (`@.agents/workflows/name.md`)
- The documentation IS the automation's interface — same artifact
  drives both human understanding and agent execution
- [Note: The workflow docs follow a consistent format that works for
  both audiences. "When" tells a human the trigger condition; it tells
  Claude the precondition. "Done When" gives a human a checklist; it
  gives Claude a verification command.]

### Slide 19: Design Pattern — Small World, Many Agents

- Craft data to create context for agents
- Invoke many agents in parallel, each focused on a discrete problem

### Slide 20: Small World, Many Agents — Examples

- /add-release-notes:
  collect.sh -> prepare.sh -> review.sh + review-prompt.md
  - Spawns one agent per Jira issue, each with pre-fetched evidence
    (Jira, DFBUGS, GitHub PRs, git logs)
  - Agent evaluates, does not search — evidence is already collected
  - Each removal is a separate revertable commit
- /cve-fix:
  detect.sh -> fix-all.sh -> review.sh + review-prompt.md
  - Runs across multiple repos and branches in parallel

### Slide 21: Design Pattern — Proper Plans

- Effort into quality plans is massively productive
- (all the well-known relevant expressions)
- More human effort collaborating on plans

### Slide 22: Proper Plans — Examples

- SEP-0031: modernize-enhancements -> enhancements/pull/267
- SEP-0032: cve-fix-refactoring -> shipyard/pull/2383
- configure-downstream.sh: one command generates 49 files across
  3 commits — only possible because extensive planning went into
  the Konflux onboarding design

---

## Part 5: Wrap Up

### Slide 23: Lessons Learned

- Inline skills become unmaintainable at ~1,000 lines — extract to
  standalone scripts with thin skill wrappers
- `set -e` is dangerous in verification code — silently swallows
  diagnostics before error-handling can run. Use `&&`/`||` to capture
  exit codes explicitly. (Hit this bug twice: PRs #52, #57)
- Real releases are the only reliable test — 0.22.1 and 0.23.1
  exposed bugs no amount of upfront design predicted
- Human-in-the-loop at danger points — skills stop before push/apply
  for user review, default to safest option
- [Note: this slide adds authenticity. Every pattern has a cost, and
  the lessons came from real pain, not theory.]

### Slide 24: Demo End

- Check on /cve-fix skill results
- [Note: the payoff — the skill has been running throughout the talk.
  Show the results live.]

### Slide 25: What's Next (optional)

- Autorelease conductor (Issue #64): a meta-skill that orchestrates
  all other skills
- Key insight: stateless re-evaluation, not a state machine
- No loop tracking, no history, no state — re-check structured state
  each time and pick the next incomplete step
- "The quality bar shifts entirely to the state checker's
  thoroughness"
- [Note: a strong forward-looking closer. The conductor design emerged
  from the experience of building and using 15 individual skills —
  it's the natural next step of the patterns in this talk.]

---

## Possible Additions

- **Live demo of release-status.sh** — would concretize "Design the
  Context" if there's time for a second demo moment
- **The graduation path visualized** — a single slide showing the
  trajectory of multiple skills from Phase 0 to Phase 3 over time
- **Audience interaction** — "what processes in your work could be
  written-down-as-automation?" after the New Domains slide
- **Plugin distribution** — the repo is a Claude Code plugin,
  installable via marketplace. Shows skills as distributable packages.
- **Timing data** (from PR #48) — Y-stream release: 17-50 hours,
  Z-stream: 13-42 hours. Calendar time: 1-3 weeks including QE.
  Critical path: QE approval (3-14 days external dependency).
