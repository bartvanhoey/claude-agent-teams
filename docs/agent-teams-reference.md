# Agent Teams — Master Reference Guide

> Source: https://code.claude.com/docs/en/agent-teams  
> Claude Code v2.1.32+ required. Run `claude --version` to check.

---

## Table of Contents

1. [What Are Agent Teams?](#1-what-are-agent-teams)
2. [Agent Teams vs Subagents](#2-agent-teams-vs-subagents)
3. [Enabling Agent Teams](#3-enabling-agent-teams)
4. [Starting a Team](#4-starting-a-team)
5. [Controlling a Team](#5-controlling-a-team)
6. [Architecture & Internals](#6-architecture--internals)
7. [Hooks for Quality Gates](#7-hooks-for-quality-gates)
8. [Token Costs](#8-token-costs)
9. [Best Practices](#9-best-practices)
10. [Use Case Examples](#10-use-case-examples)
11. [Troubleshooting](#11-troubleshooting)
12. [Known Limitations](#12-known-limitations)
13. [Subagents Reference](#13-subagents-reference)

---

## 1. What Are Agent Teams?

Agent teams coordinate multiple independent Claude Code instances. One session is the **team lead**; the rest are **teammates**. Each teammate has its own context window, claims tasks from a shared list, and communicates directly with other teammates — not just back to the lead.

**Experimental — disabled by default.**

**Best use cases (parallel exploration adds real value):**
- Research and review: multiple angles investigated simultaneously
- New modules or features: each teammate owns a separate file set
- Debugging with competing hypotheses: parallel investigation converges faster
- Cross-layer coordination: frontend/backend/tests owned by different teammates

**Poor fit:**
- Sequential tasks with many dependencies
- Same-file edits (causes conflicts)
- Routine, simple changes (single session is cheaper)

---

## 2. Agent Teams vs Subagents

| | Subagents | Agent Teams |
|---|---|---|
| **Context** | Own context; results return to caller | Own context; fully independent |
| **Communication** | Report results to main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list, self-coordination |
| **Best for** | Focused tasks where only the result matters | Complex work needing discussion and collaboration |
| **Token cost** | Lower — results summarized back | Higher — each teammate is a separate instance |

**Key distinction:** subagents cannot talk to each other. Agent team teammates share a task list and can send direct messages to any named teammate.

Use subagents when workers only need to report back. Use agent teams when workers need to share findings, challenge each other, and coordinate independently.

---

## 3. Enabling Agent Teams

Agent teams are **disabled by default**. Enable via `settings.json` or environment variable:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Or in your shell: `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

---

## 4. Starting a Team

Describe the task and team structure in natural language. Claude creates the team, spawns teammates, and begins coordinating.

**Good example — independent roles with no overlap:**
```
I'm designing a CLI tool that helps developers track TODO comments across
their codebase. Create an agent team to explore this from different angles:
one teammate on UX, one on technical architecture, one playing devil's advocate.
```

Claude will:
1. Create a shared task list
2. Spawn teammates for each perspective
3. Coordinate exploration
4. Synthesize findings
5. Clean up the team when finished

---

## 5. Controlling a Team

### Display Modes

| Mode | How it works | Requirements |
|---|---|---|
| `in-process` (default) | All teammates in your main terminal; use Shift+Down to cycle | Any terminal |
| `tmux` / split panes | Each teammate in its own pane; click to interact | tmux or iTerm2 with `it2` CLI |

Auto-detection: uses split panes if already inside tmux; otherwise in-process.

Override in `~/.claude/settings.json`:
```json
{ "teammateMode": "in-process" }
```

Or per-session:
```bash
claude --teammate-mode in-process
```

**tmux works best on macOS.** Use `tmux -CC` inside iTerm2 as the suggested entry point.  
**Split panes are NOT supported** in VS Code integrated terminal, Windows Terminal, or Ghostty.

### Keyboard Shortcuts (In-Process Mode)

| Key | Action |
|---|---|
| Shift+Down | Cycle to next teammate (wraps back to lead after last) |
| Enter | View a teammate's session |
| Escape | Interrupt a teammate's current turn |
| Ctrl+T | Toggle task list |

### Specifying Teammates and Models

Claude chooses team size from the task, or you can be explicit:
```
Create a team with 4 teammates to refactor these modules in parallel.
Use Sonnet for each teammate.
```

Teammates do **not** inherit the lead's `/model` by default. Set a default in `/config` → "Default teammate model". Choose "Default (leader's model)" to have teammates follow the lead.

### Requiring Plan Approval

```
Spawn an architect teammate to refactor the authentication module.
Require plan approval before they make any changes.
```

Flow:
1. Teammate works in read-only plan mode
2. Sends plan approval request to lead
3. Lead approves or rejects with feedback
4. If rejected, teammate revises and resubmits
5. Once approved, teammate begins implementation

Control lead judgment with criteria: *"only approve plans that include test coverage"* or *"reject plans that modify the database schema."*

### Task Assignment

The shared task list has three states: **pending → in progress → completed**. Tasks can depend on other tasks; blocked tasks auto-unblock when dependencies complete. Task claiming uses file locking to prevent race conditions.

- **Lead assigns**: tell the lead which task to give to which teammate
- **Self-claim**: after finishing, a teammate picks up the next unassigned, unblocked task

### Shutting Down Teammates

```
Ask the researcher teammate to shut down
```

The lead sends a shutdown request. The teammate can approve (exits gracefully) or reject with an explanation.

### Cleaning Up

```
Clean up the team
```

Always run cleanup through the **lead**, not a teammate. Cleanup checks for active teammates and fails if any are still running — shut them down first.

---

## 6. Architecture & Internals

### Components

| Component | Role |
|---|---|
| Team lead | The main session; creates, spawns, and coordinates |
| Teammates | Separate Claude Code instances working assigned tasks |
| Task list | Shared work items with states and dependencies |
| Mailbox | Messaging system for inter-agent communication |

### How Teams Start

- **You request it**: explicitly ask for an agent team
- **Claude proposes it**: Claude suggests a team for parallel work; you confirm before it proceeds

Claude never creates a team without your approval.

### Storage (local only)

| What | Where |
|---|---|
| Team config | `~/.claude/teams/{team-name}/config.json` |
| Task list | `~/.claude/tasks/{team-name}/` |

**Do not hand-edit the team config.** It holds runtime state (session IDs, tmux pane IDs) and is overwritten on every state update.

The `members` array in the team config contains each teammate's name, agent ID, and agent type. Teammates can read this file to discover others.

**There is no project-level team config.** A `.claude/teams/teams.json` in your project is treated as an ordinary file, not configuration.

### Context and Communication

Each teammate loads the same project context as a regular session: `CLAUDE.md`, MCP servers, and skills. The lead's conversation history does **not** carry over. Include task-specific details in the spawn prompt.

**Information sharing mechanisms:**
- Automatic message delivery (no polling needed)
- Idle notifications when a teammate stops
- Shared task list visible to all agents
- Direct named messaging: any teammate can message any other by name

The lead assigns names when spawning. To get predictable names, tell the lead what to call each teammate in the spawn instruction.

To reach everyone, send **one message per recipient** — there is no broadcast.

### Using Subagent Definitions for Teammates

Reference any subagent type (project, user, plugin, or CLI-defined) when spawning a teammate:

```
Spawn a teammate using the security-reviewer agent type to audit the auth module.
```

The teammate uses that definition's `tools` allowlist and `model`. The definition body is **appended** to the teammate's system prompt (not replacing it). Team coordination tools (`SendMessage`, task management) are always available regardless of `tools` restrictions.

**Note:** `skills` and `mcpServers` frontmatter fields are **not applied** when a subagent definition runs as a teammate. Teammates load skills and MCP servers from project/user settings like a regular session.

### Permissions

All teammates start with the **lead's permission settings**. If the lead uses `--dangerously-skip-permissions`, all teammates do too. You can change individual teammate modes after spawning, but not at spawn time.

---

## 7. Hooks for Quality Gates

Three hooks enforce rules at key lifecycle events:

### `TeammateIdle`
Fires when a teammate is about to go idle.

**Input fields:**
| Field | Type | Description |
|---|---|---|
| `teammate_id` | string | Unique identifier |
| `teammate_name` | string | Name assigned by lead |
| `teammate_type` | string | e.g., `"general-purpose"`, `"Explore"`, or custom name |

**Exit code 2** (or `{"continue": false}`) keeps the teammate working.

```bash
#!/bin/bash
if pending_tasks_exist; then
  echo "Pending tasks remain, teammate should continue" >&2
  exit 2
fi
exit 0
```

```json
{ "continue": false, "stopReason": "Research tasks still in progress" }
```

### `TaskCreated`
Fires when a task is being created via `TaskCreate`.

**Input fields:** `task_id`, `task_name`, `task_description`, `assigned_to`

**Exit code 2** (or `{"decision": "block", "reason": "..."}`) prevents creation.

```json
{ "decision": "block", "reason": "Teammate workload is at capacity" }
```

### `TaskCompleted`
Fires when a task is being marked complete.

**Input fields:** `task_id`, `task_name`, `task_description`, `assigned_to`, `completion_notes`

**Exit code 2** (or `{"decision": "block", "reason": "..."}`) prevents completion.

```bash
#!/bin/bash
task_id=$(jq -r '.task_id' < /dev/stdin)
if ! test_results_exist "$task_id"; then
  echo "Task completion blocked: required tests not found" >&2
  exit 2
fi
exit 0
```

### Hook Exit Codes (All Three Hooks)

| Exit code | Behavior |
|---|---|
| 0 | Action proceeds normally |
| 2 | Action blocked; stderr shown to user |
| Other | Non-blocking error; execution continues (except `TeammateIdle` treats any non-zero as non-blocking) |

---

## 8. Token Costs

Agent teams use **significantly more tokens** than a single session — roughly **~7x more** when teammates run in plan mode, because each teammate maintains its own context window as a separate Claude instance.

**Cost reduction strategies:**
- Use **Sonnet** for teammates (best capability/cost balance for coordination)
- Keep teams **small** — token usage scales linearly with active teammates
- Keep **spawn prompts focused** — everything in the prompt adds to context from the start
- **Clean up** when done — active teammates consume tokens even when idle
- Keep tasks **small and self-contained** to limit per-teammate context

**Rate limits for teams (per-user recommendations):**

| Team size | TPM per user | RPM per user |
|---|---|---|
| 1–5 users | 200k–300k | 5–7 |
| 5–20 users | 100k–150k | 2.5–3.5 |
| 20–50 users | 50k–75k | 1.25–1.75 |
| 50–100 users | 25k–35k | 0.62–0.87 |
| 100–500 users | 15k–20k | 0.37–0.47 |
| 500+ users | 10k–15k | 0.25–0.35 |

---

## 9. Best Practices

### Give Teammates Enough Context

Teammates don't inherit lead conversation history. Include task-specific details in the spawn prompt:

```
Spawn a security reviewer teammate with the prompt: "Review the authentication module
at src/auth/ for security vulnerabilities. Focus on token handling, session
management, and input validation. The app uses JWT tokens stored in
httpOnly cookies. Report any issues with severity ratings."
```

`CLAUDE.md` loads normally — use it to provide project-wide guidance to all teammates.

### Team Size

- Start with **3–5 teammates** for most workflows
- Token cost scales linearly — each teammate has its own context window
- Coordination overhead increases with more teammates
- Diminishing returns beyond ~5
- Target **5–6 tasks per teammate** — keeps everyone productive without context switching
- Scale up only when work genuinely benefits from true parallelism

### Task Sizing

| Size | Problem |
|---|---|
| Too small | Coordination overhead exceeds benefit |
| Too large | Long runs without check-ins; wasted effort risk |
| Just right | Self-contained unit with a clear deliverable (a function, a test file, a review) |

If the lead isn't creating enough tasks, ask it to split the work into smaller pieces.

### Avoid File Conflicts

Two teammates editing the same file causes overwrites. Design tasks so **each teammate owns a different set of files**.

### Don't Let the Lead Do Work

If the lead starts implementing instead of delegating:
```
Wait for your teammates to complete their tasks before proceeding
```

### Monitor and Steer

Check in on progress, redirect approaches that aren't working, synthesize findings as they arrive. Unattended teams increase the risk of wasted effort.

### Start with Research and Review

If new to agent teams, begin with tasks that have clear boundaries and don't involve code writing: PR reviews, library research, bug investigation. This demonstrates parallel exploration value without coordination complexity.

---

## 10. Use Case Examples

### Parallel Code Review

Split review criteria into independent domains so all angles get thorough attention simultaneously:

```
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

Each reviewer applies a different filter to the same PR. The lead synthesizes findings after they finish.

### Competing Hypothesis Debugging

Make teammates adversarial to fight single-path anchoring bias:

```
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific
debate. Update the findings doc with whatever consensus emerges.
```

The debate structure is the key mechanism. A theory that survives active peer challenge is much more likely to be the actual root cause than one found by sequential investigation.

---

## 11. Troubleshooting

### Teammates Not Appearing

- In in-process mode, press Shift+Down — they may be running but not visible
- Check that the task was complex enough to warrant a team
- Verify tmux is in PATH: `which tmux`
- For iTerm2: verify `it2` CLI is installed and Python API is enabled in preferences

### Too Many Permission Prompts

Pre-approve common operations in permission settings before spawning teammates.

### Teammates Stopping on Errors

Use Shift+Down (in-process) or click the pane (split mode) to check output. Either give direct instructions or spawn a replacement teammate.

### Lead Shuts Down Early

Tell the lead to keep going or explicitly wait for teammates to finish.

### Orphaned tmux Sessions

```bash
tmux ls
tmux kill-session -t <session-name>
```

---

## 12. Known Limitations

| Limitation | Detail |
|---|---|
| No session resumption | `/resume` and `/rewind` do not restore in-process teammates; lead may message teammates that no longer exist |
| Task status lag | Teammates sometimes fail to mark tasks complete; check and nudge manually |
| Slow shutdown | Teammates finish their current request/tool call before shutting down |
| One team at a time | A lead can only manage one team; clean up before creating a new one |
| No nested teams | Teammates cannot spawn their own teams; only the lead manages the team |
| Fixed lead | The session that creates the team is lead for its lifetime; leadership cannot transfer |
| Permissions set at spawn | All teammates start with lead's permission mode; cannot set per-teammate modes at spawn time |
| Split panes: terminal limits | Requires tmux or iTerm2; not supported in VS Code terminal, Windows Terminal, or Ghostty |

---

## 13. Subagents Reference

Subagents are the lighter-weight alternative — they run within a single session and return results to the caller without inter-agent communication. They are also the building block for teammate definitions.

### Built-In Subagents

| Agent | Model | Tools | Purpose |
|---|---|---|---|
| **Explore** | Haiku | Read-only | Fast codebase search and analysis |
| **Plan** | Inherited | Read-only | Research during plan mode |
| **general-purpose** | Inherited | All tools | Complex multi-step tasks |
| statusline-setup | Sonnet | — | `/statusline` configuration |
| claude-code-guide | Haiku | — | Claude Code feature questions |

### Subagent File Format

```markdown
---
name: code-reviewer           # required; lowercase letters and hyphens
description: Reviews code...  # required; Claude uses this to decide when to delegate
tools: Read, Glob, Grep       # optional; inherits all if omitted
model: sonnet                 # optional; sonnet/opus/haiku/full-model-id/inherit
permissionMode: default       # optional
maxTurns: 10                  # optional
memory: project               # optional; user/project/local
isolation: worktree           # optional; isolated git worktree
background: false             # optional; always run as background task
color: blue                   # optional; red/blue/green/yellow/purple/orange/pink/cyan
---

System prompt content here.
```

### Subagent Scopes (Priority Order)

| Location | Scope | Priority |
|---|---|---|
| Managed settings | Organization-wide | 1 (highest) |
| `--agents` CLI flag | Current session only | 2 |
| `.claude/agents/` | Current project | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin `agents/` | Where plugin is enabled | 5 (lowest) |

When multiple subagents share the same name, the higher-priority location wins.

### Invoking Subagents

**Natural language** (Claude decides):
```
Use the code-reviewer subagent to analyze recent changes
```

**@-mention** (guaranteed delegation for one task):
```
@"code-reviewer (agent)" look at the auth changes
```

**Session-wide** (whole session uses that subagent's system prompt + tools):
```bash
claude --agent code-reviewer
```

Or set default in `.claude/settings.json`:
```json
{ "agent": "code-reviewer" }
```

### Model Resolution Order

1. `CLAUDE_CODE_SUBAGENT_MODEL` environment variable
2. Per-invocation `model` parameter
3. Subagent definition's `model` frontmatter
4. Main conversation's model

### Foreground vs Background

- **Foreground**: blocks main conversation; permission prompts surface to you
- **Background**: runs concurrently; auto-denies any tool call that would prompt
- Press **Ctrl+B** to background a running task
- Ask Claude to "run this in the background"

### Persistent Memory

```yaml
memory: project   # .claude/agent-memory/<name>/
memory: user      # ~/.claude/agent-memory/<name>/
memory: local     # .claude/agent-memory-local/<name>/ (not checked into git)
```

When enabled, the subagent's system prompt includes instructions for reading/writing the memory directory, and the first 200 lines / 25KB of `MEMORY.md` are injected at startup.

### Hooks for Subagents

**In subagent frontmatter** (runs only while that subagent is active):
```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
```

**In settings.json** (main session responding to subagent lifecycle):
```json
{
  "hooks": {
    "SubagentStart": [{ "matcher": "db-agent", "hooks": [{ "type": "command", "command": "./setup.sh" }] }],
    "SubagentStop":  [{ "hooks": [{ "type": "command", "command": "./cleanup.sh" }] }]
  }
}
```

### When to Use Subagents vs Main Conversation

**Use main conversation when:**
- Task needs frequent back-and-forth
- Multiple phases share significant context
- Making a quick targeted change
- Latency matters (subagents start fresh)

**Use subagents when:**
- Task produces verbose output you don't need in main context
- You want to enforce tool restrictions
- Work is self-contained and can return a summary

**Use agent teams when:**
- Workers need to share findings and coordinate independently
- Sustained parallelism exceeds a single context window
