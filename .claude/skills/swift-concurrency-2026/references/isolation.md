# Isolation

Where code is allowed to run, and what mutable state it is allowed to touch. In 2026, the model has three new pieces worth knowing: MainActor-default app targets, `nonisolated(nonsending)`, and `@concurrent`.


## MainActor by default (Xcode 26)

In Xcode 26, new app targets are created with **MainActor isolation as the default**. This is the recommended posture for UI-shaped code — most of your view layer, view models, and lifecycle code is already main-thread by intent, and the new default makes that explicit instead of forcing `@MainActor` annotations on everything.

This is **not** the right default for libraries. Library targets should remain mostly `nonisolated` so callers can use them from any isolation domain. The Foundation team gave `UndoManager` as an example of a Foundation type that *is* `@MainActor` — that's the exception, not the rule. If you're writing a library, default to `nonisolated` and opt specific UI-only types into `@MainActor`.

**Review rules:**

- Flag `@MainActor` annotations on every type or method in an app target that already has MainActor-default isolation — they are redundant noise. Keep them only on types that explicitly need to document the boundary (e.g., a protocol conformance).
- Flag `MainActor.run { ... }` blocks where the surrounding context is already MainActor-isolated. Under the MainActor-default setting, those are almost always removable.
- In a library target, flag any `@MainActor` type whose purpose is not UI/view layer. Library types should justify their isolation, not assume it.


## `nonisolated`

`nonisolated` declares that a member has no preference for isolation — it can be called from any context. It does **not** mean "run on a background thread." It means "the compiler is not going to enforce any particular isolation here, so you'd better not touch isolated mutable state."

Use it for:

- Pure helpers on an actor or `@MainActor` type that don't read isolated state.
- Conformances to protocols whose requirements need to be callable from any isolation domain (e.g., `Hashable`, `Equatable`, `CustomStringConvertible`).
- Constants and `let` properties that are safe to read from anywhere.

**Review rules:**

- Flag `nonisolated` on a method that reads or writes the enclosing actor's mutable state — the compiler will catch this, but make sure the diagnostic is acted on rather than worked around with `MainActor.assumeIsolated`.
- Flag `nonisolated` paired with a synchronous body that calls an async API and discards the result with `Task { ... }` — that's usually a sign the caller wanted `async` propagation but couldn't get it.


## `nonisolated(nonsending)`

Newer variant that applies to **async functions**. The semantics: a `nonisolated(nonsending)` async function does **not** immediately hop to the shared cooperative executor when called. It stays in the calling isolation domain until something inside it forces a hop.

The motivation: a plain `nonisolated async` function used to imply "this will leave your isolation domain the moment you `await` it," which surprised people writing async code from MainActor — their suspended work would resume on a background thread even when nothing in the body needed it. `nonisolated(nonsending)` preserves the caller's isolation across the call.

**When to use:** async helpers that don't intrinsically need to run off the calling actor, but should still be callable from anywhere. Default to this for library async APIs unless you specifically *want* to offload.

**Review rules:**

- If a project enables the approachable-concurrency upcoming features (default in new Xcode projects), plain `nonisolated async` already adopts the `nonsending` behavior. Don't add the explicit attribute redundantly.
- If a project has *not* enabled approachable concurrency and is on the old semantics, calling a `nonisolated async` function from MainActor will hop off the main thread. That's often a latent bug — flag and recommend either enabling approachable concurrency or annotating explicitly.


## `@concurrent`

The 2026 way to say "run this on the cooperative thread pool." `@concurrent` is the **opt-in** to background execution — it inverts the old default where `nonisolated async` would silently offload.

Two described forms (verify exact syntax against the current Swift Evolution proposal — the Q&A described both but showed no code):

- On a method: `@concurrent func ...` — every call to this async method runs on the cooperative pool.
- On a task closure: `Task { @concurrent in ... }` — this specific task body runs on the pool, regardless of where it was created.

The compiler enforces the obvious safety property: inside a `@concurrent` body, you cannot directly access mutable MainActor (or any actor) state. You can `await` it, but you cannot touch it synchronously.

**When to use:** the work is CPU-bound or genuinely benefits from off-main execution, **and you've profiled to confirm**.

**Review rules:**

- Flag `@concurrent` annotations added without a paired Instruments justification, especially on a `@MainActor` view model's I/O methods. The Q&A explicitly warned against pre-emptive offloading: tasks created on MainActor already have suspension points that let UI render.
- Flag `Task.detached { ... }` where `Task { @concurrent in ... }` is intended. `Task.detached` discards **all** surrounding context (priority, task-local values, isolation). `Task { @concurrent in }` keeps task-local values and priority but only relinquishes isolation. The latter is almost always what people actually wanted.


## Custom actors

Apple's guidance: there's no hard cap on actor count, but every actor is an isolation boundary, and isolation boundaries are where data has to cross — which is where Sendable, hops, and complexity live. Reach for an actor when you have **genuine shared mutable state with multiple writers**: a cache, a database connection, a network session manager.

**Review rules:**

- Flag actors with no mutable stored properties — they should be a struct or a `nonisolated` type.
- Flag actors used as namespaces (only `static` members) — make it a `caseless enum`.
- Flag actors that are only ever called from MainActor — collapse them into `@MainActor` on a class.


## Custom executors

Most code never needs a custom executor. The default cooperative pool and MainActor's executor cover the common cases. Custom executors exist for advanced scenarios (e.g., binding an actor to a specific queue for compatibility with a legacy subsystem). If you see one being added, the bar is "we have a concrete reason the default executor cannot satisfy" — not "we'd like more control."


## `MainActor` guarantees

On Apple platforms, MainActor is the main thread. Always. The one caveat: C, Objective-C, and C++ interop can call into MainActor-isolated code from arbitrary threads, bypassing static checking. If you're writing a class that's bridged to Obj-C and might be called from a CFRunLoop, a CoreAudio callback, or a third-party C library, use `MainActor.assertIsolated()` at boundaries to catch violations at runtime instead of relying solely on the static checker.
