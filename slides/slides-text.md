# Slides: Skill Design Patterns

<!-- Copy-paste ready. Cyan items = hyperlinks in GSlides. -->
<!-- --- separates slides. Title line = slide title. -->

---

Skill Design Patterns

---

Level-Setting

- "Skill" -> "Claude skill"
- Claude interactive -> automation that runs Claude non-interactively
- ~20 skills for release management, goal: handoff-ready
- All early lessons, would love to hear feedback from others

---

What a Skill Ecosystem Looks Like

- ~20 skills | 27 make targets | 28 scripts (9,772 lines)
- 22 workflow docs (context for both humans and agents)
- 7+ repos, 20-step lifecycle, previously entirely manual

---

Demo Start

- Start a skill running in background

---

Core Lesson: New Domains of Automation

- Complex automation that wasn't practical before — now feasible
- Still build on shared, quality tools
- Processes should be Written-Down-as-Automation
- Shared skills and custom skills both valuable

---

Core Lesson: Design Around Indeterminism

- Extract as much work to deterministic logic as possible
- Agent call = function call for "fuzzy" logic

---

Core Lesson: Design the Context (more not always better)

- Interactive: more context is typically better
- Non-interactive: unnecessary context causes inconsistencies

---

Design Pattern: Pseudocode as Prototype

- Skills start as English in Markdown, but run like code
- Recommended way to get started
- Four phases — most skills converge at Phase 3

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

- Script runs standalone — three entry points:
  - Script (CI) | Make (humans) | Skill (Claude)
- [scripts/create-component-release.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/create-component-release.sh)
  - [SKILL.md](https://github.com/stolostron/submariner-release-management/blob/main/skills/create-component-release/SKILL.md)
  - [Makefile](https://github.com/stolostron/submariner-release-management/blob/main/Makefile#L93)
- [scripts/rpm-lockfile-update.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/rpm-lockfile-update.sh)
  - [SKILL.md](https://github.com/stolostron/submariner-release-management/blob/main/skills/rpm-lockfile-update/SKILL.md)
  - [Makefile](https://github.com/stolostron/submariner-release-management/blob/main/Makefile#L101)
- (many more end in this state)

---

Design Pattern: Pulse-Agnostic Docs

- Context for agents that's also docs for humans
- Write once, serve both audiences
- Example: /context
  - (next slide)

---

Example: Pulse-Agnostic Docs

- When | Process | Done When
- Same doc: trigger/precondition, steps/instructions,
  checklist/verification
- [submariner-release-management/.agents/workflows](https://github.com/stolostron/submariner-release-management/tree/main/.agents/workflows)
- [submariner-operator/.agents/workflows](https://github.com/submariner-io/submariner-operator/tree/devel/.agents/workflows)

---

Design Pattern: Small World, Many Agents

- Pre-fetch evidence deterministically, create focused context
- Many agents in parallel, each on a discrete problem

---

Example: Small World, Many Agents

- /add-release-notes
  - [collect.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/collect.sh)
    -> [prepare.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/prepare.sh)
    -> [review.sh](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/review.sh)
    \+ [review-prompt.md](https://github.com/stolostron/submariner-release-management/blob/main/scripts/release-notes/review-prompt.md)
- /cve-fix
  - detect -> fix-all -> review + prompt
    ([PR](https://github.com/submariner-io/shipyard/pull/2383))
- Agent evaluates, never searches — per-unit revertable commits

---

Design Pattern: Proper Plans

- Plans for agents deserve more rigor, not less
- Ambiguity a human navigates = failure mode for agents

---

Example: Proper Plans

- [SEP-0031](https://github.com/submariner-io/enhancements/blob/devel/seps/SEP-0031-modernize-enhancements.md)
  -> [enhancements/pull/267](https://github.com/submariner-io/enhancements/pull/267)
- [SEP-0032](https://github.com/submariner-io/enhancements/pull/268)
  -> [shipyard/pull/2383](https://github.com/submariner-io/shipyard/pull/2383)
- Enhancement proposals as agent-consumable specs
- configure-downstream.sh: 49 files, 3 commits

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
