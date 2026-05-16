# Tasks: lifecycle, cancellation, deadlines

`Task` is cheap to create but not free to forget about. The 2026 guidance from Apple's engineers reinforced a few rules that are easy to violate in practice.


## Cancellation is cooperative

A `Task` does not stop when something else "expects" it to. It stops when:

1. Someone calls `cancel()` on its handle, **and**
2. The task's body actually checks for cancellation — via `try Task.checkCancellation()`, by awaiting a cancellation-aware API (`URLSession`, `Task.sleep(for:)`, most `AsyncSequence` operations), or by checking `Task.isCancelled` at loop boundaries.

Long-running synchronous work (a tight `for` loop, an expensive parser) will run to completion even after cancellation. If the loop body can't be made async, insert explicit `Task.checkCancellation()` calls at loop boundaries.

**Review rules:**

- Flag long synchronous loops inside a `Task` body that have no cancellation check — they leak CPU after the owning view dismisses.
- Flag `await someAsyncCall()` followed by code that should not run after cancellation but doesn't check — use `try await` and let cancellation propagate, or `if Task.isCancelled { return }` before the side effect.


## Tasks are not auto-cancelled by scope

A `Task { ... }` created inside a method does not get cancelled when the method returns, nor when the surrounding object deallocates. **The owner must store the handle and cancel explicitly.** The Q&A called this out as the single most common cancellation bug.

Pattern:

```swift
@MainActor
final class FeedLoader {
    private var loadTask: Task<Void, Never>?

    func load() {
        loadTask?.cancel()
        loadTask = Task {
            // ...
        }
    }

    deinit {
        loadTask?.cancel()
    }
}
```

**The exception** is SwiftUI's `.task` modifier (covered in `swiftui-patterns.md`) — that one does cancel on view disappearance. Everywhere else, you own the handle or you've leaked the work.

**Review rules:**

- Flag fire-and-forget `Task { ... }` inside any type that has a meaningful lifecycle (view models, services, controllers) unless the work is genuinely "complete-or-discard, never cancel."
- Flag `deinit` on a type with stored `Task` handles that does not cancel them.


## `weak self` in task captures

Not a blanket rule. The Q&A's answer was nuanced:

- **Use `weak self`** when the task can outlive `self`, the work is long, and you want `self` to deallocate (typical: a `Task` started from `onAppear` in a view that may dismiss before the work finishes).
- **Skip `weak self`** when the task is short, you actively want it to complete before `self` deallocates (e.g., saving state on dismiss), or when cancellation is wired up properly so the task can't outlive `self` anyway.

The anti-pattern to flag: `[weak self] in guard let self else { return }` immediately at the top of a `Task { ... }` body that runs once and finishes in milliseconds. That's cargo-culted from closure-based GCD code and adds no value.


## `Task.detached` vs `Task { @concurrent in }`

`Task.detached` discards **everything** from the surrounding context — priority, task-local values, isolation. It is rarely what you want.

If your real intent is "I want to run this on the cooperative pool but keep priority and task-local values," use `Task { @concurrent in ... }` instead (see `isolation.md`).

**Review rules:**

- `Task.detached` should be load-bearing — paired with a comment explaining what context is being deliberately escaped. If there's no such reason, suggest `Task { @concurrent in }`.


## `withDeadline` (Swift Evolution, accepted)

An accepted proposal adds a deadline API to `Task` — when the deadline is exceeded, the task is cancelled.

**Naming caveat from the Q&A:** the API was called `withDeadline` at recording, but the team noted "the naming is still under debate." Before recommending it in code, check the current Swift Evolution proposal and the toolchain version targeted by the project. If the project pins to a toolchain that ships it, use it; otherwise, fall back to the well-known pattern of racing the work against `Task.sleep(for:)` in a `TaskGroup` and cancelling the loser.

**Review rules:**

- Flag custom deadline implementations (Sleep + cancel + flag) in projects targeting a toolchain that ships `withDeadline`. The native API is the right choice once available.
- Don't recommend `withDeadline` for projects on older toolchains.


## Async `defer` (Swift 6.4)

In Swift 6.4, the long-standing restriction on `async` calls inside `defer` blocks is lifted. You can `await` inside `defer` — useful for cleanup that genuinely needs to be async (closing an async resource, flushing a queue).

Don't migrate existing sync `defer` bodies to async just because you can. Async `defer` adds suspension points to the unwind path, which makes reasoning about timing harder.


## Swift 6.4 diagnostic for unhandled task errors

Swift 6.4 adds a diagnostic for a common bug: creating a `Task` with a throwing body and never reading the result, so thrown errors disappear silently.

If you see `Task { try await thing.doRiskyWork() }` and the result is discarded, in 6.4 the compiler will warn. Either:

- Use `Task` with explicit `try?` / `try!` inside the body to make the swallowing explicit, or
- Store the handle and read the result somewhere.


## What never to do

- **Never block an async function with `DispatchSemaphore`** to "wait for" an async result. The cooperative pool has a fixed number of threads; blocking one to wait for work that's queued on the same pool will deadlock. Refactor the caller to be async, or refactor the dependency to expose a synchronous entry point.
- **Never use `Task.sleep(nanoseconds:)`.** Use `Task.sleep(for: .seconds(1))`. (This rule is also in `swiftui-pro/references/swift.md`.)
- **Never replace a cancellation-aware `await` with a fire-and-forget wrapper** to "simplify the call site." You're discarding cancellation propagation.
