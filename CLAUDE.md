# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

A workspace for **authoring Claude Code skills**, not an application codebase. There is no build, lint, or test step. Each subfolder under `.claude/skills/` is a self-contained skill that auto-loads when Claude Code is invoked from inside this directory.

Skills here are **project-scoped** — they activate only when the working directory is `/Users/sepena/Documents/SkillsTest` (or below). To make a skill globally available across all projects, move or symlink its folder to `~/.claude/skills/`.

## Currently installed skills

- `.claude/skills/swift-concurrency-2026/` — review + authoring guidance for Swift concurrency, grounded in Apple's April 2026 "Meet with Apple" Q&A. Covers the post-Xcode-26 surface (`@concurrent`, `nonisolated(nonsending)`, MainActor-default app targets, Swift 6.4 changes). Read its `SKILL.md` before modifying.

## Skill structure convention

Every skill follows the same layout (mirrors `~/.claude/skills/swiftui-pro/`):

```
<skill-name>/
├── SKILL.md              # entry point with YAML frontmatter
└── references/           # topic-specific reference files (optional)
    └── *.md
```

`SKILL.md` frontmatter fields that matter:

- `name` — kebab-case slug, must match the folder name.
- `description` — **this is the trigger.** Future Claude instances see this string to decide whether to load the skill. Use concrete verbs ("reviews", "guides authoring") and specific surface area; vague descriptions miss their triggers.
- `metadata` (optional) — useful for `source`, `baseline` (toolchain/SDK version), `version`.

Reference files are loaded on demand via `${CLAUDE_SKILL_DIR}/references/<name>.md` paths inside `SKILL.md`. Keep references topically focused so a partial load is meaningful.

## When editing or adding skills

- **Don't duplicate guidance** that already lives in a neighboring skill — cross-reference instead. The user's `~/.claude/skills/swiftui-pro/references/swift.md` already covers timeless Swift concurrency rules (no GCD, `Task.sleep(for:)`, etc.); skills here should focus on whatever delta they add.
- **Ground claims in primary sources.** If a skill is built from a video, blog post, or session, record the URL and date in `metadata.source` and mark inferred syntax as "verify against current reference" where you couldn't ground it verbatim.
- **The folder name and `name:` field must match** — discovery breaks otherwise.
