# swift-concurrency-2026

A [Claude Code](https://claude.com/claude-code) toolkit for iOS work, organised as one **skill** (knowledge) and three **subagents** (roles that load the skill).

The skill is grounded in Apple's April 2026 *Meet with Apple* Q&A on Swift concurrency. It complements the existing [swiftui-pro](https://github.com/twostraws/swiftui-pro-agent-skill) skill — that one covers the timeless Swift and SwiftUI rules; this one covers the **post-Xcode-26 concurrency delta**: `@concurrent`, `nonisolated(nonsending)`, MainActor-default app targets, the approachable-concurrency upcoming features, `withDeadline`, and Swift 6.4 changes.

## Layout

```
.claude/
├── skills/swift-concurrency-2026/   ← knowledge
└── agents/                          ← roles that load the skill
    ├── ios-pr-reviewer.md
    ├── concurrency-migration-planner.md
    └── ios-architect.md
```

Knowledge stays in skills, roles stay in agents. Update the skill once and every agent picks up the change.

## The skill

When the skill triggers, Claude reviews Swift code that uses `async`/`await`, actors, `Task`, `@MainActor`, `Sendable`, or SwiftUI's `.task` modifier — and guides authoring of new concurrency code. It surfaces:

- **Correctness issues** — data races, missing cancellation that leaks tasks, blocking semaphores in async code, `@unchecked Sendable` without a stated invariant.
- **API modernization** — removable `MainActor.run`, `Task.detached` that should be `Task { @concurrent in }`, `nonisolated async` where `@concurrent` is intended.
- **Authoring decisions** — when to use which isolation, how to wire cancellation, how to bridge actors into SwiftUI views.

Five reference files load on demand:

| Reference | Topics |
|---|---|
| `isolation.md` | MainActor default in Xcode 26, `nonisolated`, `nonisolated(nonsending)`, `@concurrent`, custom actors and executors |
| `tasks.md` | Task lifecycle, cooperative cancellation, `weak self`, `withDeadline`, async `defer`, Swift 6.4 diagnostics |
| `sendable.md` | `Sendable`, `@unchecked Sendable`, `sending`, `SendableMetatype`, NSObject and Obj-C interop |
| `swiftui-patterns.md` | `.task` cancellation semantics, async button taps, view-model design, actor-to-view bridging |
| `migration.md` | Swift 5→6 path, approachable-concurrency upcoming features, GCD/`OperationQueue` displacement, async testing with Swift Testing |

Triggers, output formats, and core principles: [`SKILL.md`](.claude/skills/swift-concurrency-2026/SKILL.md).

## The agents

Three subagents covering the three modes a staff engineer works in — **verify**, **sequence**, **decide**. Each is read-only (no Write/Edit), pinned to opus, and preloads the relevant skills via the `skills:` frontmatter field.

| Agent | Mode | Spawn when… | Returns |
|---|---|---|---|
| [`ios-pr-reviewer`](.claude/agents/ios-pr-reviewer.md) | verify | "review this PR / branch / diff" for iOS Swift code | Findings by file, grouped by severity (correctness / API modernization / hygiene), with before/after snippets |
| [`concurrency-migration-planner`](.claude/agents/concurrency-migration-planner.md) | sequence | "plan the Swift 6 migration", "how should we adopt strict concurrency", "what order should we migrate" | Phased plan with goal / scope / risk / effort / exit criteria per phase |
| [`ios-architect`](.claude/agents/ios-architect.md) | decide | "how should I structure X", "should we use actor or @MainActor", "where should this state live" | 2-3 design options with explicit tradeoffs, recommendation last with an "I'd reconsider if" clause |

The agents are spawned automatically by Claude Code based on the user's prompt matching the agent's `description:` field. To force one, prefix the prompt with `@ios-pr-reviewer` (or the agent's name).

## Install

**Globally** — makes the skill and agents active in every Claude Code session:

```sh
git clone https://github.com/msepena/swift-concurrency-2026.git /tmp/swift-concurrency-2026
mkdir -p ~/.claude/skills ~/.claude/agents
cp -R /tmp/swift-concurrency-2026/.claude/skills/swift-concurrency-2026 ~/.claude/skills/
cp -R /tmp/swift-concurrency-2026/.claude/agents/*.md ~/.claude/agents/
```

**Per-project** — only active when working from that project:

```sh
mkdir -p /path/to/your/project/.claude/skills /path/to/your/project/.claude/agents
cp -R .claude/skills/swift-concurrency-2026 /path/to/your/project/.claude/skills/
cp -R .claude/agents/*.md /path/to/your/project/.claude/agents/
```

Claude Code discovers skills and agents on next session start. The agents reference the skill by name (`skills: swift-concurrency-2026, swiftui-pro` in their frontmatter), so install the skill alongside the agents — or the agents will fall back to running without it.

`swiftui-pro` is a separate skill referenced by `ios-pr-reviewer` and `ios-architect`. Install it from [github.com/twostraws/swiftui-pro-agent-skill](https://github.com/twostraws/swiftui-pro-agent-skill) for the full experience.

## Source and caveats

The skill is grounded in the Q&A session "[Q&A: Swift concurrency | Meet with Apple](https://www.youtube.com/live/E95agtPgaa0)" recorded **2026-04-28**, via a [formatted transcript on Substack](https://antongubarenko.substack.com/p/q-and-a-swift-concurrency-formatted).

The source contains no runnable code blocks — every code snippet in this skill was inferred from paraphrased descriptions in the transcript. The skill calls this out inline and recommends verifying three items against the current Swift Evolution proposals or Swift language reference:

- `withDeadline` — naming was "still under debate" at recording.
- `nonisolated(nonsending)` — exact attribute placement.
- `@concurrent` — described as usable on both methods and task closures; confirm against your target toolchain's reference.

Apple's official documentation is the source of truth; this skill is an opinionated, staff-engineer-level reading of what Apple's Foundation and SwiftUI teams emphasized in that one session. Compiler diagnostics override skill guidance.

## License

MIT.
