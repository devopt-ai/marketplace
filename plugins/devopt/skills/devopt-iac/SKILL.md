---
name: devopt-iac
description: >
  Manage DevOpt resources (executors, tasks, pipelines, triggers) as code via
  DevOpt IaC. Use when the user wants to define, review, or apply DevOpt
  infrastructure declaratively — "add a DevOpt task", "define a pipeline",
  "apply my DevOpt config", or reconciling drift between files and the live
  account.
---

# DevOpt IaC

DevOpt resources are managed Terraform-style: declarative definition files,
then a plan/apply reconcile against the account.

## Workflow

1. Define resources in your project's DevOpt IaC files (executors, tasks,
   pipelines, triggers). Each resource has a stable key; references between
   resources use those keys.
2. Plan first: ask DevOpt (via the devopt MCP tools) to compute the plan —
   review the create/update/delete set before touching anything.
3. Apply only a reviewed plan. Apply is idempotent: re-running converges to
   the declared state.

## Rules of thumb

- Never hand-edit live resources that IaC owns — the next apply reverts it.
- Small diffs: change one resource group per apply so plans stay reviewable.
- Secrets never go in definition files; reference DevOpt-stored credentials.
