# Slides: Skill Design Patterns

<!-- Copy-paste ready. Cyan items = hyperlinks in GSlides. -->
<!-- --- separates slides. Title line = slide title. -->

---

Skill Design Patterns

---

What's a Skill & Why It Matters

- "Skill" -> "Claude skill"
- Claude interactive to create automation that runs Claude
  non-interactively
- ~15 skills for release management, goal: handoff-ready
- All early lessons, would love to hear feedback from others

---

What a Skill Ecosystem Looks Like

- 15 skills | 27 make targets | 28 scripts (9,772 lines)
- 22 workflow docs (context for both humans and agents)
- 7+ repos, 20-step lifecycle, previously entirely manual

---

Demo Start

- Start a skill running in background

---

Core Lesson: New Domains of Automation

- Custom, complex automation that would not have been practical
  before is now possible to produce quickly and maintain
- Still good to build on shared, quality tools
- Processes should be Written-Down-as-Automation
- Shared skills and custom skills are both valuable

---

Core Lesson: Design Around Indeterminism

- Extract as much work to deterministic logic as possible
- Treat agent call like a function call for "fuzzy" logic that
  would be overly complex to automate with traditional automation

---

Core Lesson: Design the Context (more not always better)

- In your Claude interactive sessions, more context is typically
  better
- In Claude instances that are meant to solve a narrower problem
  repeatedly, unnecessary context leads to unwanted inconsistencies

---

Design Pattern: Pseudocode as Prototype

- Skills can start as English descriptions of processes in Markdown,
  but they can run like code
- Recommended way to get started

---

Pseudocode as Prototype Phase 0: Mostly md

- [submariner/.agents/workflows/cve-fix.md](https://github.com/submariner-io/submariner/blob/devel/.agents/workflows/cve-fix.md)
- [submariner-release-management/.agents/workflows/scan-cves.md](https://github.com/stolostron/submariner-release-management/blob/main/.agents/workflows/scan-cves.md)

---

Pseudocode as Prototype Phase 1: Mixed md/sh

- [skills/cve-fix/SKILL.md](https://github.com/submariner-io/shipyard/pull/2383/files)

---

Pseudocode as Prototype Phase 2: Mostly sh

- /add-release-notes
  - [scripts/release-notes/auto-apply.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/auto-apply.sh)
- /cve-fix
  - WIP PR: [shipyard/pull/2383](https://github.com/submariner-io/shipyard/pull/2383)

---

Pseudocode as Prototype Phase 3: All sh, optional agent

- Wrapped by both make/agent
- [scripts/create-component-release.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/create-component-release.sh)
  - [skills/create-component-release/SKILL.md](https://github.com/stolostron/submariner-release-management/blob/main/skills/create-component-release/SKILL.md)
  - [Makefile#L93](https://github.com/stolostron/submariner-release-management/blob/main/Makefile#L93)
- [scripts/rpm-lockfile-update.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/rpm-lockfile-update.sh)
  - [skills/rpm-lockfile-update/SKILL.md](https://github.com/stolostron/submariner-release-management/blob/main/skills/rpm-lockfile-update/SKILL.md)
  - [Makefile#L101](https://github.com/stolostron/submariner-release-management/blob/main/Makefile#L101)
- (many more end in this state)

---

Design Pattern: Pulse-Agnostic Docs

- Create context for agents that's also docs for humans
- Massively increases productivity of docs
- Example: /context
  - (next slide)

---

Example: Pulse-Agnostic Docs

- [submariner-release-management/.agents/workflows](https://github.com/stolostron/submariner-release-management/tree/main/.agents/workflows)
- [submariner-operator/.agents/workflows](https://github.com/submariner-io/submariner-operator/tree/devel/.agents/workflows)

---

Design Pattern: Small World, Many Agents

- Craft data to create context for agents
- Invoke many agents in parallel, each focused on a discrete problem

---

Example: Small World, Many Agents

- /add-release-notes
  - [scripts/release-notes/collect.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/collect.sh)
  - [scripts/release-notes/prepare.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/prepare.sh)
  - [scripts/release-notes/review.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/review.sh)
  - [scripts/release-notes/review-prompt.md](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/review-prompt.md)
- /cve-fix
  - [scripts/cve/detect.sh](https://github.com/submariner-io/shipyard/pull/2383)
  - [scripts/cve/fix-all.sh](https://github.com/submariner-io/shipyard/pull/2383)
  - [scripts/cve/review.sh](https://github.com/submariner-io/shipyard/pull/2383)
  - [scripts/cve/review-prompt.md](https://github.com/submariner-io/shipyard/pull/2383)

---

Design Pattern: Proper Plans

- Plans for agents deserve more rigor, not less
- Ambiguity a human navigates = failure mode for agents
- More human effort collaborating on plans before agent execution

---

Example: Proper Plans

- [seps/SEP-0031-modernize-enhancements.md](https://github.com/submariner-io/enhancements/blob/devel/seps/SEP-0031-modernize-enhancements.md)
  - Created [enhancements/pull/267](https://github.com/submariner-io/enhancements/pull/267)
- [seps/SEP-0032-cve-fix-refactoring.md](https://github.com/submariner-io/enhancements/pull/268)
  - Created [shipyard/pull/2383](https://github.com/submariner-io/shipyard/pull/2383)

---

Lessons Learned

- Inline skills unmaintainable at ~1,000 lines -> extract to scripts
- set -e silently swallows errors -> use &&/||
- Real usage is the only reliable test
- Human-in-the-loop at danger points

---

Demo End

- Check on skill results

---

What's Next

- Conductor: meta-skill, stateless re-evaluation, no state machine
- "Quality bar shifts to the state checker's thoroughness"
- Open question: hosting with many secrets?
- Open question: top-level UI? (Slack bot, MCP, CLI)
