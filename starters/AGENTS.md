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

## Code and changes
- **Plan non-trivial work first.** Write the plan down before executing.
- **Pin before you modify.** For changes to existing code that lacks coverage, write tests that capture current behavior; verify they pass; then make the change.
- **Push past unit tests** when the real question is "does this actually work" — integration, browser, or container loops give better feedback.
- **Update docs alongside code.** Not after.
- **Don't reformat code you didn't touch.** When modifying an existing file, match the surrounding style exactly. Don't reformat lines you didn't intentionally change — it inflates diffs, masks the real change, and creates merge conflicts for no reason.
- **Every commit and PR should compile and run independently.** Don't leave the codebase in a broken intermediate state. A reviewer or future CI run should be able to check out any commit on the branch and have it work.
- **Check for an existing spec before inventing.** If there's a plan, design doc, or spec for what you're implementing, follow it. If you can't find one, ask before inventing an approach — there may be one you haven't seen.

## Tools and commands
- **Use the project's allow/deny lists.** If a command isn't on either, ask.
- **Don't disable security tooling** (secret scanning, pre-commit hooks, linters) to streamline work. The friction is doing its job.
- **Prefer scripts over agent loops for mechanical work.** If you find yourself doing the same sequence repeatedly, suggest extracting it. This holds even when an agent is orchestrating other agents — invoking a script is a few tokens; an agent re-deriving the same mechanics is hundreds or thousands, with variance instead of determinism.

## Reviews and PRs
- **Before opening a PR for a meaningful change**, get a fresh-context review.
- **Draft PR descriptions** that cover: goal, summary, non-obvious decisions, what's *not* changed and why, link to spec/plan, open questions.

## Multi-vendor / multi-agent
- **This file is canonical.** Other agent-specific files (CLAUDE.md, .cursorrules, etc.) should reference this one rather than duplicate. If you find them out of sync, flag it.

## When in doubt
Ask. Asking is cheap; undoing isn't.
