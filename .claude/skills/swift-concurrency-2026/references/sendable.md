# Sendable, `sending`, and crossing isolation boundaries

`Sendable` is the type-level promise that a value can safely cross isolation boundaries. The 2026 surface has two tools that matter beyond the basic conformance: the `sending` keyword for one-shot ownership transfer, and the `SendableMetatype` protocol for generic APIs that need to constrain metatypes.


## The basic `Sendable` rules (recap)

- Value types composed entirely of `Sendable` members are automatically `Sendable`.
- Reference types must be `Sendable` explicitly. Either:
  - The class is `final`, all stored properties are immutable (`let`) and `Sendable`, and there is no inheritance — the compiler synthesises conformance.
  - Or you write `@unchecked Sendable` and accept responsibility for the invariants yourself.
- Actor types are implicitly `Sendable`.
- `@MainActor`-isolated classes are implicitly `Sendable` (their isolation gives the guarantee).
- Closures crossing isolation domains must be `@Sendable` (or, in newer code, take a `sending` parameter — see below).


## `@unchecked Sendable` — when it's acceptable

The Q&A's guidance: `@unchecked Sendable` is fine **when you can articulate the synchronization invariant in one sentence**. Acceptable examples:

- A legacy `NSObject` subclass that internally synchronizes everything with a lock.
- A wrapper around an immutable underlying value (e.g., a wrapper around a `URL` or a wrapper around `Data` that copies on init).
- A type that holds only `let` properties of `Sendable` types but inherits from a non-`Sendable` superclass (preventing synthesis).

Not acceptable:

- Silencing a compiler warning on a struct with mutable `var` fields because "we always access it from the same place." The compiler is telling you something real; "we always do X" is exactly the kind of invariant that breaks during the next refactor.

**Review rules:**

- Every `@unchecked Sendable` should have a one-line comment naming the synchronization mechanism. If there is no comment, flag it. If the comment is "we promise we never mutate this," consider whether the type should just be a struct or a `final class` with `let` properties.
- Flag `@unchecked Sendable` on view models — those should be `@MainActor`-isolated instead.


## `sending` parameters

`sending` is a parameter (or return) modifier that says: "the caller transfers exclusive ownership of this value to the callee at this call site." Once transferred, the caller can't touch the value any more. This lets you pass a non-`Sendable` value across an isolation boundary safely — because the type system has guaranteed there is no other reference.

Typical use: passing a `NSObject` subclass or other non-`Sendable` type into a `Task` body once, after which the caller no longer holds it.

```swift
func enqueue(_ work: sending Work) {
    Task {
        await processor.handle(work)
    }
}
```

**When to recommend `sending` instead of `@Sendable`/`@unchecked Sendable`:**

- The value is created at the call site and immediately handed off — there is no shared ownership.
- The type is genuinely not thread-safe, so making it `Sendable` would be a lie.

**Review rules:**

- Flag `@unchecked Sendable` on a type used in exactly one place where ownership is transferred — `sending` is the better fix.
- Flag closure parameters typed as `@escaping @Sendable () -> Void` where the closure is only ever invoked once — consider `sending () -> Void` if the project's toolchain supports it.


## `SendableMetatype`

For generic APIs that take a type (not a value) and need to use it across isolation domains — e.g., a generic decoder that takes `T.Type` and runs decoding on a background actor.

```swift
func decode<T: Decodable & SendableMetatype>(
    _ type: T.Type,
    from data: Data
) async throws -> T { ... }
```

Without `SendableMetatype`, passing `T.Type` to an actor or `@concurrent` body trips a Sendable diagnostic — the metatype itself has to cross. With `SendableMetatype`, the constraint is explicit and the compiler is satisfied.

**Review rules:**

- Flag generic async APIs that take a `T.Type` and produce diagnostics about non-Sendable metatypes — `SendableMetatype` is the right constraint to add.
- Don't add `SendableMetatype` speculatively. Add it when a constraint is needed.


## Isolated conformances

A protocol conformance can be isolated to an actor. For example, a type can conform to `Equatable` only when accessed from MainActor. When you try to pass such a value into a context that needs the unisolated conformance (e.g., into a sorted collection that runs comparison off-actor), the compiler will complain.

The fix is usually one of:

- Add `sending` to the parameter so the value is transferred (and the conformance is reachable in the new context).
- Constrain the API to `SendableMetatype` so the metatype itself can cross.
- Reconsider whether the isolated conformance is actually intentional — most `Equatable` and `Hashable` conformances should be `nonisolated`.


## NSObject and Obj-C interop

NSObject doesn't conform to `Sendable`. When you need to pass an `NSObject` subclass into a concurrent context, the options ranked by preference:

1. **Extract the Sendable properties.** Instead of passing the whole `NSPersistentStoreCoordinator`, pass the `URL` and the configuration you need.
2. **Use `sending`** if the object is created at the call site and handed off exactly once.
3. **Wrap in a `Sendable` adapter** that copies the values you need into a struct.
4. **`@unchecked Sendable` on a wrapper class** whose body is a `Lock`-protected reference. Last resort, requires the one-line invariant comment.


## Sendable closures and `@Sendable`

A `@Sendable` closure can be called from any isolation domain — meaning it cannot capture non-Sendable state by reference. Common pitfalls:

- Capturing a `weak self` where `self` is a non-Sendable class. The capture is allowed (it's a weak reference), but if the closure body then `await`s and calls into `self`, you've crossed an isolation boundary and need either `self` to be Sendable or to hop back onto its isolation domain.
- Capturing a local mutable `var` — the closure captures by reference, so the `var` must be Sendable, which a `var` of a non-Sendable type is not.

**Review rule:** flag `@Sendable` closures that do `self.someProperty = ...` without an explicit hop (`await MainActor.run { ... }` or similar) — that's a data race waiting to happen.
