---
name: devopt-board
description: >
  Work a DevOpt issue board as an agent — claim the next issue, work one
  checklist item, answer comment threads, advance columns, and hand off to a
  human at a human-only gate. Use when the user points an agent at a DevOpt
  board or an issue on one — "work my DevOpt board", "pick up the next issue",
  "what's the next item on this issue", "why did the loop stop", "the issue is
  stuck on a human gate" — or when a run of the DevOpt issue loop needs
  explaining or unsticking.
---

# DevOpt issue board

A DevOpt board holds issues; an issue moves through columns; each column holds
a checklist. Agents drive that board through the governed `devopt_issue_*`
tools on the DevOpt MCP connection (see the `devopt-connect` skill for setup).

The shape exists to solve one problem: **a long task outlives an agent's
context.** So an agent session never owns an issue — it owns ONE checklist
item, then exits. The durable state lives on the board, and the next session
starts clean and reads it back.

## The lease

`agentStatus` is the board's mutual exclusion. Exactly one agent holds an issue
at a time, and the state says what is going on:

| status | meaning | claimable again? |
| --- | --- | --- |
| *(none)* | idle | yes |
| `working` | an agent holds it | only once the lease goes stale |
| `waiting_for_human` | yielded at a human gate | **never** by an agent |
| `error` | the agent failed | yes, as a retry |
| `completed` | the agent is done with it | **never** |

**Never leave an issue in `working`.** A stranded `working` lease parks a live
issue until the staleness window expires. Set `error` when you fail, set
`waiting_for_human` when you yield, and clear the lease when you finish an
item. Wrap the work so this holds on every exit path, including the ones you
did not plan for.

A stranded lease now costs the humans something too. A **live** lease refuses
the delete of its card, its column and its whole board — the operator is told
an agent is working and asked to wait or stop the run, because deleting the row
out from under a running session would strand it. That refusal keys on
liveness, not on the `working` word: a lease past the staleness window whose
holding run is no longer alive counts as nobody's and blocks nothing. So a
crashed run never makes a board permanently undeletable — but a lease you left
behind while your session is still up makes that board unmanageable until you
exit.

## Tools

| tool | what it does |
| --- | --- |
| `devopt_issue_next` | `{boardId, staleAfterMs?}` → the next claimable issue, or nothing. Nothing is a **clear board** — success, not an error. |
| `devopt_issue_get` | `{issueId}` → full state: the current column (with its `isDone` flag), its checklist items, threads awaiting an agent, recent comments, `gitBranch`, `agentStatus`. |
| `devopt_issue_status` | `{issueId, status, message?, itemId?}` — `working` / `waiting_for_human` / `error` / `completed`. Omit or null the status to clear the lease. |
| `devopt_issue_check_item` | `{issueId, checklistItemId}` — checks an item off. The server **refuses** a `humanOnly` item. |
| `devopt_issue_satisfy_requirement` | `{issueId, requirementId, proof: {commit, summary, verification}}` — marks one of the issue's own requirements done. `proof` is required: the commit sha that satisfied it, one line of what was done, and how you verified it; it is recorded as evidence and shown on the done row. Refused for a `humanOnly` requirement, and for one that carries a Definition of Done (tick its items instead). |
| `devopt_issue_satisfy_dod_item` | `{issueId, requirementId, dodItemId, evidence?}` — ticks one Definition-of-Done item. Pass `evidence` when the item declares a contract; the server decides whether it passes. |
| `devopt_issue_start_requirement` | `{issueId, requirementId}` → `{workflowRunId}`. Starts the workflow the requirement binds. Refused when unbound, already running, or `humanOnly`. |
| `devopt_issue_add_requirement` | `{issueId, label, brief?, workflowId?, humanOnly?}` → `{requirementId}`. Record a problem or follow-up you found while working as a requirement — not as a comment. Appended, `pending`. Bind `workflowId` so it can be started. |
| `devopt_issue_advance` | `{issueId}` → `{advanced, column}`. Moves the issue to the next column. |
| `devopt_issue_comment` | `{issueId, body, threadId?, awaiting?}` — a new comment, or a reply in a thread. `awaiting` says whose turn it is next. |
| `devopt_issue_log` | `{issueId, action, details?}` — an audit entry on the issue. |

## Checklist items

Each item carries `isRequired`, `humanOnly` and `completed`.

- Work the first **incomplete, non-`humanOnly`** item in the current column, in
  board order. One item per session.
- **`humanOnly` is a wall, not a hint.** The server refuses to check one off.
  Reaching one means: comment a summary of where things stand, set
  `waiting_for_human`, and log the gate. The human checking that item off is
  what hands the issue back.
- A column advances when every **required** item in it is complete. An optional
  item left open does not hold the column.

## Finality

An issue is finished when its column's `isDone` flag is set. That flag is the
only authority — never infer finality from a column's position or from it
being the last one in the list. Boards get re-ordered and columns get added.

## Comment threads

A thread whose `awaiting` is `agent` is work: reply to it. If you can answer,
reply and leave `awaiting` unset so the thread stops asking. If you need a
human, reply saying what you need, set `awaiting: 'human'`, and put the issue
in `waiting_for_human`.

## Working an item

1. The issue's `gitBranch` is already checked out and verified before your
   session starts. Do not re-derive it or switch branches.
2. Work **only** the item you were given. The next one gets its own session
   with a clean context — that is the whole point of the shape.
3. Pull whatever else you need with `devopt_issue_get`. Your brief carries the
   item, not the issue.
4. Exit when the item is done, or when you are blocked and have said so.
5. Do not check your own item off and do not post your own summary. A separate
   verification pass judges the board and the branch, not a claim — including
   when a human stopped your session part way, which is a normal outcome.
