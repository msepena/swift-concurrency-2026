---
description: Draft a feature spec file under `specs/` for a new feature. Takes a feature name and a high-level description, then produces a structured markdown spec covering the proposed branch name, prompt overview, acceptance criteria, edge cases, test cases, and any open questions. Pauses to ask clarifying questions before writing the file.
argument-hint: <feature-name> <high-level description...>
disable-model-invocation: true
allowed-tools: Bash(git *) Bash(mkdir *) Bash(ls *) Bash(test *) Read Write AskUserQuestion
---

# /spec — author a feature spec under `specs/`

## Inputs

- `$0` — feature name in kebab-case (required). Example: `dark-mode-toggle`. If the user types something like `Dark Mode Toggle`, normalize it: lowercase, replace whitespace/underscores with `-`, strip anything that isn't `[a-z0-9-]`.
- `$ARGUMENTS` from position 1 onward — the high-level description of the feature (required). Free-form prose.

If either is missing, do **not** stop — use `AskUserQuestion` to collect what's missing before continuing.

## Repo state (resolved before this prompt reaches Claude)

- Repo root: !`git rev-parse --show-toplevel 2>/dev/null || echo "(not a git repo)"`
- Current branch: !`git branch --show-current 2>/dev/null || echo "(detached or no repo)"`
- `specs/` exists: !`test -d specs && echo yes || echo no`
- Existing spec files: !`ls specs/*.md 2>/dev/null || echo "(none)"`

If the repo state shows "not a git repo", stop and tell the user this command needs to run inside a git repository.

## Procedure

1. **Resolve the feature name and description.**
   - Normalize `$0` to kebab-case as described under *Inputs*.
   - If the normalized name is empty, ask the user for it via `AskUserQuestion`.
   - If the description (`$ARGUMENTS` from position 1 onward) is empty or shorter than a sentence, ask the user to expand it.

2. **Refuse to clobber.** The target path is `specs/<feature-name>.md`. If it already exists, stop. Show the existing path and tell the user to rename, delete, or pick a different feature name — do not overwrite.

3. **Propose a branch name.** Default to `feature/<feature-name>`. Do **not** run `git checkout` — the user runs git themselves. Just record the suggested command in section 1 of the spec.

4. **Reason about the feature.** From the description, draft:
   - **Acceptance criteria** — concrete, testable bullets. Each should be something a reviewer could check off.
   - **Edge cases** — what breaks at boundaries (empty input, offline, concurrent updates, permissions, large data, etc.). Pick the ones that actually apply to this feature; don't pad with generic items.
   - **Test cases** — unit, integration, and UI-level cases that map back to the acceptance criteria. Name each case in one line.
   - **Open questions** — anything the description left ambiguous.

5. **Pause to resolve ambiguities.** Before writing the file, use `AskUserQuestion` to resolve the *blocking* open questions — the ones where you'd otherwise have to guess and the guess would materially change the spec. Keep this to one `AskUserQuestion` call with up to 4 questions. Non-blocking unknowns stay in the "Open questions" section of the file.

6. **Create the `specs/` directory if needed**, then write `specs/<feature-name>.md` with this exact structure:

   ```markdown
   # <Feature title (Title Case from the feature name)>

   ## 1. Branch

   Suggested branch: `feature/<feature-name>`

   Run when ready:

   ```sh
   git checkout -b feature/<feature-name>
   ```

   ## 2. Overview

   <One short paragraph paraphrasing the user's prompt — what the feature is and why. Quote any specific phrasing the user used verbatim where it's load-bearing.>

   ## 3. Acceptance criteria

   - [ ] <criterion 1>
   - [ ] <criterion 2>
   - ...

   ## 4. Edge cases

   - <edge case 1>
   - <edge case 2>
   - ...

   ## 5. Test cases

   - <test case 1>
   - <test case 2>
   - ...

   ## 6. Open questions

   - <unresolved question 1>
   - <unresolved question 2>

   (If everything was resolved during the clarification step, write "None — all clarifications resolved during spec creation.")
   ```

7. **Report to the user.** After writing, output:
   - **Spec:** the absolute path to the file just written.
   - **Branch (proposed):** `feature/<feature-name>` and the exact `git checkout -b` command.
   - **Next step:** a one-line suggestion (e.g., "Review the spec, then run the checkout command to start the feature branch.").
   - Do not print the full spec contents back — the user can open the file.

## Notes for Claude

- Do not run `git checkout`, `git branch`, or any state-mutating git command. The branch is a suggestion, not an action.
- Do not invent acceptance criteria the description doesn't support. If the description is too thin to produce concrete criteria, that's a signal to ask more questions in step 5 — not to pad the file.
- Keep the spec terse. Bullets, not paragraphs. The spec is a working document, not a design doc.
- If the user's description contains URLs, Figma links, ticket IDs, or file paths, preserve them verbatim in the Overview section.
- Use `Write` for the spec file, not Bash heredocs.
