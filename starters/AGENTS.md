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
- **Rich opening context is framing, not a question.** When a user opens with paragraphs of background, confirm what they're actually asking before drilling into the first specific thread. The first detail that catches your attention is rarely the real ask.
- **Stick to observable behavior and stated reasoning when summarizing humans.** Reviews, retrospectives, and discussion summaries get reused and circulated. Speculation about people's motivations, traits, or unspoken intent doesn't belong in written output — describe what was said and done, not what you guessed about why.
- **Autonomy is the intersection of phase and blast radius, not either axis alone.** A destructive action *described* in a planning conversation is fine — nothing executes. The same action *running* inside an autonomous loop is dangerous. Calibrate latitude to the multiplication: planning + reversible = wide latitude; autonomous + irreversible = tight gates regardless of how confident the reasoning sounds.

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
- **Never inline literal credentials.** Reference secrets by env-var name (`$DB_PASS`, `$API_TOKEN`) and read from `.env` or the platform's secret store. Don't echo, log, or paste literal values into commands, code snippets, or tool output — even temporarily for debugging. Output is scrolled back to, copied into tickets, and shared.
- **No customer data or PII in prompts, sessions, or tool calls.** Even for "just one debug query." Sanitize identifying fields before pasting; substitute realistic-shape synthetic data when shape matters. "The model won't remember it" is not a control — sessions are logged and the data has already left your environment.
- **Enumerate the destructive surface before autonomous work.** When entering a project that uses powerful CLIs (cloud platforms, edge compute, DNS, container orchestration), if no per-project allow/deny list covers the destructive subcommands, propose building one before running anything autonomous. Use AI to enumerate the surface ("for tool X, list every subcommand; mark each safe / mutating / destructive") rather than discovering gaps mid-session.
- **When a risky pattern recurs, wrap it in a script and allowlist the script — not the underlying tool.** If you keep asking permission for `python3 ...` or `bash -c ...`, extract the recurring operation as a small named script, get it reviewed once, and put only the script on the allowlist. Wildcard access on a powerful interpreter is hard to scope; a named script is bounded by its parameters.
- **Surface allowlist gaps proactively.** If you find yourself asking permission repeatedly for a clearly safe command (`git status`, `ls`, `pytest`), suggest adding it to the project allowlist rather than continuing to interrupt. Recurring prompts on safe operations are tax, not safety.

## Reviews and PRs
- **Push and open a PR — don't merge locally as the default close.** Even for small changes, default to push + PR so human review and CI run. "Merge locally" isn't a co-equal option; it's a deliberate exception that should be named and justified.
- **Frame evaluation as alignment and synthesis, not demolition.** When reviewing someone else's design, proposal, or PR, distinguish "technically sound" from "right priority right now" from "right approach for this org." Surface process and cross-team concerns explicitly, not just code-level ones. The goal is to make the work better, not to win the review.
- **Before opening a PR for a meaningful change, get a fresh-context review.** A different model is ideal but not required — the key property is clean context with no stake in prior decisions. Give the reviewer only the spec/plan/readme + the diff; ask "what's missing or wrong?" rather than "does this look good?"
- **Run the full test suite before opening the PR.** Failing tests don't go in the PR; they go on the to-fix list.
- **Do a documentation review pass before the PR.** Separate from the code review — different question: *does the doc accurately describe what now exists?* Check READMEs, ADRs, inline comments at non-obvious decision points, and the relevant AGENTS.md. Docs that drift from code are worse than no docs — they actively mislead.
- **Draft PR descriptions** that cover: goal, summary of changes, non-obvious decisions and why, what was explicitly *not* changed, link to spec/plan, open questions. A reviewer who can read your reasoning in 30 seconds is faster and more useful than one reconstructing it from the diff.
- **Keep the PR description current as the work evolves.** If review feedback prompts a meaningful change, follow-up commits adjust scope, or you defer something explicitly — update the description. A reviewer's second read shouldn't have to reconstruct what changed since the first.

## Phase handoffs
- **When you finish a phase, draft the next agent's prompt — don't carry into the next phase in the same session.** Planning → execution → tests → review → docs → PR are distinct phases; each wants a fresh-context agent loaded only with the artifacts it needs.
- **Suggest the next handoff explicitly.** When implementation is done, draft a prompt for a fresh test-writing or review agent (with only the spec + diff). When tests pass, draft a prompt for a review agent (with only the spec + diff, asking what's missing). The user decides whether to fire it themselves, dispatch it as a subagent if the platform supports it, or override your suggested prompt.
- **The clean handoff is the point.** Continuing into the next phase yourself defeats the separation that makes the work safer and more reliable.
- **Right-size what you hand off.** Too small wastes turns on coordination; too big lets goals drift mid-stream and evicts the rules the agent was operating under. Tells of too small: approving every tool call, setup never amortizes. Tells of too big: the work goes inconsistent mid-stream, the executor starts re-planning, scope quietly shifts between the first turn and the last. When a planned handoff looks too big, propose chunking it — each chunk a fresh session with the slice of the plan it needs, nothing more.

## Subagent dispatch and parallelism
- **Use subagents to keep the parent session focused, not just to go faster.** A fresh-context subagent is the right tool when a subtask would bloat the parent's context (read-heavy exploration, long file dumps, exhaustive grep, deep dives that aren't on the main path) or when it deserves focused attention without distraction from unrelated parent context. Give the subagent only what it needs; accept its summary back, not its full transcript. The win is token economy and signal-to-noise in the parent, not just wall-clock.
- **Parallelism is for the model's time, not the user's.** When fanning out parallel agents on independent slices of work, consolidate their output into a single structured finding stream for the user. N parallel result streams demand N times the user's attention; one synthesized stream demands one. Use parallel agents to apply more model effort, not to multiply what the human has to read.
- **Same-work parallelism is usually fine; cross-context parallelism is where attention fragments.** Multiple agents on different slices of one feature share a mental model and consolidate cleanly. Multiple agents on *unrelated* workstreams (a dashboard tweak + a backend refactor + a side investigation) each demand their own context, and the 4th stream is the one that gets the careless approval. Default to consolidating cross-context streams sequentially, or surface them for an explicit user attention budget — don't fan them out by default just because the worktrees allow it.

## When something goes wrong
- **An agent failure is usually a docs/instructions gap, not a one-off problem.** If you didn't understand a package, missed a spec, or repeated a mistake, the fix is to update the docs, AGENTS.md, or shared instructions — not to silently work around it. Then verify the fix worked by running the same task in a fresh-context session.

## Multi-vendor / multi-agent
- **This file is canonical.** Other agent-specific files (CLAUDE.md, .cursorrules, etc.) should reference this one rather than duplicate. If you find them out of sync, flag it.

## Project conventions

The categories below capture the kinds of rules that vary by project and don't generalize. Fill in the ones that apply; **delete sections that don't.** Leaving an unfilled placeholder is a signal that this file hasn't been customized — replace it or remove it.

### Code style and formatting
*Project-specific style or naming rules a fresh agent couldn't infer from "match the surrounding style." Keep this short — formatter output should usually be enough.*
- _Example: 2-space indent, single quotes in JS/TS. Run `npm run format` before commit._
- _Example: All exported Go funcs require doc comments starting with the function name._

### Build, test, and verification commands
*Concrete shell invocations a fresh agent should use, rather than guessing.*
- _Example: Full test suite — `make test-all` (sets required env vars; running `pytest` directly will fail)._
- _Example: Single test — `pytest path/to/test_file.py::TestClass::test_name`._
- _Example: Type check — `npm run typecheck`._

### Tooling and package management
*"Use X, not Y" rules that prevent lockfile drift, version mismatches, or environment skew.*
- _Example: Use `bun`, not `npm` or `yarn`. The repo's lockfile is `bun.lockb`._
- _Example: Python deps managed via `uv`; don't `pip install` directly._

### Off-limits paths and modules
*Files or directories that need extra care or explicit human approval before modification.*
- _Example: `migrations/` — never edit existing migration files; always create a new one._
- _Example: `vendor/` — never reformat or modify._
- _Example: `auth/middleware.ts` — read `docs/architecture/auth.md` before touching._

### Branch, commit, and PR conventions
*Project-specific workflow rules that affect how the agent names branches, structures commits, or formats PR descriptions.*
- _Example: Branch names follow `<type>/<ticket>-<slug>`, e.g., `feat/PROJ-123-add-x`._
- _Example: Commit messages follow Conventional Commits._

### Other project-specific DO and DO NOT
*Catch-all for rules that don't fit a named section above. Delete this subsection if everything fits above.*

**DO:**
- _e.g., Read `docs/architecture.md` before touching the auth module._

**DO NOT:**
- _e.g., Don't add new top-level dependencies without asking._

## When in doubt
Ask. Asking is cheap; undoing isn't.
