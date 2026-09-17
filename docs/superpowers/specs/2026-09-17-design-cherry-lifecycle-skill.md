# Design: Cherry Studio Lifecycle Analysis Skill

## Goal

Create a public, read-only skill that helps an agent decide whether a Cherry
Studio main-process capability should use the service lifecycle system, detects
unnecessary lifecycle abstractions, and recommends the smallest architecture-
conformant lifecycle design when lifecycle is justified.

The skill is an architecture decision aid, not an implementation agent. It may
inspect a request, diff, commit, file, directory, or proposed design, but it
must not edit product code, create an implementation plan, or treat a review
request as authorization to change the target.

## Repository

- GitHub repository: `AtomsH4/cherry-studio-skills`
- Visibility: public
- Default branch: `main`
- License: MIT
- Initial skill: `skills/design-cherry-lifecycle/`
- Future Cherry Studio skills may be added as sibling directories under
  `skills/`.

This is a skill collection rather than a Codex plugin. The first release does
not need MCP servers, app integrations, package publishing, custom installers,
or CI infrastructure.

## Trigger Boundary

Use the skill for Cherry Studio design or review work involving possible
main-process lifecycle ownership, including long-lived resources, persistent
side effects, startup ordering, activation, pause/resume, restart, shutdown,
cleanup, or recovery.

Do not trigger it merely because a class is named `Service`, a workflow spans
multiple services, an IPC route exists, or a business entity has archive,
restore, or deletion states. Those are evidence to inspect, not proof that the
service lifecycle system is appropriate.

## Required Evidence

Before recommending lifecycle, the analysis must identify:

1. The concrete resource or persistent side effect being owned.
2. When it is created and how long it survives.
3. Which component is responsible for releasing or undoing it.
4. What startup, shutdown, restart, failure, and recovery behavior is required.
5. Existing Cherry Studio owners and comparable implementations.
6. Why a direct-import singleton or integration into an existing owner is
   insufficient.

When repository access is available, current repository documentation and code
are authoritative. At minimum, inspect `AGENTS.md`, the lifecycle reference
entry point and decision guide, relevant process architecture documentation,
nearby README files, and analogous services. Load DataApi or IPC documentation
only when those boundaries are involved.

## Decision Model

The top-level verdict is exactly one of:

- **Lifecycle required**: the component owns a resource or persistent side
  effect that outlives a method call and needs coordinated initialization,
  cleanup, pause/resume, restart, or shutdown behavior.
- **Lifecycle not justified**: the work is stateless, request-scoped, pure data
  access, pure computation, or better owned by an existing component.
- **Insufficient evidence**: ownership, lifetime, cleanup, or failure behavior
  cannot be established from the available material.

After lifecycle is justified, select the smallest fitting pattern:

- ordinary lifecycle service;
- lifecycle plus `@Conditional` for immutable boot-time exclusion;
- lifecycle plus `Activatable` for runtime-controlled or on-demand resources;
- lifecycle plus `Pausable` for coordinated temporary suspension;
- integration into an existing lifecycle owner instead of a new service.

The skill must separately evaluate data ownership, command orchestration, and
runtime-resource ownership. A stateless coordinator may remain a direct-import
singleton even when it executes a complex business command.

## Over-Design Checks

The analysis must challenge a proposed lifecycle service when:

- `onInit` and `onStop` would be empty or ceremonial;
- all resources are created and released inside one method call;
- the class only groups stateless IPC handlers;
- lifecycle is being used as dependency injection for otherwise stateless
  logic;
- an existing lifecycle owner can own the resource without losing cohesion;
- `@DependsOn` restates cross-phase ordering already guaranteed by the
  container;
- `Conditional`, `Activatable`, or `Pausable` is added without the condition or
  state transition it represents;
- a new registry, adapter, state machine, or extension point has no current
  consumer-driven need;
- domain-specific behavior is being added to the generic lifecycle engine.

The correction for over-design is removal or consolidation, not a more
elaborate version of the same abstraction.

## Analysis Workflow

1. Resolve the target and state that the task is read-only.
2. Read governing documentation and comparable implementations.
3. Reconstruct the current or proposed ownership and resource lifetime.
4. Apply the decision model and over-design checks.
5. Compare the recommended choice with at least the relevant non-lifecycle
   alternatives: a direct-import singleton and integration into an existing
   owner.
6. If lifecycle is justified, describe the minimum lifecycle shape and its
   observable contracts.
7. Report evidence gaps instead of inventing resources, dependencies, phases,
   or cleanup requirements.

## Output Contract

The result contains these sections:

1. **Verdict** — one top-level verdict with a short rationale and confidence.
2. **Evidence** — owned resources, persistent side effects, current owners,
   lifetime, cleanup, concurrency, failure, and recovery facts, each tied to a
   file, symbol, diff line, or explicit requirement where possible.
3. **Over-design assessment** — unnecessary lifecycle machinery or an explicit
   statement that none was evidenced.
4. **Alternatives considered** — direct singleton, existing-owner integration,
   and the recommended option with meaningful tradeoffs.
5. **Recommended lifecycle design** — only when justified: owner, phase,
   same-phase dependencies, hooks, disposables, optional lifecycle interface,
   IPC placement, error strategy, shutdown behavior, and recovery approach.
6. **Implementation outline** — affected areas and behavioral tests, without
   editing files or expanding into an executable step-by-step plan.
7. **Open evidence** — facts that must be confirmed before implementation.

Findings must distinguish mandatory correctness issues from design tradeoffs.
The skill must not claim a problem solely from naming, code style, or the
absence of an abstraction.

## Progressive Disclosure

`SKILL.md` will contain the trigger boundary, read-only rule, high-level
workflow, and routing instructions. Detailed decision criteria, false-positive
guards, Cherry Studio lifecycle constraints, and the report contract will live
in `references/lifecycle-analysis.md` and be loaded whenever the skill runs.

The skill will not copy whole repository manuals. It will point to the current
Cherry Studio documentation and encode only the review behavior needed to turn
that documentation into a consistent decision.

## Validation

Behavioral evaluation will cover at least these cases:

1. A timer/listener-owning service with cleanup requirements: lifecycle is
   required.
2. A request-scoped export or backup helper: lifecycle is not justified.
3. A stateless business-command coordinator: complexity alone does not justify
   lifecycle.
4. A class created only to host stateless IPC handlers: report over-design.
5. A heavy runtime resource controlled by a preference: recommend
   `Activatable` only after establishing the runtime toggle.
6. A service needing coordinated temporary suspension: distinguish
   `Pausable` from `Activatable`.
7. A proposal with missing lifetime and cleanup information: return
   insufficient evidence rather than guessing.
8. A lifecycle proposal that modifies the generic container for one domain:
   identify entity leakage and keep domain behavior on the declaration side.

Validation checks observable decisions and evidence quality, not exact wording
or heading snapshots. The finished skill must also pass the available skill
frontmatter and structure validator.

## Non-Goals

- Implementing or refactoring Cherry Studio services.
- Replacing the general-purpose PR review skill.
- Reviewing renderer component lifecycle or React Effects in general.
- Designing domain entity state machines unless they directly affect
  main-process runtime-resource ownership.
- Automatically enforcing mechanical rules that belong in lint or static
  analysis.
