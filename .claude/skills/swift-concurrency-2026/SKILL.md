---
name: swift-concurrency-2026
description: Reviews and guides authoring of Swift concurrency code — actor isolation (MainActor, nonisolated, nonisolated(nonsending)), @concurrent offloading, Task lifecycle and cancellation, Sendable conformance, SwiftUI .task integration, and Swift 5→6 migration. Grounded in Apple's April 2026 "Meet with Apple" Q&A. Use when reading, writing, or reviewing Swift code that uses async/await, actors, Tasks, or @MainActor.
license: MIT
argument-hint: "[focus area: isolation | tasks | sendable | swiftui | migration]"
metadata:
  source: "Q&A: Swift concurrency | Meet with Apple — youtube.com/live/E95agtPgaa0 (2026-04-28)"
  baseline: "Swift 6.2 / Xcode 26 / iOS 26"
  version: "1.0"
---

Review and author Swift concurrency code grounded in the guidance Apple's Foundation, SwiftUI, and compiler engineers gave in the April 2026 "Meet with Apple" Q&A. Focus on the 2026 surface area: `@concurrent`, `nonisolated(nonsending)`, MainActor-default app targets, the approachable-concurrency upcoming features, `withDeadline`, and Swift 6.4 changes.

Report only genuine problems. Do not invent issues, and do not nitpick stylistic choices the project has already settled on.


## When to use this skill

Load when the code or task involves any of:

- `async`/`await`, `Task { ... }`, `Task.detached`, `TaskGroup`, `AsyncSequence`
- Actor declarations, `@MainActor`, `nonisolated`, `nonisolated(nonsending)`, `@concurrent`
- `Sendable`, `@unchecked Sendable`, `sending`, `SendableMetatype`
- SwiftUI `.task` / `.task(id:)` modifiers, async button actions, async ViewModels
- Strict-concurrency migration, "approachable concurrency" upcoming feature flags, removing GCD

This skill is complementary to `swiftui-pro`. When reviewing a SwiftUI codebase end-to-end, run `swiftui-pro` for view/data/navigation/accessibility and run this skill for the concurrency layer. Do not duplicate `swiftui-pro/references/swift.md` rules (no GCD, prefer `Task.sleep(for:)`, etc.) — those still apply.


## Process

If `argument-hint` names a focus area, load only the matching reference. Otherwise, load all five before reviewing or designing:

1. Isolation — `${CLAUDE_SKILL_DIR}/references/isolation.md`
   MainActor default in Xcode 26, `nonisolated`, `nonisolated(nonsending)`, `@concurrent`, custom actors and executors.
2. Tasks — `${CLAUDE_SKILL_DIR}/references/tasks.md`
   Task creation, cooperative cancellation, `weak self`, `withDeadline`, async `defer`, Swift 6.4 diagnostics.
3. Sendable — `${CLAUDE_SKILL_DIR}/references/sendable.md`
   `Sendable`, `@unchecked Sendable`, `sending`, `SendableMetatype`, NSObject and Obj-C interop.
4. SwiftUI patterns — `${CLAUDE_SKILL_DIR}/references/swiftui-patterns.md`
   ViewModel design, `.task` cancellation semantics, async button taps, actor-to-view bridging.
5. Migration — `${CLAUDE_SKILL_DIR}/references/migration.md`
   Swift 5→6 path, approachable-concurrency upcoming features, GCD/`OperationQueue` displacement, async testing.


## Core principles

These are the recurring themes Apple's engineers emphasized. Treat them as load-bearing.

- **Profile before offloading.** Do not pre-emptively push work off MainActor or into background pools because it "feels expensive." Tasks created on MainActor include suspension points that allow UI to update. Move work off the main actor only after Instruments shows a real bottleneck.
- **Isolation is a design choice, not a default to be optimized away.** Limit isolation domains (actors) to genuine ownership boundaries — caches, database access, network sessions — not "anything that touches data."
- **Cancellation is cooperative.** Tasks do not auto-cancel when their enclosing scope deallocates (with the exception of SwiftUI's `.task` modifier on view disappear). The owner of a task must call `cancel()` — typically by storing the `Task` handle on a type and cancelling in `deinit` or on a lifecycle hook.
- **The compiler is the source of truth.** If the compiler says you have a data race or a Sendable violation, the compiler is right. Adding `@unchecked Sendable` or `MainActor.assumeIsolated` to silence it is a load-bearing decision that requires a written justification (you, the human, are vouching for the invariant).
- **Never block an async context with semaphores.** Risks deadlock against the cooperative thread pool. Refactor towards synchronous shutdown paths or async-native coordination.
- **MainActor is the main thread on Apple platforms** — but interop with C/Obj-C/C++ can bypass static checking. When bridging, prefer runtime checks (`MainActor.assertIsolated()`) at boundaries.


## Output format (review mode)

Organize findings by file. For each issue:

1. State the file and line(s).
2. Name the rule and which reference it comes from (e.g., "Unnecessary `@MainActor.run` under MainActor-default isolation — see `isolation.md`").
3. Show a brief before/after.

Skip files with no issues. End with a prioritized summary — group by severity:

- **Correctness (high):** data races, missing cancellation that leaks tasks, blocking semaphores in async code, `@unchecked Sendable` without justification.
- **API modernization (medium):** removable `MainActor.run`, `Task.detached` that should be `Task { @concurrent in }`, `nonisolated async` where `@concurrent` is intended.
- **Hygiene (low):** unnecessary `weak self` for short-lived tasks, redundant isolation annotations.


## Output format (authoring mode)

When generating new code, surface decisions inline (one line each) rather than producing them silently:

- Isolation choice: `@MainActor` / actor / `nonisolated` / `@concurrent` — and why.
- Cancellation strategy: stored `Task` handle / SwiftUI `.task` / `withDeadline` / none (and why none is safe).
- Sendable posture: which boundary crossings exist, and how each is satisfied.

Keep the rationale to one sentence per decision. The goal is auditability, not documentation.


## Source caveats

This skill is grounded in the 2026-04-28 Q&A summary at the Substack linked in `metadata.source`. Where Apple's answers were paraphrased rather than shown as code (the source has no runnable snippets), this skill marks the syntax as "as described, verify against current Swift Evolution." Three items in particular:

- `withDeadline` — naming was "still under debate" at recording. Verify against the latest Swift Evolution before recommending the exact spelling.
- `nonisolated(nonsending)` — semantics are stable; exact attribute placement should be verified against the Swift 6.2 reference.
- `@concurrent` — described as usable on both methods (`@concurrent func ...`) and task closures (`Task { @concurrent in }`); confirm against the language reference for your target toolchain.
