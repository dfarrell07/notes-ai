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
- Will use /cve-fix as one example to talk about bigger ideas

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
- [Example to mention: bundle-image-update.sh — 536 lines of
  deterministic SHA extraction/validation, agent only invoked for
  fuzzy parts like release notes review]

### Slide 7: Core Lesson — Design the Context

- In your Claude interactive sessions, more context is typically better
- In Claude instances that are meant to solve a narrower problem
  repeatedly, unnecessary context leads to unwanted inconsistencies
- [Example to mention: release-status.sh (1,474 lines) — crafts
  focused context for both humans and agents to understand release
  state]

---

## Part 3: What We Built

Context for the design patterns that follow. Shows the scale and
complexity of the system the patterns were applied to.

### Slide 8: The Release Automation Landscape

- Submariner release lifecycle: 20 steps, previously manual
- 27 Make targets, 28 scripts (9,776 lines of automation)
- 15 Claude Code skills as slash commands
- 22 agent workflow docs as detailed runbooks
- 88 release YAML configs across versions 0.20-0.24
- Eliminated the midstream repo entirely
- [Note: these numbers give weight to the patterns — this isn't a
  toy example, it's a production system]

### Slide 9: Snapshot Pipeline Overview (optional)

- 5 upstream repos (operator, submariner, lighthouse, shipyard, subctl)
- Konflux builds 8 components -> snapshots with image SHAs
- bundle-image-update -> component releases -> FBC catalog
- FBC catalog covers 6 OCP versions (4.16-4.21)
- QE gets catalog URLs via `make get-fbc-urls`
- End-to-end SHA traceability across 10 sources
- [Note: include this slide if audience benefits from understanding
  the pipeline. Skip if time is tight — the point is complexity, and
  slide 8 already conveys that]

---

## Part 4: Design Patterns

### Slide 10: Design Pattern — Pseudocode as Prototype

- Skills can start as English descriptions of processes in Markdown,
  but they can run like code
- Recommended way to get started
- Four phases of evolution (following slides)

### Slide 11: Phase 0 — Mostly md

- submariner/.agents/workflows/cve-fix.md
- submariner-release-management/.agents/workflows/scan-cves.md

### Slide 12: Phase 1 — Mixed md/sh

- skills/cve-fix/SKILL.md

### Slide 13: Phase 2 — Mostly sh

- /add-release-notes -> scripts/release-notes/auto-apply.sh
- /cve-fix -> WIP PR: shipyard/pull/2383

### Slide 14: Phase 3 — All sh, optional agent

- Wrapped by both make and agent
- scripts/create-component-release.sh +
  skills/create-component-release/SKILL.md +
  submariner-release-management/Makefile#L18
- scripts/rpm-lockfile-update.sh +
  skills/rpm-lockfile-update/SKILL.md +
  submariner-release-management/Makefile#L23
- (many more end in this state)
- [Note: emphasize that Phase 3 is where most skills converge — the
  natural end state is deterministic shell, optionally wrapped by an
  agent for the fuzzy parts. 15 skills at various phases, most here.]

### Slide 15: Design Pattern — Pulse-Agnostic Docs

- Create context for agents that's also docs for humans
- Massively increases productivity of docs
- Example: /context

### Slide 16: Pulse-Agnostic Docs — Examples

- submariner-release-management/.agents/workflows
- submariner-operator/.agents/workflows

### Slide 17: Pulse-Agnostic Docs — At Scale

- 22 workflow docs serve as both agent context and team runbooks
- CLAUDE.md encodes the full 20-step release workflow
  (Y-stream and Z-stream paths)
- The documentation IS the automation's interface — same artifact
  drives both human understanding and agent execution
- [Note: this is the most compelling "at scale" story. The short talk
  showed the pattern; this slide shows what happens when you commit
  to it across an entire release lifecycle]

### Slide 18: Design Pattern — Small World, Many Agents

- Craft data to create context for agents
- Invoke many agents in parallel, each focused on a discrete problem

### Slide 19: Small World, Many Agents — Examples

- /add-release-notes:
  collect.sh -> prepare.sh -> review.sh + review-prompt.md
- /cve-fix:
  detect.sh -> fix-all.sh -> review.sh + review-prompt.md
- [Note: both follow the same shape — deterministic scripts collect
  and prepare data, then a focused agent reviews with a purpose-built
  prompt. /cve-fix also runs across multiple repos and branches in
  parallel]

### Slide 20: Design Pattern — Proper Plans

- Effort into quality plans is massively productive
- (all the well-known relevant expressions)
- More human effort collaborating on plans

### Slide 21: Proper Plans — Examples

- SEP-0031: modernize-enhancements -> enhancements/pull/267
- SEP-0032: cve-fix-refactoring -> shipyard/pull/2383
- [Also worth mentioning: configure-downstream.sh — one command
  generates 49 files across 3 commits, only possible because
  extensive planning went into the Konflux onboarding design]

---

## Part 5: Wrap Up

### Slide 22: Demo End

- Check on /cve-fix skill results
- [Note: the payoff — the skill has been running throughout the talk.
  Show the results live]

### Slide 23: Key Takeaways (optional)

- New domains of automation are now accessible — write processes down
  as automation
- Design around indeterminism — deterministic shell first, agent for
  fuzzy logic
- Start with pseudocode in markdown, evolve toward shell — most skills
  end at Phase 3
- Write docs that serve both humans and agents — the investment
  multiplies
- [Note: include if talk format benefits from a summary. The demo end
  may be a stronger closer on its own]

---

## Possible Additions

Things not yet in the outline that could be added:

- **Lessons learned / what didn't work** — would add authenticity,
  the user mentioned this as a possibility
- **Live demo of release-status.sh** — would concretize "Design the
  Context" if there's time for a second demo moment
- **The graduation path visualized** — a single slide showing the
  trajectory of multiple skills from Phase 0 to Phase 3 over time
- **Audience interaction** — "what processes in your work could be
  written-down-as-automation?" after the New Domains slide
