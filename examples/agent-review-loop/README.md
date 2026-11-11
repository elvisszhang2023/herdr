# Herdr Agent Review Loop

Phase 1 automation for a local two-agent workflow. By default the
Orchestrator owns the Git and Herdr isolation lifecycle:

```text
Orchestrator worktree → dynamic Workspace/Pane → Codex coding → local commit → local tests → Claude review
     ↑                                  ↓
     └──── findings and automatic fix ┘
```

The loop runs for at most three review rounds. It does not push, create a PR,
or merge. After review approval, the managed worktree and branch remain for a
human to inspect and create a Draft PR.

When configured, the orchestrator can also notify a human when any pane listed
in `notification_pane_ids` enters `blocked`, or when the orchestrator enters `awaiting_human`. The
notification is sent only on a new blocked reason and is not used to reply to
the Agent or approve a merge.

## Prerequisites

- macOS or Linux
- Herdr 0.9.0 or newer
- Python 3.11 or newer
- A running Herdr server
- A clean source repository checkout

The reviewer pane is instructed to be read-only. The Orchestrator creates a
new worktree from `base_branch`, opens a new Herdr Workspace, splits a second
Pane, and starts Codex and Claude in those dynamically returned Panes. The
source checkout is never switched to the task branch.

## Configure

Link the example plugin while developing it:

```bash
herdr plugin link /absolute/path/to/herdr/examples/agent-review-loop
herdr plugin config-dir example.agent-review-loop
```

Copy `config.example.toml` to the printed config directory as `config.toml`.
Set the source repository path, base branch, test command, and task. Workspace
and Pane IDs are not configured in worktree mode; Herdr returns them when the
Orchestrator creates the task Workspace.

To drive the task from a GitHub Issue instead of a local `task`, configure:

```toml
[github]
enabled = true
repo = "owner/repository"
issue_number = 123
write_back = true
comment_updates = true
```

The orchestrator reads the open Issue through the local `gh` CLI and sends its
title and body to the coding Agent as requirements. The Issue text is treated
as untrusted project input; it cannot authorize secret access, permission
changes, bypassing review, or merging. With `write_back = true`, lifecycle
comments are appended to the Issue. The current Phase 1 implementation still
does not create or update a PR, push, or merge.

In worktree mode, the Orchestrator checks that the source repository is clean,
creates a task branch such as `codex/issue-123` from `base_branch`, and places
it under `worktree_root` (by default, a sibling `<repo>-worktrees` directory).
It then creates the Herdr Workspace and both Agent Panes. The coding Agent does
not create or switch branches. The worktree is intentionally kept after
completion for inspection and later PR creation.

Optional notifications use a single incoming webhook:

```toml
[notification]
enabled = true
platform = "feishu" # generic, feishu, or wecom
webhook_url = "https://open.feishu.cn/open-apis/bot/v2/hook/..."
timeout_seconds = 10
```

`feishu` sends a text bot message, `wecom` sends a Markdown bot message, and
`generic` sends a JSON event payload. Keep webhook URLs in the local config
directory and never commit them. Notification delivery failures are logged but
do not stop the review loop; the notification key remains unrecorded so a later
reconciled event can retry it.

The dynamically returned Pane IDs are stored in the state file and used for
notifications and event subscriptions. The currently focused Pane is never
used as an implicit target. Fixed Workspace/Pane configuration is no longer
supported.

## Run

Open the orchestrator in a Herdr tab:

```bash
herdr plugin pane open \
  --plugin example.agent-review-loop \
  --entrypoint orchestrator
```

Or start it directly from this directory:

```bash
python3 orchestrator.py --config /path/to/config.toml
```

The orchestrator subscribes to `pane.agent_status_changed` before sending the
first prompt. It sends the coding task to Codex, waits for a settled coder
state, requires a new commit, runs the configured test command, and sends Claude
a Review prompt that contains the base SHA and current head SHA. Review
findings are parsed from a JSON object and sent back to Codex. Returning to an
idle state without a new commit is treated as an incomplete coding round.

## Expected review output

Claude is asked to output one JSON object:

```json
{
  "result": "approved",
  "reviewed_head_sha": "def456",
  "findings": [],
  "summary": "No blocking issues found."
}
```

or:

```json
{
  "result": "changes_requested",
  "reviewed_head_sha": "def456",
  "findings": [
    {
      "severity": "blocking",
      "file": "src/example.rs",
      "line": 42,
      "message": "The empty result is not handled.",
      "suggestion": "Add an explicit empty-result branch and a test."
    }
  ],
  "summary": "One blocking issue found."
}
```

Only `blocking` and `important` findings trigger another coding round. Minor
findings are retained in the state file but do not block approval.

## State and logs

The state file defaults to the plugin state directory when launched by Herdr,
or `.herdr/agent-review-loop.json` under the repository for direct runs. The
state records the task, pane IDs, base/head SHAs, current phase, review round,
and the last failure or Review result.

The program prints phase transitions and command failures to its terminal Pane.

## Safety boundaries

- A clean `main` or `master` is automatically left untouched while the
  orchestrator creates a task branch from it.
- Dirty worktrees and detached HEADs are rejected unless explicitly prepared.
- The configured test command is an argv array, not a shell string.
- Review is tied to the exact `head_sha` that was tested.
- Review Agent prompts prohibit writes, commits, pushes, resets, and merges.
- The loop stops after `max_review_rounds`.
- The loop never runs `git push`, PR creation, or merge commands.
