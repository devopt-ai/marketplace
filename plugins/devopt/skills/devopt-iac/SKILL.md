---
name: devopt-iac
description: >
  Manage DevOpt resources (executors, tasks, workflows, triggers) as code via
  DevOpt IaC, including whether a workflow step runs headless or interactively.
  Use when the user wants to define, review, or apply DevOpt infrastructure
  declaratively — "add a DevOpt task", "define a workflow", "apply my DevOpt
  config", reconciling drift between files and the live account — or when a
  step needs a human in the loop: "make this step interactive", "let me attach
  to that step's terminal", "why was my interactive step refused".
---

# DevOpt IaC

DevOpt resources are managed Terraform-style: declarative definition files,
then a plan/apply reconcile against the account.

## Workflow

1. Define resources in your project's DevOpt IaC files (executors, tasks,
   workflows, triggers). Each resource has a stable key; references between
   resources use those keys.
2. Plan first: ask DevOpt (via the devopt MCP tools) to compute the plan —
   review the create/update/delete set before touching anything.
3. Apply only a reviewed plan. Apply is idempotent: re-running converges to
   the declared state.

## Rules of thumb

- Never hand-edit live resources that IaC owns — the next apply reverts it.
- Small diffs: change one resource group per apply so plans stay reviewable.
- Secrets never go in definition files; reference DevOpt-stored credentials.

## Interactive workflow steps

A workflow step can declare an `interaction` block to opt out of headless
execution. Modes:

- `headless` — the default. No session, structured output, safe unattended.
- `attachable` — the step runs in a session a human may watch and type into,
  but the run never waits for one.
- `interactive` — the step requires a live human owner; without one it is
  refused rather than left blocked.
- `prompt` — the step pauses until someone answers. **On by default.** An
  operator can disable it fleet-wide with `WORKFLOW_PROMPT_MODE=false`, which
  rejects `mode: prompt` at plan, at save and at dispatch; if a definition is
  refused with that message, the switch is thrown on that server.

Authoring rules — all enforced at plan/save, so a bad definition fails before
it ever runs:

1. **Task steps only.** Approval steps, fan-out/parallel groups and
   sub-workflow steps must not declare `interaction` — approvals already have
   their own human semantics, a group is a coordinator, and a sub-workflow
   delegates to its child graph.
2. **Absent is not headless.** An omitted field means "inherit"; the effective
   config resolves executor defaults, then account/scope overrides, then the
   task, then the step. Write only the keys you intend to change — writing a
   key you did not mean to set silently overrides an outer layer.
3. **The executor must advertise the `accepts-input` capability.** Without it
   a non-headless step is refused at spawn on every run, so plan/save rejects
   it up front when the executor is known. Add the capability to the executor,
   or leave the step headless.
4. **Outer layers can lock interaction.** An account- or task-level policy may
   freeze `interaction.mode` (or any other `interaction.*` key). A step that
   writes a locked key is rejected — raise it with whoever owns the policy
   instead of working around it.
5. **Non-headless steps carry no structured output.** An interactive step runs
   as a terminal session and accumulates nothing to validate or pass on. Do
   not pair one with an output contract, an output-injecting edge, or a
   downstream step that references its output — each combination is rejected.
6. **Unattended triggers change the outcome.** On a schedule- or webhook-fired
   run there is no human to resolve: `attachable` quietly degrades to headless
   and the run continues, while `interactive` (and `prompt`) fail fast with
   `interaction_unattended`. Saving such a workflow warns you; if the step
   genuinely needs a human, trigger it manually.
7. **Give an unanswered prompt somewhere to go.** If a prompt step's
   `onUnanswered` disposition is not `cancel`, do not also swallow the failure
   with an `onFailure` of `skip`/`continue` and no failure route — the
   unanswered prompt would vanish with no consequence, and that is rejected.

### Known limits today

State these plainly when a user asks for something in this list — do not
design around them silently.

- `prompt` mode is unavailable unless the server operator has enabled it.
- Nobody is notified when a step pauses for input, and there is no deep link
  to the waiting step — a human has to be watching the run to answer it.
- `idleTimeoutMs` is accepted on a step but currently has no effect at the
  workflow layer (you get a warning on save). Nothing reaps an interactive
  step that simply goes idle.
- There is no API to read back the effective interaction config or which keys
  an outer policy locked. To find out, run a plan and read its errors and
  warnings.
- If the server restarts while a step is paused for input, that step fails
  with `orphaned_by_restart` and is never re-dispatched — a prompt already
  asked must not be re-asked blind. The human re-runs it.
- Interaction failures are deterministic and are never retried, even on a step
  configured with a retry disposition. They take the terminal failure route
  immediately.
