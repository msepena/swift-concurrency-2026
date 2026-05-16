# Migration: Swift 5 → 6, GCD → structured concurrency

The Q&A's framing: migration is incremental, target-by-target, file-by-file. There is no "flip the switch" answer that's responsible on a real codebase.


## Approachable concurrency upcoming features

New Xcode projects enable a set of upcoming feature flags collectively referred to as **"approachable concurrency."** These flags change a few defaults:

- `nonisolated async` adopts `nonisolated(nonsending)` semantics by default — async calls stay in the caller's isolation domain unless explicitly offloaded.
- Diagnostics around isolation are friendlier and point at the actual decision point rather than the deepest type-system root cause.
- (Other flags in this bundle change over time; verify against the current Swift Evolution proposal for the toolchain in use.)

**If your project does not have approachable concurrency enabled:** plain `nonisolated async` calls from MainActor still silently hop off the main thread. That's a footgun the new defaults exist to fix. Enabling the upcoming feature flags is typically the first migration step.

**Review rules:**

- In a Swift 6 project without approachable concurrency enabled, every `nonisolated async` declaration is a potential silent hop. Don't review concurrency in such a project without flagging this first — it's a blast radius issue.
- In a project *with* approachable concurrency enabled, plain `nonisolated async` is fine. Don't double-annotate with `nonisolated(nonsending)`.


## Swift 5 → Swift 6 incremental path

The Q&A's recommended sequence:

1. **Enable approachable concurrency upcoming features.** Establishes the new semantic baseline.
2. **Run the migration tooling** (Xcode's concurrency migrator). It will offer fix-its for many sites; accept the obvious ones.
3. **Adopt `@concurrent` for existing async functions** that were relying on the old "auto-hop" behavior of `nonisolated async`. This preserves the old behavior explicitly rather than silently.
4. **Enable strict concurrency checking incrementally** — by category (`-strict-concurrency=targeted` then `=complete`) and by target. Don't flip the whole monorepo at once.
5. **Address each diagnostic on its merits.** A diagnostic is information, not a bug. Sometimes the fix is `Sendable` conformance; sometimes it's `sending`; sometimes it's restructuring to put state behind an actor; sometimes it's `@unchecked Sendable` with a written invariant.

**Review rules:**

- Flag broad use of `MainActor.assumeIsolated` across a migration PR — it's the migration equivalent of `@unchecked Sendable` and needs justification.
- Flag wholesale moves from `class` to `actor` during migration. Actors are a design decision, not a migration shortcut.


## GCD and OperationQueue displacement

Apple's position is unambiguous: **migrate away from GCD and `OperationQueue` toward structured concurrency**. The Q&A's nuance:

- **Migrate target by target**, not file by file. A target that is partly GCD and partly async makes bridging harder, not easier.
- **New code adopts structured concurrency.** No new `DispatchQueue.global().async { ... }` even in a target that still has legacy GCD.
- **Prioritize hot paths** — code that's frequently modified or central to the architecture. Cold maintenance code can stay on GCD until there's a reason to touch it.
- **Don't refactor working legacy code just to modernize it.** The bar is "this code is causing a problem" or "I'm in here anyway for another reason."

The simple bridge primitives:

- `DispatchQueue.global().async { ... }` → `Task { @concurrent in ... }` or `Task.detached { ... }` (the former is usually right).
- `DispatchQueue.main.async { ... }` from a non-MainActor context → `await MainActor.run { ... }`, or restructure so the caller is already on MainActor.
- `OperationQueue` with dependencies → `TaskGroup` for sibling concurrency, structured `await` for sequence.
- `DispatchSemaphore` waiting on async work → don't. There is no equivalent. Refactor.


## Testing async code

The endorsed framework is **Swift Testing**. Test functions can be `async` and can be annotated with `@MainActor` (or any other actor) for isolation.

```swift
import Testing

@Suite("CheckoutModel")
struct CheckoutModelTests {
    @Test @MainActor
    func purchase_setsLoadingThenSuccess() async throws {
        let api = MockAPI(purchaseResult: .success(()))
        let model = CheckoutModel(api: api)

        model.purchase()
        #expect(model.state == .loading)

        await model.purchaseTask?.value
        #expect(model.state == .success)
    }
}
```

Notes:

- Async tests don't need any special setup beyond `async` on the function.
- For tests of MainActor types, annotate the test method (or the whole suite) with `@MainActor`. The test runner handles the hop.
- For tests of actors or non-MainActor types, don't annotate — the test will hop into the actor on `await`.
- For deterministic timing, prefer awaiting a known signal (a state change, a completion handle) over `Task.sleep`. Sleeps are the source of most flaky concurrency tests.

**Review rules:**

- Flag XCTest-style tests of async code (`expectation(description:)` + `fulfill()`) being added in 2026. Migrate to Swift Testing.
- Flag `Task.sleep` used as a synchronization primitive in tests.


## Migration anti-patterns

- **Mass `@MainActor` annotation as a migration shortcut.** Symptoms: every type in a target marked `@MainActor` to silence Sendable diagnostics, no actual reason for the annotation. Recommend reverting and addressing diagnostics individually.
- **`@unchecked Sendable` as a search-and-replace.** Each instance is a load-bearing assertion; each one needs the one-line invariant comment.
- **Wrapping every legacy call in a `Task`.** Loses cancellation, loses ordering guarantees, scatters task ownership.
- **`MainActor.assumeIsolated` everywhere.** Tells the compiler "I know better" — which is true sometimes (interop boundaries) and false most of the time (poor design).
