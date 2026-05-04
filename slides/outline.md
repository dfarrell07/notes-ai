---
date: 2026-05-04T18:00:00-04:00
topic: slides
tags: [outline, slide-deck, talk-prep, skill-design-patterns]
---

# Slide Outline: Skill Design Patterns

General-audience talk about patterns for building Claude skills and
agent automation. Examples drawn from release management automation
(~15 skills) but patterns are universally applicable.

<!-- Each slide: title, bullets, [notes] for speaker context -->

---

## Part 1: Introduction

### Slide 1: Title

- Skill Design Patterns
- [Subtitle TBD — options: "Lessons from building agent automation",
  "What I learned building 15 skills", or just the title alone]

### Slide 2: What's a Skill & Why It Matters

- "Skill" = Claude skill
- Use Claude interactively to create automation that runs Claude
  non-interactively
- Built ~15 skills for release management — building, testing,
  securing, and shipping software releases
- Patterns emerged that apply to any domain
- All early lessons, would love to hear feedback from others

### Slide 3: What a Skill Ecosystem Looks Like

- Skills: 15 slash commands
- Make: 27 targets
- Scripts: 28 scripts, 9,772 lines
- Docs: 22 workflow runbooks
- Configs: 88 release YAMLs
- Spanning 7+ repos with shared agent infrastructure
- Covering a 20-step release lifecycle that was previously
  entirely manual

### Slide 4: Demo Start

- Starting one of the skills in the background
- We'll check results at end of talk
- [Note: use /cve-fix — it runs long enough and illustrates multiple
  patterns. Don't dwell on what it does; the point is that it's
  running autonomously]

---

## Part 2: Core Lessons

### Slide 5: Core Lesson: New Domains of Automation

- Custom, complex automation that would not have been practical before
  is now possible to produce quickly and maintain
- Still good to build on shared, quality tools
- Processes should be Written-Down-as-Automation
- Shared skills and custom skills are both valuable

### Slide 6: Core Lesson: Design Around Indeterminism

- Extract as much work to deterministic logic as possible
- Treat agent call like a function call for "fuzzy" logic that would
  be overly complex to automate traditionally
- The deterministic part collects data and attempts fixes;
  the agent part reviews, judges, and handles ambiguity
- [Example: a script tries every CVE fix deterministically, exits
  with "needs review" status code for ambiguous cases, then an agent
  evaluates with pre-fetched evidence — agent judges, never searches]
- [Example: a 536-line script handles SHA extraction/validation
  deterministically — agent is only invoked for release notes review]

### Slide 7: Core Lesson: Design the Context

- In your Claude interactive sessions, more context is typically better
- In Claude instances that are meant to solve a narrower problem
  repeatedly, unnecessary context leads to unwanted inconsistencies
- [Example: a 1,474-line status dashboard crafts focused context for
  both humans and agents. The agent doesn't see the whole system —
  it sees exactly what it needs to evaluate the current state.]

---

## Part 3: Design Patterns

Each pattern has a concept slide followed by an example slide.

### Slide 8: Design Pattern: Pseudocode as Prototype

- Skills can start as English descriptions of processes in Markdown,
  but they can run like code
- Recommended way to get started
- Skills naturally evolve through four phases
- Most skills converge at Phase 3

### Slide 9: Phase 0 — Mostly md

- Write the process in English as a markdown document
- Claude reads it and follows it like a runbook
- Low investment, immediate value
- [Example: a workflow doc describing CVE scanning steps, a runbook
  for configuring downstream builds]

### Slide 10: Phase 1 — Mixed md/sh

- Embed bash snippets in the skill definition
- Markdown provides structure and decision logic;
  bash provides precision for deterministic steps
- Works to get started but becomes hard to maintain at ~1,000 lines
- [Example: a skill definition with 1,484 lines of inline bash —
  this worked but was the forcing function for evolving further]

### Slide 11: Phase 2 — Mostly sh

- The scripts do the heavy lifting
- Skill wraps one or more scripts with markdown framing
- Scripts start to become independently useful
- [Example: release notes script called by skill, CVE fix scripts
  called by skill wrapper]

### Slide 12: Phase 3 — All sh, optional agent

- The script IS the logic — it runs standalone
- Three entry points to the same code:
  - Script directly (CI pipelines, other scripts)
  - Make / task runner (humans at the command line)
  - Skill wrapper (Claude — thin wrapper that exec's the script)
- Agent adds judgment only where deterministic logic can't
- Most skills naturally converge here
- [Note: this pattern emerged from real pain. Four skills were
  refactored from Phase 1 to Phase 3 in a single day once the
  pattern was clear. Inline skills couldn't be tested independently,
  used from CI, or debugged outside Claude.]

### Slide 13: Design Pattern: Pulse-Agnostic Docs

- Create context for agents that's also docs for humans
- Write once, serve both audiences
- Massively increases productivity of docs

### Slide 14: Example: Pulse-Agnostic Docs

- Use a consistent document structure that works for both:
  - "When" = trigger condition for a human, precondition for an agent
  - "Process" = steps for a human, instructions for an agent
  - "Done When" = checklist for a human, verification command
    for an agent
- A top-level doc (like CLAUDE.md) can reference these as steps in
  a workflow — the agent follows the references, the human reads
  the same docs as runbooks
- [Example: 22 workflow docs in the orchestration repo, plus workflow
  docs in 5 upstream repos — same format everywhere. Same artifacts,
  two audiences, zero duplication.]

### Slide 15: Design Pattern: Small World, Many Agents

- Craft data to create focused context for agents
- Invoke many agents in parallel, each focused on a discrete problem

### Slide 16: Example: Small World, Many Agents

- Pattern: deterministic scripts collect and prepare data, then
  focused agents review with purpose-built prompts
- Each agent gets pre-fetched evidence — it evaluates, never searches
- Per-unit commits for reviewability and revertability
- [Example: release notes — one agent per Jira issue, each gets
  pre-fetched data from Jira, bug tracker, GitHub PRs, and git logs.
  Agent only decides keep/remove; each removal is a separate commit.]
- [Example: CVE fix — one agent per repo, each given scan results,
  fix summary, and package location data. Agent reviews the
  deterministic fixes and handles remaining ambiguous cases.]

### Slide 17: Design Pattern: Proper Plans

- Effort into quality plans is massively productive
- (all the well-known relevant expressions)
- More human effort collaborating on plans before agent execution

### Slide 18: Example: Proper Plans

- Written design docs before implementation pay off enormously
  when agent execution depends on getting the structure right
- Good plans enable complex automation: one command generating
  dozens of files across multiple commits, only possible because
  the design was fully worked out first
- [Example: a setup script generates 49 files across 3 commits —
  the planning phase was longer than the implementation]
- [Example: design docs (SEPs) that became PRs — the plan was
  the implementation spec]

---

## Part 4: Wrap Up

### Slide 19: Lessons Learned

- Inline skills become unmaintainable at ~1,000 lines — extract to
  standalone scripts with thin skill wrappers
- `set -e` is dangerous in scripts with error handling — silently
  swallows diagnostics. Use `&&`/`||` to capture exit codes.
- Real usage is the only reliable test — no amount of upfront design
  predicts what breaks when the system runs for real
- Human-in-the-loop at danger points — skills should stop before
  destructive actions for user review
- [Note: these lessons add authenticity. The audience wants to know
  what not to do, not just what worked.]

### Slide 20: Demo End

- Check on skill results
- [Note: the payoff — the skill has been running throughout the talk.
  Show the results live.]

### Slide 21: What's Next

- Conductor pattern: a meta-skill that orchestrates other skills
- Key insight: stateless re-evaluation, not a state machine
- No loop tracking, no history — re-check structured state each time
  and pick the next incomplete step
- "The quality bar shifts entirely to the state checker's thoroughness"
- Open question: how to host skill automation that needs many secrets?
  (cluster credentials, registry auth, entitlements, GitHub/Jira
  tokens, signing keys) — today runs on a developer laptop, but
  that doesn't scale
- [Note: the secrets/hosting question is a real unsolved problem that
  the audience may have insights on. Good way to open discussion.]

---

## Possible Additions

- **Live demo of a status dashboard** — concretize "Design the
  Context" if time allows a second demo moment
- **Audience interaction** — "what processes in your work could be
  written-down-as-automation?" after the New Domains slide
- **Timing data** — Y-stream: 17-50 hours, Z-stream: 13-42 hours,
  if audience cares about effort/ROI
- **Plugin distribution** — skills as distributable packages,
  installable via marketplace
