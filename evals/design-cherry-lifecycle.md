# Design Cherry Lifecycle evaluation cases

Run every case below with a fresh agent. Each case is independent: do not carry conclusions, terminology, or assumptions from another case into it.

Evaluate the agent on its observable architectural decision and the evidence it uses, not on exact wording. A response passes when it reaches the required classification, identifies the relevant ownership and lifecycle facts, and avoids the listed category errors.

## Case 1: Project index watcher

### Prompt

Cherry Studio is adding `ProjectIndexService`. When the application starts, the service builds an in-memory project index and starts a chokidar watcher. File events update the index. During application shutdown, pending index changes must be flushed and the watcher must be closed. Repository inspection has confirmed that no existing service owns the project index, watcher lifetime, or shutdown flush. The nearest watcher implementation is only a reusable primitive and does not own these responsibilities.

Should `ProjectIndexService` participate in the main-process Lifecycle system? Explain the decision and what repository evidence you would inspect before implementing it.

### Expected invariants

- Classifies the service as **Lifecycle required** because it owns process-long resources and persistent side effects: the watcher, mutable in-memory index, pending writes, and shutdown obligations.
- Connects startup and shutdown behavior to lifecycle hooks, including starting the watcher only at the appropriate phase and flushing pending changes before closing it.
- Treats ownership as a design question, not just registration mechanics: before recommending a new service, it explains that existing lifecycle owners were checked for the project index, watcher, and flush path and none owns them.
- Avoids introducing a second watcher or competing index owner and distinguishes the nearby watcher primitive from a component that owns its lifetime.
- Names concrete evidence to inspect, such as lifecycle documentation, the service registry, nearby watcher/index services, their tests, and current shutdown handling.

## Case 2: Per-export encoder

### Prompt

`ConversationExportService` exposes an `exportConversation()` operation. Each invocation creates an encoder, streams one export, and releases the encoder in that operation's `finally` path. The service keeps no watcher, timer, subscription, connection, cache, or other resource between calls. A developer proposes making it a lifecycle service because callers could then use `application.get()` and dependency injection.

Should this service participate in Lifecycle? Explain why or why not.

### Expected invariants

- Classifies the service as **not a Lifecycle service** because all resources are operation-scoped and released within each export.
- Distinguishes an object's lifetime from the lifetime of resources created by one method call.
- Rejects `application.get()` access or dependency-injection convenience as sufficient justification for Lifecycle participation.
- Recommends the repository's simpler non-lifecycle shape, such as a direct-import named singleton or a plain function/module, based on nearby conventions.
- Does not invent long-lived state, background cleanup, or speculative reuse to justify lifecycle registration.

## Case 3: Archive command coordinator

### Prompt

`AgentArchiveCoordinator.archiveAgent()` validates an agent, asks the existing `AgentRuntimeService` owner to stop the runtime, performs several database writes in one transaction, and asks the existing scheduler owner to remove scheduled work. The coordinator itself stores no entity state and owns no timer, runtime, queue, subscription, database connection, or other long-lived resource. A proposal says it should extend `BaseService` because it collaborates with several services and the ordering is important.

Should the coordinator participate in Lifecycle? Analyze its responsibility boundaries.

### Expected invariants

- Classifies the coordinator as **not a Lifecycle service**; the number of collaborators and the importance of call ordering do not establish resource ownership.
- Separately identifies four concerns: command orchestration, persisted entity state, transaction ordering/atomicity, and long-lived resource ownership.
- Keeps runtime lifecycle with the existing runtime owner and scheduled-work lifecycle with the existing scheduler owner.
- Keeps database consistency in the established transaction boundary and does not confuse transaction sequencing with application service lifecycle.
- Recommends a narrow command/coordinator abstraction only if it adds useful domain behavior; it does not introduce `BaseService` merely as a service locator or orchestration container.

## Case 4: Stateless system info handler

### Prompt

`SystemInfoService` would only register IpcApi handlers that return the application version and platform. The answers are stateless lookups. It owns no long-lived resource, mutable state, subscription, timer, or cleanup. The proposal wraps the handlers in a new lifecycle class and adds it to `serviceRegistry.ts` for consistency.

Assess the proposal and recommend the smallest fitting design.

### Expected invariants

- Identifies the proposed lifecycle class as **overdesign** for stateless app-version and platform lookups.
- Explains that uniform class shape or registry consistency is not evidence of a resource lifecycle.
- Recommends the smallest repository-consistent IpcApi handler/module placement, reusing an existing owner or namespace when one already fits.
- Preserves schema, handler, and security conventions at the IpcApi boundary without manufacturing start/stop behavior.
- Avoids adding lifecycle hooks whose only effect is registering trivial stateless handlers unless repository evidence shows an existing lifecycle owner is the required registration boundary.

## Case 5: Runtime-controlled local discovery

### Prompt

`LocalDiscoveryService` owns an mDNS browser. A runtime user preference can be toggled repeatedly: enabling it starts discovery, and disabling it releases the browser. The preference may change many times during one application session. A status IpcApi endpoint must remain registered and answer even while discovery is disabled. A developer is choosing between `@Conditional` and `Activatable`.

Which lifecycle pattern should the service use, and how should the responsibilities be split?

### Expected invariants

- Selects **Activatable**, not `@Conditional`.
- Explains that `@Conditional` controls whether the service exists or is registered at composition/startup time, while the requirement is repeatable runtime activation and deactivation.
- Keeps lightweight, always-needed control/status IPC registered for the service lifetime.
- Starts the mDNS browser in activation and fully releases it in deactivation, with repeated enable/disable transitions handled safely.
- Treats the preference as activation policy or input, not as justification to reconstruct the whole service or lose the status endpoint.

## Case 6: Backup pause and drain

### Prompt

`SyncSchedulerService` owns a scheduler and worker resources. Before a backup snapshot, it must stop accepting new sync jobs, let in-flight work drain, and then signal that the backup can proceed. Its resources and configuration should remain allocated during the backup. After the snapshot, normal scheduling resumes without reconstructing the service. The lifecycle system offers start/stop, activation/deactivation, and a pausing contract.

Which lifecycle capability fits, and what behavior must it guarantee?

### Expected invariants

- Selects **Pausable** for the backup window.
- Distinguishes pausing from stopping or deactivating: pause blocks new work and drains in-flight work while retaining the resources needed for an efficient resume.
- Requires pause completion to mean the drain is complete, so backup ordering can depend on that observable guarantee.
- Requires resume to reopen scheduling after the backup without rebuilding the service or silently dropping queued/in-flight work outside the defined policy.
- Keeps normal application startup/shutdown ownership separate from the temporary pause/resume protocol.

## Case 7: Underspecified model client

### Prompt

“`ModelClientService` caches a client and makes provider calls.” Decide whether it should use Cherry Studio's Lifecycle system and, if so, which lifecycle interfaces or decorators it needs.

### Expected invariants

- Returns **Insufficient evidence** instead of inferring lifecycle ownership from the word “cache” or from the `Service` suffix.
- Lists the missing facts needed for a decision, including what the client object owns, whether it holds connections/sockets/processes, cache scope and eviction, initialization cost, cleanup requirements, and whether state persists across calls.
- Asks whether configuration or credentials can change at runtime and whether that requires activation, restart, pause, or simple per-call reconstruction.
- Asks who currently owns provider clients, whether an existing service already manages them, and what startup/shutdown or error-recovery behavior is required.
- Does not prescribe `BaseService`, `Activatable`, `Pausable`, `@Conditional`, or service-registry changes until the ownership and transition evidence is available.

## Case 8: Entity-specific lifecycle engine branch

### Prompt

`LifecycleManager` currently contains a branch like `if (serviceName === 'AgentRuntimeService')` to handle agent-specific runtime behavior. A proposed change adds more special cases there because agent entities need custom start, stop, and restore ordering. Separately, `AgentRuntimeService` may own actual long-lived runtime processes.

Review the architecture. What is wrong, and what questions must be answered before changing the engine or the service?

### Expected invariants

- Identifies the `serviceName` branch as **domain/entity leakage into the generic lifecycle engine** and rejects adding more agent-specific branches there.
- Recommends expressing generic lifecycle contracts in the engine and keeping agent entity state, restore policy, and domain ordering in the appropriate agent-domain owner.
- Separately evaluates `AgentRuntimeService` resource ownership: if it truly owns long-lived processes, sessions, subscriptions, or shutdown cleanup, the service may legitimately participate in Lifecycle even though the engine must remain generic.
- Does not conclude that removing the engine special case also removes the runtime owner's lifecycle obligations.
- Requires evidence about who owns runtime resources, which transitions are generic versus agent-specific, current tests/contracts, and whether an existing domain coordinator can implement restore ordering without modifying the generic engine.

## Case 9: Platform-conditional native selection hook

### Prompt

`NativeSelectionHookService` is supported only on macOS. The platform is known before application startup and cannot change during a session. On macOS, the service installs a process-long native selection hook that must be closed during shutdown. On other platforms, the entire service should be excluded. This is a platform capability, not a runtime preference. Always-present consumers may expose features that optionally use the hook when it exists.

Should this capability participate in Lifecycle, and which decorator or interface should it use? Explain how consumers should access it when the condition excludes the service.

### Expected invariants

- Classifies the service as **Lifecycle required** because it owns a process-long native hook and mandatory shutdown cleanup.
- Selects `@Conditional(onPlatform('darwin'))` because the platform predicate is known before startup, evaluated once during composition, and immutable for the session.
- Does not model the platform as a runtime preference, `Activatable` toggle, or reason to keep an empty service registered on unsupported platforms.
- Excludes the entire service when the condition is false; always-present consumers use `application.getOptional('NativeSelectionHookService')` and handle absence explicitly.
- Avoids an unconditional `@DependsOn` from an always-present service to the conditional service, which would create dependency-resolution errors when excluded; a required consumer must share the same condition.
- Installs the hook in the appropriate startup hook, tracks its cleanup, and closes it during shutdown, including partial-initialization failure handling.
