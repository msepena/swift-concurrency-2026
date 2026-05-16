# swift-concurrency-2026

A [Claude Code](https://claude.com/claude-code) skill for reviewing and authoring Swift concurrency code, grounded in Apple's April 2026 *Meet with Apple* Q&A session on Swift concurrency.

It complements the existing [swiftui-pro](https://github.com/twostraws/swiftui-pro-agent-skill) skill — that one covers the timeless Swift and SwiftUI rules; this one covers the **post-Xcode-26 concurrency delta**: `@concurrent`, `nonisolated(nonsending)`, MainActor-default app targets, the approachable-concurrency upcoming features, `withDeadline`, and Swift 6.4 changes.

## What it does

When triggered, the skill reviews Swift code that uses `async`/`await`, actors, `Task`, `@MainActor`, `Sendable`, or SwiftUI's `.task` modifier — and guides authoring of new concurrency code. It surfaces:

- **Correctness issues** — data races, missing cancellation that leaks tasks, blocking semaphores in async code, `@unchecked Sendable` without a stated invariant.
- **API modernization** — removable `MainActor.run`, `Task.detached` that should be `Task { @concurrent in }`, `nonisolated async` where `@concurrent` is intended.
- **Authoring decisions** — when to use which isolation, how to wire cancellation, how to bridge actors into SwiftUI views.

## What it covers

Five reference files load on demand:

| Reference | Topics |
|---|---|
| `isolation.md` | MainActor default in Xcode 26, `nonisolated`, `nonisolated(nonsending)`, `@concurrent`, custom actors and executors |
| `tasks.md` | Task lifecycle, cooperative cancellation, `weak self`, `withDeadline`, async `defer`, Swift 6.4 diagnostics |
| `sendable.md` | `Sendable`, `@unchecked Sendable`, `sending`, `SendableMetatype`, NSObject and Obj-C interop |
| `swiftui-patterns.md` | `.task` cancellation semantics, async button taps, view-model design, actor-to-view bridging |
| `migration.md` | Swift 5→6 path, approachable-concurrency upcoming features, GCD/`OperationQueue` displacement, async testing with Swift Testing |

The full set of triggers, output formats, and core principles is in [`SKILL.md`](.claude/skills/swift-concurrency-2026/SKILL.md).

## Install

**Globally (recommended, makes the skill active in every Claude Code session):**

```sh
git clone https://github.com/msepena/swift-concurrency-2026.git /tmp/swift-concurrency-2026
mkdir -p ~/.claude/skills
cp -R /tmp/swift-concurrency-2026/.claude/skills/swift-concurrency-2026 ~/.claude/skills/
```

**Per-project (only active when working from that project):**

```sh
mkdir -p /path/to/your/project/.claude/skills
cp -R .claude/skills/swift-concurrency-2026 /path/to/your/project/.claude/skills/
```

Claude Code discovers skills on next session start.

## Source and caveats

The skill is grounded in the Q&A session "[Q&A: Swift concurrency | Meet with Apple](https://www.youtube.com/live/E95agtPgaa0)" recorded **2026-04-28**, via a [formatted transcript on Substack](https://antongubarenko.substack.com/p/q-and-a-swift-concurrency-formatted).

The source contains no runnable code blocks — every code snippet in this skill was inferred from paraphrased descriptions in the transcript. The skill calls this out inline and recommends verifying three items against the current Swift Evolution proposals or Swift language reference:

- `withDeadline` — naming was "still under debate" at recording.
- `nonisolated(nonsending)` — exact attribute placement.
- `@concurrent` — described as usable on both methods and task closures; confirm against your target toolchain's reference.

Apple's official documentation is the source of truth; this skill is an opinionated, staff-engineer-level reading of what Apple's Foundation and SwiftUI teams emphasized in that one session. Compiler diagnostics override skill guidance.

## License

MIT.
