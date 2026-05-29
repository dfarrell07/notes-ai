# Claude Code Task Queue: GitHub Issue-Based Design

Date: 2026-05-28

## Overview

A private GitHub repo acts as a task queue. The user creates issues from their
phone describing work to do. A systemd timer on the laptop polls for new issues,
dispatches Claude Code in headless mode, and opens PRs that the user reviews on
their phone.

## Architecture

```
Phone (GitHub Mobile)          Laptop
  |                              |
  |  create issue                |
  |  "repo: submariner-operator" |
  |  "Fix the flaky e2e test"   |
  |                              |
  |                              |  systemd timer fires (every 2 min)
  |                              |  poller.sh runs
  |                              |    gh issue list --repo task-queue --label queued
  |                              |    acquires lock
  |                              |    labels issue "processing"
  |                              |    resolves repo path
  |                              |    cd /path/to/target-repo
  |                              |    git checkout main && git pull
  |                              |    git checkout -b claude/42-fix-flaky-e2e
  |                              |    claude -p "..." --allowedTools ... --max-turns 50
  |                              |    git push, gh pr create
  |                              |    labels issue "done", closes it
  |                              |    comments with PR link
  |                              |
  |  phone notification: PR opened|
  |  review PR on phone          |
```

## Issue Format

The issue body uses a minimal structured format designed for phone typing:

```
repo: submariner-operator

Fix the flaky e2e test in pkg/join that times out
on slow CI machines. The timeout is 30s but should be 120s.
```

Rules:
- First line must be `repo: <short-name>`
- Blank line separator
- Everything after is the prompt passed to Claude
- Issue title is used for branch naming (slugified)

### Repo Map

The poller maintains a map of short names to absolute paths and GitHub
org/repo identifiers:

```bash
declare -A REPO_PATH=(
  [submariner-operator]="/home/dfarrell07/go/src/github.com/submariner-io/submariner-operator"
  [submariner]="/home/dfarrell07/go/src/github.com/submariner-io/submariner"
  [admiral]="/home/dfarrell07/go/src/github.com/submariner-io/admiral"
  [lighthouse]="/home/dfarrell07/go/src/github.com/submariner-io/lighthouse"
  [shipyard]="/home/dfarrell07/go/src/github.com/submariner-io/shipyard"
  [notes-ai]="/home/dfarrell07/notes-ai"
)

declare -A REPO_REMOTE=(
  [submariner-operator]="submariner-io/submariner-operator"
  [submariner]="submariner-io/submariner"
  [admiral]="submariner-io/admiral"
  [lighthouse]="submariner-io/lighthouse"
  [shipyard]="submariner-io/shipyard"
  [notes-ai]="dfarrell07/notes-ai"
)
```

If the `repo:` line is missing or the short name is not in the map, the poller
comments on the issue with an error and labels it `failed`.

## State Machine

Issues move through states tracked by GitHub labels:

```
(new issue created)
        |
        v
    [queued]  <-- default label, applied by issue template or poller
        |
        v
  [processing]  -- poller claims the issue
        |
       / \
      v   v
  [done] [failed]
```

- `queued`: Ready to be picked up. The poller queries for these.
- `processing`: Claude is actively working. Prevents re-pickup on next poll.
- `done`: PR opened successfully. Issue is also closed.
- `failed`: Claude errored. Issue stays open. User must triage.

The poller query: `gh issue list --repo TASK_QUEUE_REPO --label queued --json number,title,body --jq '.[]' --state open`

## Locking

Two layers prevent concurrent execution:

1. **Systemd**: The timer unit does not start a new instance if the previous
   is still running (default behavior with `Type=oneshot`).

2. **PID lock file**: Defense-in-depth for manual runs.

```bash
LOCKFILE="/run/user/$(id -u)/claude-queue.lock"

acquire_lock() {
  if [ -f "$LOCKFILE" ]; then
    local pid
    pid=$(cat "$LOCKFILE")
    if kill -0 "$pid" 2>/dev/null; then
      echo "Another instance running (PID $pid), exiting"
      exit 0
    fi
    echo "Removing stale lock (PID $pid)"
    rm -f "$LOCKFILE"
  fi
  echo $$ > "$LOCKFILE"
  trap 'rm -f "$LOCKFILE"' EXIT
}
```

Issues are processed sequentially, oldest first. If there are 3 queued issues,
one timer invocation processes all 3 in order.

## Error Handling

### Claude Failure

```bash
# Run Claude with timeout wrapper
timeout 1800 claude -p "$PROMPT" \
  --allowedTools "$ALLOWED_TOOLS" \
  --max-turns 50 \
  2>"$LOG_DIR/issue-${ISSUE_NUM}-stderr.log" \
  | tee "$LOG_DIR/issue-${ISSUE_NUM}-stdout.log"

EXIT_CODE=${PIPESTATUS[0]}

if [ $EXIT_CODE -ne 0 ]; then
  # Remove processing, add failed
  gh issue edit "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO" \
    --remove-label processing --add-label failed

  # Comment with failure details
  STDERR_TAIL=$(tail -100 "$LOG_DIR/issue-${ISSUE_NUM}-stderr.log")
  gh issue comment "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO" --body "$(cat <<EOF
## Task Failed

**Exit code**: $EXIT_CODE
**Branch**: \`$BRANCH_NAME\`
**Duration**: $DURATION seconds

<details>
<summary>Error output (last 100 lines)</summary>

\`\`\`
$STDERR_TAIL
\`\`\`
</details>

The branch has been pushed for inspection. To retry, remove the \`failed\`
label and add \`queued\`.
EOF
)"
  continue  # Move to next issue
fi
```

### Specific Failure Modes

| Failure | Detection | Response |
|---------|-----------|----------|
| Claude exits non-zero | Exit code check | Comment + `failed` label |
| Claude hangs | `timeout 1800` (30 min) | Kill, exit code 124, same error flow |
| Claude makes no changes | `git diff --quiet` after run | Comment "no changes made", `failed` label |
| `gh pr create` fails | Exit code check | Comment with error, branch is still pushed |
| Target repo doesn't exist | Path check before `cd` | Comment "repo path not found", `failed` label |
| Network down | `gh` commands fail | Lock released, issue stays `queued` (not yet labeled `processing`) |

### Retry

To retry a failed issue, the user (from phone):
1. Reads the failure comment
2. Removes `failed` label
3. Adds `queued` label
4. Optionally edits the issue body to refine the prompt

## Safety Controls

```bash
# Hard limits
MAX_TURNS=50              # Prevent infinite loops
MAX_BUDGET_USD=5.00       # Cost cap per issue
TIMEOUT_SECONDS=1800      # 30 minute wall clock limit

# Tool restrictions -- explicit allowlist
ALLOWED_TOOLS="Bash(git *),Bash(go *),Bash(make *),Bash(grep *),Bash(find *),Read,Edit,Write"

# What's NOT allowed:
# - No internet access (no curl, no wget)
# - No docker/podman
# - No rm -rf, no sudo
# - No git push (the poller script handles push, not Claude)
```

The `--allowedTools` flag is the primary safety mechanism. Claude can read,
edit, write files, run tests, and use git for diffing/status, but cannot push,
delete branches, or access the network.

Adjustments per repo type could be configured in the repo map:

```bash
declare -A REPO_ALLOWED_TOOLS=(
  [submariner-operator]="Bash(git *),Bash(go *),Bash(make *),Bash(golangci-lint *),Read,Edit,Write"
  [notes-ai]="Bash(git *),Read,Edit,Write"
)
```

## Branch Naming

Convention: `claude/<issue-number>-<slugified-title>`

```bash
slugify() {
  echo "$1" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-' | sed 's/^-//;s/-$//' | head -c 50
}

BRANCH_NAME="claude/${ISSUE_NUM}-$(slugify "$ISSUE_TITLE")"
```

Examples:
- Issue #42 "Fix flaky e2e test" -> `claude/42-fix-flaky-e2e-test`
- Issue #7 "Add retry logic to gateway" -> `claude/7-add-retry-logic-to-gateway`

## PR Body

```bash
gh pr create \
  --title "$ISSUE_TITLE" \
  --repo "$TARGET_REMOTE" \
  --body "$(cat <<EOF
## Summary

This PR was generated by Claude Code in response to a task queue issue.

**Task**: ${TASK_QUEUE_REPO}#${ISSUE_NUM}
**Prompt**: $(echo "$PROMPT" | head -20)

## Changes

$(claude -p "Summarize the changes in this git diff concisely" --max-turns 3 <<< "$(git diff main...HEAD)")

## Review Checklist

- [ ] Changes match the requested task
- [ ] Tests pass
- [ ] No unintended modifications
EOF
)"
```

Note: The PR summary could also be captured from Claude's original output
rather than running a second Claude invocation. The poller could capture
Claude's final output and use it directly.

## Poller Script

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- Configuration ---
TASK_QUEUE_REPO="dfarrell07/claude-task-queue"  # Private repo
LOG_DIR="/home/dfarrell07/.local/share/claude-queue/logs"
LOCKFILE="/run/user/$(id -u)/claude-queue.lock"
MAX_TURNS=50
MAX_BUDGET_USD=5.00
TIMEOUT_SECONDS=1800
CLAUDE_BIN="/home/dfarrell07/.local/bin/claude"

# --- Repo Maps ---
declare -A REPO_PATH=(
  [submariner-operator]="/home/dfarrell07/go/src/github.com/submariner-io/submariner-operator"
  [notes-ai]="/home/dfarrell07/notes-ai"
  # ... more repos
)

declare -A REPO_REMOTE=(
  [submariner-operator]="submariner-io/submariner-operator"
  [notes-ai]="dfarrell07/notes-ai"
)

declare -A REPO_ALLOWED_TOOLS=(
  [submariner-operator]="Bash(git *),Bash(go *),Bash(make *),Read,Edit,Write"
  [notes-ai]="Bash(git *),Read,Edit,Write"
)

declare -A REPO_DEFAULT_BRANCH=(
  [submariner-operator]="devel"
  [notes-ai]="main"
)

# --- Functions ---
log() { echo "[$(date -Iseconds)] $*" | tee -a "$LOG_DIR/poller.log"; }

acquire_lock() {
  if [ -f "$LOCKFILE" ]; then
    local pid
    pid=$(cat "$LOCKFILE")
    if kill -0 "$pid" 2>/dev/null; then
      log "Another instance running (PID $pid), exiting"
      exit 0
    fi
    log "Removing stale lock (PID $pid)"
    rm -f "$LOCKFILE"
  fi
  echo $$ > "$LOCKFILE"
  trap 'rm -f "$LOCKFILE"' EXIT
}

slugify() {
  echo "$1" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-' | sed 's/^-//;s/-$//' | head -c 50
}

parse_repo() {
  # Extract repo short name from first line of issue body
  local body="$1"
  echo "$body" | head -1 | grep -oP '^repo:\s*\K\S+'
}

parse_prompt() {
  # Extract prompt: everything after the first blank line
  local body="$1"
  echo "$body" | sed '1,/^$/d'
}

fail_issue() {
  local issue_num="$1" message="$2"
  gh issue edit "$issue_num" --repo "$TASK_QUEUE_REPO" \
    --remove-label processing --add-label failed
  gh issue comment "$issue_num" --repo "$TASK_QUEUE_REPO" --body "$message"
  log "Issue #$issue_num failed: $message"
}

# --- Main ---
mkdir -p "$LOG_DIR"
acquire_lock

log "Polling for queued issues..."

# Fetch all queued issues, oldest first
ISSUES=$(gh issue list --repo "$TASK_QUEUE_REPO" \
  --label queued --state open \
  --json number,title,body \
  --jq 'sort_by(.number) | .[]' 2>/dev/null) || {
  log "Failed to fetch issues (network error?), exiting"
  exit 0
}

if [ -z "$ISSUES" ]; then
  log "No queued issues found"
  exit 0
fi

# Process each issue
echo "$ISSUES" | jq -c '.' | while IFS= read -r ISSUE; do
  ISSUE_NUM=$(echo "$ISSUE" | jq -r '.number')
  ISSUE_TITLE=$(echo "$ISSUE" | jq -r '.title')
  ISSUE_BODY=$(echo "$ISSUE" | jq -r '.body')

  log "Processing issue #$ISSUE_NUM: $ISSUE_TITLE"

  # Parse repo and prompt from body
  REPO_SHORT=$(parse_repo "$ISSUE_BODY")
  if [ -z "$REPO_SHORT" ]; then
    fail_issue "$ISSUE_NUM" "Missing \`repo:\` line in issue body. Expected format:\n\n\`\`\`\nrepo: submariner-operator\n\nYour prompt here\n\`\`\`"
    continue
  fi

  if [ -z "${REPO_PATH[$REPO_SHORT]+x}" ]; then
    fail_issue "$ISSUE_NUM" "Unknown repo: \`$REPO_SHORT\`. Known repos: ${!REPO_PATH[*]}"
    continue
  fi

  TARGET_DIR="${REPO_PATH[$REPO_SHORT]}"
  TARGET_REMOTE="${REPO_REMOTE[$REPO_SHORT]}"
  TOOLS="${REPO_ALLOWED_TOOLS[$REPO_SHORT]}"
  DEFAULT_BRANCH="${REPO_DEFAULT_BRANCH[$REPO_SHORT]:-main}"
  PROMPT=$(parse_prompt "$ISSUE_BODY")

  if [ -z "$PROMPT" ]; then
    fail_issue "$ISSUE_NUM" "Empty prompt. Add a description after the \`repo:\` line."
    continue
  fi

  if [ ! -d "$TARGET_DIR" ]; then
    fail_issue "$ISSUE_NUM" "Repo path not found: \`$TARGET_DIR\`"
    continue
  fi

  # Claim the issue
  gh issue edit "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO" \
    --remove-label queued --add-label processing
  gh issue comment "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO" \
    --body "Processing started at $(date -Iseconds) on $(hostname)"

  # Prepare branch
  BRANCH_NAME="claude/${ISSUE_NUM}-$(slugify "$ISSUE_TITLE")"
  START_TIME=$(date +%s)

  (
    cd "$TARGET_DIR"

    # Clean state: fetch, checkout default branch, pull
    git fetch origin
    git checkout "$DEFAULT_BRANCH"
    git pull --ff-only origin "$DEFAULT_BRANCH"
    git checkout -b "$BRANCH_NAME"

    # Run Claude
    FULL_PROMPT="You are processing a task from issue #${ISSUE_NUM} in ${TASK_QUEUE_REPO}.

Target repo: ${REPO_SHORT} (${TARGET_REMOTE})
Branch: ${BRANCH_NAME}

Task:
${PROMPT}

Instructions:
- Make the requested changes
- Use git to commit your changes with descriptive commit messages
- Use --signoff (-s) on all commits
- Do NOT push or create PRs -- the automation handles that
- If you cannot complete the task, explain why in your final output"

    set +e
    timeout "$TIMEOUT_SECONDS" "$CLAUDE_BIN" -p "$FULL_PROMPT" \
      --allowedTools "$TOOLS" \
      --max-turns "$MAX_TURNS" \
      2>"$LOG_DIR/issue-${ISSUE_NUM}-stderr.log" \
      >"$LOG_DIR/issue-${ISSUE_NUM}-stdout.log"
    CLAUDE_EXIT=$?
    set -e

    END_TIME=$(date +%s)
    DURATION=$((END_TIME - START_TIME))

    if [ "$CLAUDE_EXIT" -ne 0 ]; then
      STDERR_TAIL=$(tail -100 "$LOG_DIR/issue-${ISSUE_NUM}-stderr.log" 2>/dev/null || echo "(no stderr)")
      fail_issue "$ISSUE_NUM" "$(cat <<EOF
## Task Failed

**Exit code**: $CLAUDE_EXIT
**Duration**: ${DURATION}s
**Branch**: \`$BRANCH_NAME\`

<details>
<summary>Error output (last 100 lines)</summary>

\`\`\`
$STDERR_TAIL
\`\`\`
</details>

To retry: remove \`failed\` label, add \`queued\` label.
EOF
)"
      # Push branch anyway for inspection
      git push origin "$BRANCH_NAME" 2>/dev/null || true
      exit 1
    fi

    # Check if Claude actually made changes
    if git diff --quiet "$DEFAULT_BRANCH"..."$BRANCH_NAME" 2>/dev/null; then
      fail_issue "$ISSUE_NUM" "Claude completed without errors but made no changes to the code.\n\n**Duration**: ${DURATION}s\n\nClaude output:\n\`\`\`\n$(tail -50 "$LOG_DIR/issue-${ISSUE_NUM}-stdout.log")\n\`\`\`"
      exit 1
    fi

    # Push and create PR
    git push -u origin "$BRANCH_NAME"

    CLAUDE_OUTPUT=$(tail -50 "$LOG_DIR/issue-${ISSUE_NUM}-stdout.log")

    PR_URL=$(gh pr create \
      --repo "$TARGET_REMOTE" \
      --base "$DEFAULT_BRANCH" \
      --head "$BRANCH_NAME" \
      --title "$ISSUE_TITLE" \
      --body "$(cat <<EOF
## Task Queue Issue

Automatically generated from [${TASK_QUEUE_REPO}#${ISSUE_NUM}](https://github.com/${TASK_QUEUE_REPO}/issues/${ISSUE_NUM}).

## Original Request

${PROMPT}

## Claude's Summary

\`\`\`
${CLAUDE_OUTPUT}
\`\`\`

## Metadata

- **Duration**: ${DURATION}s
- **Branch**: \`${BRANCH_NAME}\`
- **Max turns**: ${MAX_TURNS}
EOF
)")

    # Mark issue done
    gh issue edit "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO" \
      --remove-label processing --add-label done
    gh issue comment "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO" \
      --body "PR opened: ${PR_URL}\n\nDuration: ${DURATION}s"
    gh issue close "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO"

    log "Issue #$ISSUE_NUM completed: $PR_URL"
  ) || {
    # Subshell failed but didn't handle it (unexpected error)
    log "Issue #$ISSUE_NUM failed with unexpected error"
    # Try to clean up labels if possible
    gh issue edit "$ISSUE_NUM" --repo "$TASK_QUEUE_REPO" \
      --remove-label processing --add-label failed 2>/dev/null || true
  }
done

log "Polling complete"
```

## Systemd Units

### Timer: claude-queue.timer

```ini
[Unit]
Description=Poll GitHub for Claude Code task queue issues

[Timer]
OnBootSec=60
OnUnitActiveSec=120
AccuracySec=10
Persistent=false

[Install]
WantedBy=timers.target
```

`OnUnitActiveSec=120` means 2 minutes after the previous run *finished*,
not 2 minutes from when it started. This naturally prevents overlap even
without the lock file.

### Service: claude-queue.service

```ini
[Unit]
Description=Process Claude Code task queue issues
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/home/dfarrell07/.local/bin/claude-queue-poller.sh
User=dfarrell07
Group=dfarrell07

# Environment
Environment=HOME=/home/dfarrell07
Environment=PATH=/home/dfarrell07/.local/bin:/usr/local/bin:/usr/bin:/bin
Environment=ANTHROPIC_API_KEY_FILE=/home/dfarrell07/.config/claude/api-key

# Resource limits
TimeoutStartSec=3600
MemoryMax=4G
CPUQuota=80%

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=claude-queue

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/home/dfarrell07
PrivateTmp=yes
```

Key systemd design choices:
- `Type=oneshot`: Timer won't start new instance while this is running
- `TimeoutStartSec=3600`: Systemd kills the whole thing after 1 hour
  (covers the 30 min Claude timeout plus overhead for multiple issues)
- `MemoryMax=4G`: Prevent runaway memory from crashing the laptop
- `CPUQuota=80%`: Leave some CPU for the user
- `ProtectSystem=strict` + `ReadWritePaths`: Only allow writes to home dir

### Installation

```bash
# As user (systemd user units)
mkdir -p ~/.config/systemd/user/
cp claude-queue.service ~/.config/systemd/user/
cp claude-queue.timer ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable claude-queue.timer
systemctl --user start claude-queue.timer

# Verify
systemctl --user status claude-queue.timer
systemctl --user list-timers
journalctl --user -u claude-queue.service -f
```

## Systemd vs Cron

Systemd timer is the right choice for this. Reasons:

| Feature | systemd timer | cron |
|---------|--------------|------|
| Overlap prevention | Built-in (Type=oneshot) | Need flock wrapper |
| Logging | journalctl with structured fields | File-based, self-managed |
| Failure tracking | `systemctl --user status` shows last failure | Parse log files |
| Resource limits | MemoryMax, CPUQuota built-in | None |
| Security | ProtectSystem, NoNewPrivileges | None |
| Depends on network | After=network-online.target | No dependency system |
| Timer after completion | OnUnitActiveSec | Fixed interval (overlaps) |

## Ansible Deployment

The Ansible role structure:

```
roles/claude_task_queue/
  defaults/main.yml      # TASK_QUEUE_REPO, timeouts, repo map
  tasks/main.yml         # Install script, systemd units, enable timer
  templates/
    claude-queue-poller.sh.j2
    claude-queue.service.j2
    claude-queue.timer.j2
  handlers/main.yml      # Restart timer on config change
```

### defaults/main.yml

```yaml
claude_queue_repo: "dfarrell07/claude-task-queue"
claude_queue_poll_interval_sec: 120
claude_queue_max_turns: 50
claude_queue_max_budget_usd: "5.00"
claude_queue_timeout_sec: 1800
claude_queue_memory_max: "4G"
claude_queue_cpu_quota: "80%"

claude_queue_repos:
  submariner-operator:
    path: "/home/{{ ansible_user }}/go/src/github.com/submariner-io/submariner-operator"
    remote: "submariner-io/submariner-operator"
    default_branch: "devel"
    allowed_tools: "Bash(git *),Bash(go *),Bash(make *),Read,Edit,Write"
  notes-ai:
    path: "/home/{{ ansible_user }}/notes-ai"
    remote: "dfarrell07/notes-ai"
    default_branch: "main"
    allowed_tools: "Bash(git *),Read,Edit,Write"
```

### tasks/main.yml

```yaml
- name: Create log directory
  file:
    path: "{{ ansible_user_dir }}/.local/share/claude-queue/logs"
    state: directory
    mode: '0750'

- name: Install poller script
  template:
    src: claude-queue-poller.sh.j2
    dest: "{{ ansible_user_dir }}/.local/bin/claude-queue-poller.sh"
    mode: '0750'
  notify: Restart claude-queue timer

- name: Create systemd user unit directory
  file:
    path: "{{ ansible_user_dir }}/.config/systemd/user"
    state: directory

- name: Install systemd service
  template:
    src: claude-queue.service.j2
    dest: "{{ ansible_user_dir }}/.config/systemd/user/claude-queue.service"
  notify: Restart claude-queue timer

- name: Install systemd timer
  template:
    src: claude-queue.timer.j2
    dest: "{{ ansible_user_dir }}/.config/systemd/user/claude-queue.timer"
  notify: Restart claude-queue timer

- name: Enable and start timer
  systemd:
    name: claude-queue.timer
    scope: user
    enabled: true
    state: started
    daemon_reload: true

- name: Create task queue GitHub repo
  command: >
    gh repo create {{ claude_queue_repo }} --private
    --description "Claude Code task queue - create issues, get PRs"
  register: repo_create
  changed_when: repo_create.rc == 0
  failed_when: repo_create.rc != 0 and 'already exists' not in repo_create.stderr

- name: Create issue labels
  loop:
    - { name: "queued", color: "0e8a16", description: "Ready for processing" }
    - { name: "processing", color: "fbca04", description: "Claude is working on this" }
    - { name: "done", color: "6f42c1", description: "PR opened successfully" }
    - { name: "failed", color: "d73a4a", description: "Processing failed" }
  command: >
    gh label create "{{ item.name }}"
    --repo {{ claude_queue_repo }}
    --color "{{ item.color }}"
    --description "{{ item.description }}"
    --force
```

## Edge Cases

### Issue edited while processing

The poller reads the issue body once at pickup time. Edits during processing
are ignored. The prompt is snapshot at claim time.

### Laptop sleeps/reboots during processing

- The `processing` label stays on the issue
- On next boot, systemd timer starts, poller runs
- The poller only looks for `queued` issues, so the stuck `processing` issue
  is skipped
- User must manually re-label to `queued` to retry
- Future improvement: add a watchdog that checks for issues stuck in
  `processing` for more than 1 hour and re-labels them

### Multiple issues, one fails

The `while` loop uses `continue` on failure, so subsequent issues are still
processed. Each issue is independent.

### Branch already exists

If the user retries an issue, the branch name is the same. The script should
handle this:

```bash
if git rev-parse --verify "$BRANCH_NAME" 2>/dev/null; then
  git branch -D "$BRANCH_NAME"
fi
git checkout -b "$BRANCH_NAME"
```

### Target repo has uncommitted changes

The poller should check for clean state:

```bash
if ! git diff --quiet || ! git diff --cached --quiet; then
  fail_issue "$ISSUE_NUM" "Target repo has uncommitted changes: \`$TARGET_DIR\`"
  continue
fi
```

### Rate limiting

GitHub API rate limits are 5000 requests/hour for authenticated users.
At 2-minute polling with ~3 API calls per poll (list + edit + comment),
that's ~90 calls/hour. Well within limits.

### Log rotation

Logs accumulate in `$LOG_DIR`. Add a systemd-tmpfiles config or a simple
find-and-delete in the poller:

```bash
# Clean logs older than 30 days
find "$LOG_DIR" -name "issue-*" -mtime +30 -delete 2>/dev/null || true
```

## Quick Start (Manual Testing)

Before deploying with Ansible, test manually:

```bash
# 1. Create the private repo
gh repo create dfarrell07/claude-task-queue --private

# 2. Create labels
gh label create queued --repo dfarrell07/claude-task-queue --color 0e8a16
gh label create processing --repo dfarrell07/claude-task-queue --color fbca04
gh label create done --repo dfarrell07/claude-task-queue --color 6f42c1
gh label create failed --repo dfarrell07/claude-task-queue --color d73a4a

# 3. Create a test issue
gh issue create --repo dfarrell07/claude-task-queue \
  --title "Test: add a comment to notes-ai README" \
  --body "repo: notes-ai

Add a comment at the top of any file explaining this is a test." \
  --label queued

# 4. Run the poller manually
./claude-queue-poller.sh

# 5. Check results
gh issue list --repo dfarrell07/claude-task-queue --state all
journalctl --user -u claude-queue.service --since "5 minutes ago"
```

## Future Enhancements

- **Priority**: `priority:high` label to process first
- **Stale processing watchdog**: Auto-reset issues stuck in `processing`
- **Dry run mode**: `repo: notes-ai --dry-run` to see what Claude would do
- **Cost tracking**: Parse Claude output for token usage, track per-issue costs
- **Issue templates**: GitHub issue template with repo dropdown
- **Slack/email notification**: On failure, notify beyond just the GitHub label
- **Multi-machine**: Multiple laptops could claim issues, using `processing`
  label as distributed lock (with hostname in comment to identify claimer)
