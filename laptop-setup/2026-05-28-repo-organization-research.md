# Repo Organization Research

Research on how to organize ~80 git repos across multiple GitHub orgs,
internal GitLab, and personal projects on a developer workstation.

## The Winning Layout: URL-Mirroring (host/org/repo)

The strongest consensus across developer blogs, community discussions,
and tool ecosystems is the **URL-mirroring** layout:

```
~/src/                              # or ~/code/, ~/workspace/, ~/projects/
  github.com/
    openshift/
      ovn-kubernetes/
      installer/
    submariner-io/
      submariner/
      lighthouse/
    personal-user/
      dotfiles/
      side-project/
  gitlab.internal.company.com/
    team/
      internal-tool/
```

### Why this wins

- **No naming collisions.** If two orgs have a repo called `utils`, they
  live at different paths. No renaming needed.
- **Easy to find anything.** The path tells you where it came from. No
  guessing whether `ovn-kubernetes` is your fork or upstream.
- **Forks are obvious.** `github.com/dfarrell07/ovn-kubernetes` vs
  `github.com/openshift/ovn-kubernetes` coexist naturally.
- **Scales indefinitely.** Works the same at 10 repos or 200.
- **Git conditional includes work perfectly.** You can set work email
  for `github.com/openshift/` and personal email for
  `github.com/personal-user/` automatically.
- **Tools support it natively.** ghq, smart-clone, and custom shell
  functions all assume this layout.
- **Go heritage.** The old GOPATH convention forced this layout. With Go
  modules you can put repos anywhere, but many Go developers kept the
  structure because it works well. Multiple developers credit GOPATH as
  the reason they adopted it and say it was the "only system that stuck."

### What people actually call the root directory

Most popular names from community surveys: `~/src`, `~/code`, `~/dev`,
`~/workspace`, `~/projects`, `~/git`. Pick one and stick with it. The
name does not matter nearly as much as the structure underneath.

## Comparison of All Patterns

### 1. GOPATH-style: ~/go/src/host/org/repo/

The original Go workspace convention. Identical structure to
URL-mirroring but rooted in `~/go/src/`. With Go modules (standard since
Go 1.14), there is zero reason to use `~/go/src/` specifically. Go
modules let you put code anywhere. The GOPATH directory is now only used
for the module cache (`~/go/pkg/mod/`) and installed binaries
(`~/go/bin/`). Using `~/go/src/` for non-Go repos would be confusing.

**Verdict:** Adopt the structure, not the path.

### 2. Flat: ~/repos/repo/

Simple. Works fine at 10 repos. Falls apart at 80. You get naming
collisions (two orgs with a repo called `api`). You lose context about
where repos came from. No natural grouping for conditional git config.

**Verdict:** Outgrown by anyone with multi-org work.

### 3. By org: ~/src/org/repo/

Better than flat. Groups repos by owner. But loses the host dimension.
If you have repos on both github.com and internal GitLab, repos from
different hosts with the same org name collide. Also harder to set up
conditional git config per host.

**Verdict:** Close to URL-mirroring but missing the host level. Add it.

### 4. By project/workspace: ~/submariner/, ~/ovnk/

Groups related repos together regardless of org. This is great for
focused work on a project that spans multiple orgs. But it creates
ambiguity: if `ovn-kubernetes` is relevant to both the `ovnk` workspace
and the `openshift` org, where does it live? You end up with symlinks or
duplicates.

**Verdict:** Useful as a mental model, but as a directory layout it
conflicts with URL-mirroring. Better to use editor workspaces (VSCode
multi-root workspaces), tmux sessions, or shell aliases to group related
repos logically without duplicating them on disk. The filesystem handles
ownership; your workflow tools handle project grouping.

### 5. ghq-style: ~/ghq/host/org/repo/

Same structure as URL-mirroring, but managed by the `ghq` tool. See
the ghq section below.

### 6. XDG: ~/Projects/ or ~/Developer/

macOS convention (`~/Developer/`). Just a different root name. The
structure underneath matters more than what you call the top level.

## ghq: Worth Using?

[ghq](https://github.com/x-motemen/ghq) is a Go CLI tool that
automates the URL-mirroring layout. ~3,300 GitHub stars, actively
maintained (latest release March 2025), MIT licensed.

### What it does

- `ghq get https://github.com/openshift/ovn-kubernetes` clones to
  `~/ghq/github.com/openshift/ovn-kubernetes` automatically.
- `ghq list` shows all managed repos.
- `ghq list | fzf` gives fuzzy search across all repos.
- Configurable root directory via `git config --global ghq.root ~/src`.
- Supports shallow clones, partial clones, bare clones.
- Handles GitHub, GitLab, Bitbucket, any Git host.

### The modern combo: ghq + gwq + fzf

A 2025 article describes a workflow that has gained traction, especially
with AI coding agents:

- **ghq** manages cloned repos under a single root.
- **[gwq](https://github.com/d-kuro/gwq)** manages git worktrees in the
  same root, naming them `repo=branch` (e.g.,
  `github.com/org/app=feature-auth`).
- **fzf** provides fuzzy search across both repos and worktrees.

The result: `~/ghq/github.com/org/app` is the main clone,
`~/ghq/github.com/org/app=feature-auth` is a worktree, and a single
`fzf` command jumps to either. This is useful for running multiple
Claude Code sessions in parallel.

### Should you adopt ghq?

**Arguments for:**
- Enforces the URL-mirroring layout automatically.
- `ghq get` is shorter than manually creating directories and cloning.
- `ghq list` makes discovery instant.
- Pairs well with fzf for fast navigation.
- The gwq integration is relevant for worktree-heavy workflows.

**Arguments against:**
- You can get 90% of the benefit with a shell function. A `git-get`
  alias that parses the URL, creates directories, and runs `git clone`
  is ~10 lines of shell.
- ghq is another tool to install and maintain.
- If you already have 80 repos in a different layout, migration is
  manual work (ghq does not auto-migrate existing repos, though you can
  `ghq get` each one fresh).
- Some developers find the tool unnecessary overhead for something a
  shell alias handles fine.

**Practical recommendation:** If starting fresh (new laptop), ghq is
worth trying. If migrating 80 existing repos, a shell function that
implements the same convention is simpler. The convention matters more
than the tool.

## Tools for Managing 80+ Repos at Scale

### Batch operations (status, fetch, pull across all repos)

| Tool | Language | What it does |
|------|----------|-------------|
| [gita](https://github.com/nosarthur/gita) | Python | Color-coded status for all repos, async fetch/pull, group repos into named sets, pass through any git command |
| [mgitstatus](https://github.com/fboender/multi-git-status) | Shell | Simple status overview of all repos in a directory. One script, easy install. `mgitstatus -f` does fetch first |
| [myrepos (mr)](https://myrepos.branchable.com/) | Perl | Supports Git, SVN, Hg, CVS. Registers repos in `~/.mrconfig`. Can retry failed operations when coming back online |
| [mani](https://github.com/alajmo/mani) | Go | YAML config file defines repos and custom commands. Good for team onboarding: clone the config, run `mani sync` to get all repos |
| [mu-repo](https://github.com/fabioz/mu-repo) | Python | Register repos once, run git commands across all. Has `mu sh` for non-git commands and `mu open-url` for batch PR creation |

### Navigation (jumping between repos quickly)

| Tool | What it does |
|------|-------------|
| [fzf](https://github.com/junegunn/fzf) | Fuzzy finder. Combine with `find` or `ghq list` to jump to any repo instantly |
| [z](https://github.com/rupa/z) / [zoxide](https://github.com/ajeetdsouza/zoxide) | Learns your navigation patterns, `z ovn` jumps to wherever you go most often |
| Custom `gt` function | `find ~/src -mindepth 3 -maxdepth 3 -type d \| fzf` piped into `cd`. Simple and effective |

### Practical recommendation for 80 repos

- **gita** is the best fit for a developer who wants quick status
  overview and batch fetch/pull. It is Python (already in your stack),
  supports grouping repos by project, and handles async operations.
- **fzf + a shell function** handles navigation. 10 lines of shell,
  zero maintenance.
- You do not need a heavy tool for 80 repos. The combination of a
  consistent directory layout + gita + fzf covers the common needs.

## Separating Work vs Personal Repos

### Directory-level separation

The URL-mirroring layout handles this automatically:

```
~/src/
  github.com/
    openshift/        # Work
    submariner-io/    # Work
    personal-user/    # Personal
  gitlab.company.com/
    team/             # Work (different host, clearly separated)
```

### Git conditional includes

Git 2.13+ supports `includeIf` directives in `~/.gitconfig` that
automatically apply different settings based on repo location:

```gitconfig
[user]
    name = Daniel Farrell
    email = personal@example.com

# Work repos on github.com (specific orgs)
[includeIf "gitdir:~/src/github.com/openshift/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/src/github.com/submariner-io/"]
    path = ~/.gitconfig-work

# Work repos on internal GitLab
[includeIf "gitdir:~/src/gitlab.company.com/"]
    path = ~/.gitconfig-work
```

Where `~/.gitconfig-work` contains:

```gitconfig
[user]
    email = dfarrell@redhat.com
```

Important details:
- Trailing slash on the path is required for recursive matching.
- `includeIf` directives must be at the bottom of `.gitconfig` (last
  setting wins).
- The `hasconfig:remote.*.url` variant can match based on remote URL
  instead of directory path (Git 2.36+), useful if you do not want to
  rely on directory structure.
- Verify with `cd ~/src/github.com/openshift/some-repo && git config user.email`.

## Go Modules and Repo Location

With Go modules (standard since Go 1.14, used by 87% of Go developers
per surveys):

- Repos can live anywhere on the filesystem. GOPATH is irrelevant for
  source code location.
- `go.mod` at the repo root defines the module path. The filesystem
  path does not need to match the module path.
- `GOPATH` is still used for the module cache (`~/go/pkg/mod/`) and
  installed binaries (`~/go/bin/`). Leave these defaults alone.
- For multi-module local development, use `go.work` files (Go 1.18+)
  to define workspaces. Add `go.work` to `.gitignore` since it is
  machine-specific.
- `replace` directives for local development go in `go.work`, not
  `go.mod`.

There is no reason to use `~/go/src/` for Go repos. Any directory
structure works. The URL-mirroring layout is popular among Go developers
precisely because GOPATH trained them to think that way, but the
filesystem path and the Go module path are now completely independent.

## Handling the Existing ~/ovnk/ Workspace Layout

The current pattern of `~/ovnk/` grouping related repos from different
orgs is a project-centric layout. It works well for focused work but
creates problems:

- Repos from `github.com/openshift/ovn-kubernetes` and
  `github.com/ovn-org/ovn-kubernetes` both go in `~/ovnk/` -- which
  org does each belong to?
- Git conditional includes are harder to set up because work and personal
  repos might be mixed in the same directory.
- If a repo is relevant to multiple projects, where does it go?

### Migration path

1. Adopt URL-mirroring as the canonical layout for all repos.
2. Use tmux sessions, shell aliases, or editor workspaces to create
   project-centric views without duplicating repos on disk:
   - A tmux session called `ovnk` that opens windows in the relevant
     repos.
   - A VSCode multi-root workspace file that groups the related repos.
   - A gita group called `ovnk` that lets you `gita pull ovnk` to
     update all related repos.
3. If you use the `go.work` file for multi-module development across
   related repos, you can place `go.work` at a convenient location and
   reference repos by their actual paths.

## Summary Recommendation

1. **Layout:** `~/src/host/org/repo/` (URL-mirroring). Simple, scalable,
   tool-friendly, solves naming collisions.
2. **Tooling:** `ghq` if starting fresh, shell function if migrating.
   Add `fzf` for navigation. Add `gita` for batch operations across
   repos.
3. **Work/personal separation:** Handled by directory structure +
   gitconfig conditional includes. No separate directory trees needed.
4. **Go repos:** Put them in the same layout as everything else. Go
   modules do not care where repos live on disk.
5. **Project grouping:** Use workflow tools (tmux, editor workspaces,
   gita groups) instead of filesystem duplication.

## Sources

- [How I organize git repos locally](https://blog.esc.sh/how-i-organize-git-repos-locally/) - Developer with ~69 repos using ~/git/host/org/repo + fzf
- [A sensible way to organize your repositories locally](https://medium.com/@pablosjv/a-sensible-way-to-organize-your-repositories-locally-965d17732510) - URL-mirroring with gitconfig conditional includes
- [How I organize my local development environment](https://www.boot.dev/blog/misc/how-i-organize-my-local-development-environment/) - ~/workspace/REMOTE/NAMESPACE/REPO, credits GOPATH as inspiration
- [My development directory structure](https://dev.to/httpjunkie/my-development-directory-structure-3p1g) - Go-style hierarchy for all languages
- [How do you organize development projects?](https://dev.to/andrewmcodes/how-do-you-organize-development-projects-on-your-computer-4dja) - Community discussion with multiple approaches
- [ghq - Remote repository management](https://github.com/x-motemen/ghq) - Tool that automates URL-mirroring layout
- [gwq - Git worktree manager](https://github.com/d-kuro/gwq) - Worktree management companion to ghq
- [ghq + gwq + fzf workflow for AI coding agents](https://zenn.dev/shunk031/articles/ghq-gwq-fzf-worktree?locale=en) - Modern workflow combining all three tools
- [Managing git repos with ghq](https://dev.to/ekeih/managing-your-git-repositories-with-ghq-3ffa) - Practical ghq experience report
- [gita - Manage many git repos](https://github.com/nosarthur/gita) - Batch operations across repos
- [mani - CLI tool for multiple repos](https://dev.to/alajmo/mani-a-cli-tool-to-manage-multiple-repositories-1eg) - YAML-configured multi-repo management
- [multi-git-status](https://github.com/fboender/multi-git-status) - Simple shell script for repo status overview
- [myrepos](https://myrepos.branchable.com/) - Multi-VCS repo management
- [mu-repo](https://github.com/fabioz/mu-repo) - Batch git commands across repos
- [Git conditional includes for work/personal](https://blog.danskingdom.com/Customize-Git-config-based-on-the-repository-path/) - Detailed setup guide
- [Separate work and personal git environments](https://medium.com/@sreejith_vu/how-to-separate-work-and-personal-git-environments-a359681b733a)
- [Multiple git configs](https://www.freecodecamp.org/news/how-to-handle-multiple-git-configurations-in-one-machine/)
- [Go workspace structure: from GOPATH to go.work](https://www.glukhov.org/post/2025/12/go-workplace-structure/)
- [Go modules: project setup without GOPATH](https://blog.francium.tech/go-modules-go-project-set-up-without-gopath-1ae601a4e868)
- [Organizing a Go module](https://go.dev/doc/modules/layout) - Official Go documentation
