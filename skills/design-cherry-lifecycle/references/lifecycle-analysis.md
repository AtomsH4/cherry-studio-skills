# Cherry Studio lifecycle analysis

## Decision principle

Lifecycle is an ownership mechanism for runtime resources and persistent side effects across startup, activation, pause/resume, restart, shutdown, and recovery. It is not a reward for complexity and not a general class or dependency-injection convention.

Base conclusions on observed contracts and behavior. Do not infer a defect from naming, style, or the absence of an abstraction. Mark a statement as either:

- **Correctness:** required to prevent a concrete contract violation, leak, race, data loss, inaccessible command, or unrecoverable state.
- **Tradeoff:** a maintainability, consistency, performance, or extensibility choice whose alternatives can still be correct.

## Verdicts

Return exactly one of these three verdicts; no other verdict labels are allowed.

| Verdict | Use when |
| --- | --- |
| `Lifecycle required` | Evidence identifies a surviving runtime resource or persistent global effect that needs an explicit owner across real lifecycle transitions, and an ordinary function, direct singleton, or existing owner cannot correctly own it. |
| `Lifecycle not justified` | The capability is request-scoped or stateless, or its actual resources and effects are already correctly owned elsewhere. |
| `Insufficient evidence` | Available evidence does not establish the resource, owner, lifetime, ordering, transitions, cleanup, or recovery facts needed to choose either of the other verdicts. |

No explicit surviving resource or persistent global effect means lifecycle has not been proven. Complicated orchestration, many methods, or a large dependency graph also does not prove lifecycle. Absence of evidence must not be converted into a default direct singleton recommendation; use `Insufficient evidence` when missing facts could change the result.

## Evidence inventory

Build this inventory from current repository documentation, code, call sites, and tests. Distinguish confirmed evidence from inference.

| Question | Evidence to find |
| --- | --- |
| Surviving resources | Handles, listeners, watchers, timers, workers, processes, sockets, queues, window-owned objects, or other state that outlives one call. |
| Persistent global effects | IPC registration, protocol/global handlers, subscriptions, process hooks, scheduled work, or other effects that remain installed after a request returns. |
| Creator and current owner | The code that creates each resource/effect and the component currently responsible for its lifetime. |
| Cleanup and partial initialization | How every acquired resource is released, including rollback when initialization succeeds only partway. |
| First valid consumer and ordering | The earliest real consumer, when the dependency becomes valid, and what ordering that consumer actually requires. |
| Repeatable transitions | Whether start/stop, activate/deactivate, pause/resume, or restart can repeat, and the required idempotency or re-entry behavior. |
| Shutdown and drain | Whether in-flight work is joined, drained, cancelled, timed out, or deliberately abandoned before process exit. |
| Startup recovery | Which durable or externally visible state must be reconciled after a crash or interrupted operation. |

If a required row cannot be established, record the precise missing fact in `Open evidence`. Do not fill gaps with a likely architecture.

## Identify the actual owner

Keep these responsibilities separate unless evidence requires the same owner:

- **Data owner:** owns durable records and data invariants.
- **Command orchestrator:** sequences a request or workflow; it may be a function or stateless singleton with no lifecycle.
- **Domain-entity state:** a job, download, session, or task's business state; its lifecycle is not automatically a service lifecycle.
- **Runtime-resource owner:** owns process-local resources or persistent effects that survive calls; this is the lifecycle candidate.

Entity lifecycle is not service lifecycle. A coordinator may remain lifecycle-free. A generic engine should remain domain-blind; do not add product-domain branches merely to house ownership elsewhere.

## Decision ladder

Prefer the smallest correct owner, in this order:

1. **Ordinary function:** request-scoped work with no surviving resource or effect.
2. **Direct-import singleton:** shared stateless logic or bounded in-memory coordination whose construction and cleanup do not require application lifecycle ordering.
3. **Existing lifecycle owner:** a resource/effect naturally belongs to an existing service and adding it does not split or blur that owner's contract.
4. **New lifecycle service:** a distinct runtime-resource owner has independently meaningful lifecycle transitions that the earlier options cannot correctly provide.

Reject lifecycle when the only rationale is:

- access through `application.get()`;
- stateless IPC request handling;
- a uniform class shape;
- speculative future reuse;
- centralizing unrelated dependencies or IPC routes;
- matching a nearby service's style without matching its resource ownership.

## Smallest justified lifecycle shape

Choose only capabilities supported by observed transitions.

| Shape | Minimum justification |
| --- | --- |
| Ordinary `BaseService` | The service owns a surviving resource or persistent effect that must start and stop with the application. |
| `@Conditional` | A platform, architecture, or environment condition is known before startup, evaluated once during composition, and immutable for the session; when false, it excludes the entire service. Do not use it for a runtime preference or toggle. |
| `Activatable` | Any repeatable runtime toggle, even for a lightweight resource, or an on-demand heavy resource requires acquire/release while the service and its IPC surface remain resident. Acquire resources in activation and fully release them in deactivation. |
| `Pausable` | Execution must stop temporarily while preserving the same service instance and its owned resources for resume; define admission and drain behavior without reconstructing or deactivating the service. |

Do not add optional interfaces for transitions the capability does not have. Empty hooks are evidence that the chosen lifecycle shape may be too large.

### Phase and dependency rules

- Derive phase from the earliest real consumer and the first moment all required inputs are valid.
- Do not default to `BeforeReady`. Earlier startup increases coupling and reduces available dependencies.
- Declare `@DependsOn` only for required same-phase ordering. Phase ordering already handles cross-phase availability; redundant dependencies obscure the real graph.
- IPC availability and resource readiness are separate questions. A command surface may be registered before heavy resources activate, provided its observable unavailable-state contract is explicit.

### Resource and transition rules

- Track every listener, timer, subscription, worker, process, handle, or cleanup callback as a disposable owned by the service.
- Resources acquired during activation belong to activation scope and must be released on deactivation, not only final shutdown.
- Define repeated start/stop, activate/deactivate, pause/resume, and restart behavior where those transitions exist. Cover partial initialization and cleanup after failure.
- Treat `onAllReady` fire-and-forget work as owned background work: capture failures and ensure shutdown joins it, drains it, cancels it, or bounds the wait explicitly.
- For queued or in-flight work, define admission closure, completion/cancellation policy, drain bounds, and what remains durable after exit.

## Boundary rules

- **Domain entities:** A record's status transitions do not justify a process lifecycle service. Locate the process-local resource owner separately.
- **Coordinator:** Complex orchestration may remain a function or direct singleton when it owns no surviving resource or persistent effect.
- **Data and transactions:** Durable business state belongs behind the repository's data boundary. Keep atomic durable transitions in the data layer's transaction mechanism; lifecycle hooks do not replace transactions.
- **IpcApi placement:** Imperative non-data commands belong on the command IPC boundary. IPC registration alone does not justify lifecycle unless the owner has a persistent registration/effect contract requiring managed cleanup.
- **Durable commit and post-commit effects:** Commit durable truth atomically before notifications or external side effects that depend on it. Define retry or reconciliation for a crash between commit and the post-commit effect.
- **Renderer state:** Renderer convenience state is not authoritative for main-process resource ownership or durable recovery.
- **Generic engines:** Keep reusable engines domain-blind. Domain policy, persistence mapping, and product-specific branching belong at an adapter or orchestrator boundary.
- **WindowManager, paths, jobs, and tests:** Load and apply their repository documentation only when the proposed owner crosses those boundaries; do not invent parallel lifecycle mechanisms.

## Over-design assessment

Check the proposal for each of these signals and explain any that apply:

- empty lifecycle hooks;
- request-scoped work wrapped in a service;
- a dependency-injection or IPC bucket with no cohesive owned resource;
- ownership split away from an existing natural owner;
- redundant phase dependencies;
- optional interfaces with no real transition;
- speculative abstractions or future-use configuration;
- domain branches added to a generic engine.

An over-design concern is a tradeoff unless it creates a concrete correctness failure. Conversely, a stylistic mismatch is not a correctness issue by itself.

## Required design detail

When the verdict is `Lifecycle required`, specify the minimum design necessary to evaluate implementation:

- owner and architectural placement;
- phase and only the required same-phase dependencies;
- hooks, activation scope, and every tracked disposable;
- public shape and narrow responsibility;
- IPC availability versus underlying resource readiness;
- partial initialization rollback and repeated-transition behavior;
- shutdown, admission closure, cancellation, join/drain policy, and time bound;
- startup reconciliation for durable versus runtime state;
- focused behavioral tests that would catch leaks, ordering errors, repeated-transition failures, partial-init leaks, shutdown loss, or recovery gaps.

For either other verdict, keep `Recommended lifecycle design / Not applicable` short and explain what owns the work instead, or which evidence is required before a design can be made.

## Output contract

Use these sections in this order and do not add substitutes:

### Verdict + confidence + rationale

State one allowed verdict, a calibrated confidence level, and the decisive resource-ownership reasoning. Do not overstate confidence when repository evidence is incomplete.

### Evidence

Summarize the evidence inventory, citing repository paths and symbols when available. Label inference as inference.

### Over-design assessment

Report applicable signals and classify each conclusion as correctness or tradeoff.

### Alternatives considered

Compare, in order, an ordinary function, direct-import singleton, existing lifecycle owner, and new lifecycle service. Explain why each smaller option is sufficient or insufficient.

### Recommended lifecycle design / Not applicable

For `Lifecycle required`, provide the required design detail above. Otherwise state `Not applicable` and identify the current or likely smallest owner without promoting an unproven choice to a recommendation.

### Implementation outline (do not edit or execute)

Give only a non-executable outline of responsibility boundaries, likely change surfaces, and focused verification. Do not edit files, supply a command sequence, write a patch, or begin execution.

### Open evidence

List every missing fact that could change the verdict, phase, dependency graph, lifecycle shape, shutdown behavior, or recovery design. If none remain, state that explicitly. Missing decisive facts require `Insufficient evidence` rather than a guessed architecture.
