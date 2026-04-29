# AGENTS.md

Guidance for AI agents working in this project. Read at session start.

> **Scope:** this file is for *judgment* — how to make good calls when the answer
> isn't fully specified. Rules that must hold regardless of what you remember are
> enforced via hooks and permission settings, not here. Don't rely on this file
> alone for safety-critical guarantees.

## Communication
- **Ask before anything irreversible.** Commits, pushes, schema migrations, force-pushes, file deletes, production mutations: pause and confirm.
- **HITL for discovery; autonomous for well-defined execution.** Design and requirements work expects back-and-forth. Once a written plan is approved, execute through it without asking on each step.
- **Surface blockers; don't guess past them.** If you're uncertain after one or two attempts, stop and ask.

## Context discipline
- **Curate, don't pile on.** Don't accumulate context that doesn't bear on the current task.
- **Plan in one session, build in another.** If you find yourself doing both, stop and produce a written plan, then start fresh.
- **Flag context bloat.** If a session accumulates past the point of clear thinking, suggest a clean handoff.
- **Read workstream config before assuming defaults.** If a workstream override file is present (e.g., `CLAUDE.workstream.md`, `.workstream/config.md`), read it before using any hardcoded port, database, env var, or service URL. Workstream overrides take precedence over project defaults.

## Code and changes
- **Plan non-trivial work first.** Write the plan down before executing.
- **Check for an existing spec before inventing.** If there's a plan, design doc, or spec for what you're implementing, follow it. If you can't find one, ask before inventing an approach.
- **Don't reformat code you didn't touch.** When modifying an existing file, match the surrounding style exactly. Don't reformat lines you didn't intentionally change — it inflates diffs, masks the real change, and creates merge conflicts for no reason.
- **Every commit and PR should compile and run independently.** Don't leave the codebase in a broken intermediate state.
- **Update docs alongside code, not after.** README sections, ADRs, inline comments at non-obvious decision points, and the relevant AGENTS.md if behavior changes — all updated in the same commit as the code.

## Testing
- **Tests are part of the work, not cleanup.** Don't declare a task done with new code lacking tests.
- **Three workflows:** *new features* — write failing tests against acceptance criteria first, then implement until they pass; *bug fixes* — write a test that captures the bug first (it should fail in the same way the bug fails), then fix the code; *refactors* — pin current behavior with tests *before* changing anything, then make the change, then verify pinning tests stay green.
- **Run the full test suite before declaring a task done.** Not just the tests for the files you touched — the whole suite. Tests pass everywhere, or the task isn't done.
- **Push past unit tests for meaningful changes.** Integration tests, browser tests, container-based end-to-end loops — wherever the real failure modes live. Unit tests give a narrow window; real execution gives feedback you can act on.
- **When a test fails, fix the underlying code, not the test.** Unless the test itself is wrong (in which case explain why before changing it). Don't adjust expected values or skip tests to make them pass — that defeats the test.
- **Be aware of test environment limits.** Some tests need Docker socket access, real network egress, or external services that may not be available in your sandbox. Document or recognize the boundary; route the run to a tier that can actually execute it rather than failing silently.

## Tools and commands
- **Use the project's allow/deny lists.** If a command isn't on either, ask.
- **Don't disable security tooling** (secret scanning, pre-commit hooks, linters) to streamline work. The friction is doing its job.
- **Prefer scripts over agent loops for mechanical work.** If you find yourself doing the same sequence repeatedly, suggest extracting it. This holds even when an agent is orchestrating other agents — invoking a script is a few tokens; an agent re-deriving the same mechanics is hundreds or thousands, with variance instead of determinism.

## Reviews and PRs
- **Before opening a PR for a meaningful change, get a fresh-context review.** A different model is ideal but not required — the key property is clean context with no stake in prior decisions. Give the reviewer only the spec/plan/readme + the diff; ask "what's missing or wrong?" rather than "does this look good?"
- **Run the full test suite before opening the PR.** Failing tests don't go in the PR; they go on the to-fix list.
- **Do a documentation review pass before the PR.** Separate from the code review — different question: *does the doc accurately describe what now exists?* Check READMEs, ADRs, inline comments at non-obvious decision points, and the relevant AGENTS.md. Docs that drift from code are worse than no docs — they actively mislead.
- **Draft PR descriptions** that cover: goal, summary of changes, non-obvious decisions and why, what was explicitly *not* changed, link to spec/plan, open questions. A reviewer who can read your reasoning in 30 seconds is faster and more useful than one reconstructing it from the diff.
- **Keep the PR description current as the work evolves.** If review feedback prompts a meaningful change, follow-up commits adjust scope, or you defer something explicitly — update the description. A reviewer's second read shouldn't have to reconstruct what changed since the first.

## Phase handoffs
- **When you finish a phase, draft the next agent's prompt — don't carry into the next phase in the same session.** Planning → execution → tests → review → docs → PR are distinct phases; each wants a fresh-context agent loaded only with the artifacts it needs.
- **Suggest the next handoff explicitly.** When implementation is done, draft a prompt for a fresh test-writing or review agent (with only the spec + diff). When tests pass, draft a prompt for a review agent (with only the spec + diff, asking what's missing). The user decides whether to fire it themselves, dispatch it as a subagent if the platform supports it, or override your suggested prompt.
- **The clean handoff is the point.** Continuing into the next phase yourself defeats the separation that makes the work safer and more reliable.

## When something goes wrong
- **An agent failure is usually a docs/instructions gap, not a one-off problem.** If you didn't understand a package, missed a spec, or repeated a mistake, the fix is to update the docs, AGENTS.md, or shared instructions — not to silently work around it. Then verify the fix worked by running the same task in a fresh-context session.

## Multi-vendor / multi-agent
- **This file is canonical.** Other agent-specific files (CLAUDE.md, .cursorrules, etc.) should reference this one rather than duplicate. If you find them out of sync, flag it.

## Project-specific DO and DO NOT

Most of the rules above are generic. Add a per-project section here listing things specific to *this* codebase or workflow — judgment calls that need a written reminder. (Hard guarantees that must always hold belong in hooks or permission settings, not here.)

**DO:**
- (example) Use the project's `make test` target rather than calling pytest directly.
- (example) Read `docs/architecture.md` before touching the auth module.

**DO NOT:**
- (example) Don't add new top-level dependencies without asking.
- (example) Don't reformat files in `vendor/` or `node_modules/`.

Replace these examples with rules specific to your project. Empty bullets are a signal that the section hasn't been customized — fill it in or delete it.

## When in doubt
Ask. Asking is cheap; undoing isn't.
