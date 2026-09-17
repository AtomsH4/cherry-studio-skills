---
name: design-cherry-lifecycle
description: Use when planning or reviewing a Cherry Studio main-process capability that may own long-lived resources or persistent side effects, or may need startup ordering, activation, pause/resume, restart, shutdown cleanup, or recovery.
---

# Design Cherry Lifecycle

Analyze whether a Cherry Studio main-process capability needs lifecycle ownership. Lifecycle exists to own runtime resources and persistent side effects; a class name, implementation complexity, uniform shape, or convenient dependency injection does not justify it.

## Scope and authority

This is read-only design analysis:

- Do not edit the target, write code, or mutate the repository under review.
- Do not produce an executable plan, run implementation commands, or begin implementation.
- Treat review authority as authority to report findings, not authority to fix them.
- Separate correctness requirements from tradeoffs and preferences.

Read [references/lifecycle-analysis.md](references/lifecycle-analysis.md) in full before analyzing a target. Its verdict set, evidence requirements, decision ladder, and output contract are mandatory.

## Gather repository evidence

When the target repository is available, read its `AGENTS.md` instructions, lifecycle README and decision guide, relevant architecture documentation, nearby README files, and comparable services. Prefer current repository documentation and code over this skill if they differ.

Load boundary-specific documentation only when the target crosses that boundary: DataApi, IpcApi, WindowManager, centralized paths, jobs/background work, or testing. Do not expand the investigation into unrelated subsystems.

## Analysis workflow

1. Parse the requested target and the claimed capability boundary.
2. Reconstruct resource ownership, lifetime, initialization, cleanup, partial failure, restart, and recovery from repository evidence.
3. Distinguish the data owner, command orchestrator, domain-entity state, and runtime-resource owner; do not collapse them into one service by default.
4. Return exactly one verdict defined in the reference. If the evidence cannot establish ownership or lifecycle requirements, return `Insufficient evidence` and enumerate the missing facts.
5. Before proposing a new lifecycle service, compare an ordinary function, a direct-import singleton, and an existing lifecycle owner.
6. Only when justified, recommend the smallest lifecycle shape and a non-executable implementation outline.

Use the reference's fixed response sections. Put every unresolved fact that could change the verdict or design in `Open evidence`.
