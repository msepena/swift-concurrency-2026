# SwiftUI integration patterns

The Q&A spent significant time on the seam between SwiftUI views and async work. The patterns below were endorsed by Apple's SwiftUI team.


## `.task` modifier — what it cancels, what it doesn't

The `.task` modifier attaches a task to a view's lifetime:

- **Cancelled on view disappear.** This is the one place in the framework where you get auto-cancellation.
- **Not cancelled on body re-evaluation.** If state changes cause `body` to re-run but the view containing the `.task` stays in the tree, the task keeps running. This is intentional.
- **Cancelled when the view is structurally removed** — e.g., the conditional `if showFoo { FooView() }` flips to false and `FooView` is destroyed.

Use `.task(id:)` when you specifically want the task to restart on a value change.

**Review rules:**

- Flag uses of `onAppear { Task { ... } }` for view-lifetime work — `.task` does it correctly and gives you cancellation for free.
- The exception: if you need to **store the `Task` handle** for explicit cancellation control (e.g., the same view also has a "Cancel" button that should stop the task mid-flight without dismissing), `.task` won't work because it doesn't expose the handle. Use `onAppear` + stored handle in that case. The Q&A called this out specifically.


## Async button taps

The endorsed pattern: **move async work to a model that exposes synchronous outputs to the view**. The view's button action calls a synchronous method on the model; the model owns the `Task`.

```swift
@MainActor
@Observable
final class CheckoutModel {
    private(set) var state: State = .idle
    private(set) var purchaseTask: Task<Void, Never>?

    func purchase() {
        purchaseTask?.cancel()
        purchaseTask = Task {
            state = .loading
            do {
                try await api.purchase()
                state = .success
            } catch is CancellationError {
                state = .idle
            } catch {
                state = .failed(error)
            }
        }
    }
}

struct CheckoutButton: View {
    @State private var model = CheckoutModel()
    var body: some View {
        Button("Buy", action: model.purchase)
            .disabled(model.state == .loading)
    }
}
```

The Q&A's stated reason: "you can also test the asynchronous work outside of your view." Pulling the async into the model makes it testable without instantiating a view hierarchy.

**Review rules:**

- Flag `Button("...") { Task { try await ... } }` patterns where the same work is invoked from multiple views or needs cancellation. Move to a model.
- For one-off, fire-and-forget actions in a leaf view (e.g., logging an analytics event on tap), inline `Task` is fine.


## URLSession cancellation in `.task`

SwiftUI's `.task` modifier will cancel the wrapped task on disappear, but it does **not** expose the task handle, so you can't cancel "just the URLSession data task" without dismissing the view.

`URLSession`'s async APIs (`data(for:)`, `data(from:)`, etc.) are **cancellation-aware** — when the surrounding task is cancelled, the network request is cancelled. So in practice, `.task { await loadFromNetwork() }` does cancel the network on disappear, as long as `loadFromNetwork()` is using `URLSession`'s async API.

If you need to cancel mid-request without dismissing (e.g., a "stop loading" button), drop down to `onAppear` + stored handle, and call `cancel()` on the stored task. That cancellation will propagate into `URLSession`.


## Bridging actors to SwiftUI views

A `View` is `MainActor`-isolated. An `actor` is not. There is no way to read an actor's property synchronously from a view — every cross has to be an `await`, which means it's an async boundary, which means it needs a `Task` or a `.task`, which means it has to land in `@State` or an `@Observable` model before the view can render it.

The Q&A acknowledged this is "almost inevitable" — there is no SwiftUI magic that lets a view subscribe to an actor's state directly. The right structure:

- Actor owns the source of truth.
- `@Observable` `@MainActor` view model holds a snapshot of what the view needs.
- View model has an `update()` async method that `await`s the actor and updates its own properties.
- View calls `update()` from `.task` or in response to user actions.

**Review rules:**

- Flag patterns that try to read an actor's properties from a view body via `Task { ... }` and a local `@State` variable assigned inside the task — this is correct but it's reinventing the view-model pattern badly. Pull it into an `@Observable` model.


## Heavy work on a `@MainActor` view model

Default position: don't pre-emptively offload. Tasks created on MainActor include suspension points. Profile first.

When profiling shows actual main-thread pressure (frame drops, hitches), the lever is `@concurrent` on the specific method that does the heavy work. The compiler will then prevent you from touching MainActor state inside that method without an explicit hop — that's the correctness guarantee.

```swift
@MainActor
@Observable
final class GalleryModel {
    private(set) var thumbnails: [Thumbnail] = []

    @concurrent func generateThumbnails(from urls: [URL]) async -> [Thumbnail] {
        // CPU-bound image work, runs on cooperative pool.
        // Cannot touch self.thumbnails here — compiler enforces.
    }

    func refresh(urls: [URL]) {
        Task {
            let new = await generateThumbnails(from: urls)
            thumbnails = new // back on MainActor here
        }
    }
}
```

(Verify `@concurrent` syntax against the toolchain — the Q&A described it but did not show code.)


## What never to do in a SwiftUI context

- **Never use `DispatchQueue.main.async { ... }` inside a view or view model.** The `swiftui-pro` rule already covers this; flag it.
- **Never start a `Task` in `init`** of a view model unless you store and cancel the handle in `deinit`. View models are recreated on view identity changes; orphaned init-time tasks accumulate.
- **Never block on a semaphore from a SwiftUI action.** The action runs on MainActor; blocking MainActor freezes the UI.
- **Never assume `.task` will re-run on every body re-evaluation.** It won't. Use `.task(id:)` if you need restart-on-change semantics.
