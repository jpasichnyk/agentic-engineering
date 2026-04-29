# Field Guide

> Status: draft. Reference doc — looked up by topic, not read top-to-bottom.

## 1. How to use this doc

This is a reference doc, not something to read top-to-bottom. When you're mid-task and need a concrete how-to — how to plan and execute cleanly, how to write tests around legacy code, how to set up a hook that blocks something dangerous — look up the relevant pattern. Each pattern is structured the same way: when to use it, how it works, common pitfalls, and cross-references to related patterns and to the principles in [`best-practices.md`](best-practices.md). Playbooks (§3) string multiple patterns together for end-to-end scenarios. Security guidance (§4) covers what we've committed to and what we're still figuring out. If you've found a pattern, tweak, or near-miss worth sharing, contribute it.

## 2. Patterns

Each pattern follows a fixed micro-structure: **When to use → How it works (steps) → Common pitfalls → Cross-references.**

### Plan-Then-Execute

**When to use** — Any work that spans multiple files, has unclear requirements, or involves non-trivial design decisions. Skip planning for: bug fixes with a clear root cause, single-field tweaks, styling/copy changes, simple dependency bumps. Require planning for: schema or data-model changes, multi-endpoint additions, anything affecting auth or middleware, work that spans multiple services or non-obvious dependencies. The threshold isn't lines of code — it's whether the change has design decisions an engineer would want to argue about. If there's nothing to argue, just execute. If there is, plan first.

**How it works**
1. Open a planning session. State the goal and any known constraints; iterate until the plan is concrete: what changes, in what order, and why.
2. Write the plan down — a doc, a chat artifact, a markdown file. The artifact is the output; if you skip this step, the session produced nothing transferable.
3. End the planning session. Do not start building.
4. Open a fresh session with no prior context.
5. Hand the executor only the plan. Two variants both work: subagent fork (the planner spawns a subagent with the plan as its prompt) or human relay (copy-paste the plan into a new session yourself).
6. **If the plan is large, break it into right-sized chunks and hand each to a fresh session.** A single executor session that has to hold the entire plan accumulates the same context bloat the planning split was meant to escape. Multiple fresh sessions, one per chunk, keep each executor's window focused on what it's actually building.

**Common pitfalls**
- Starting to build in the planning session because momentum feels good — the most common failure. The context you've accumulated is the problem, not an asset.
- Handing the executor more than the plan: prior chat, the planning transcript, "for context." That defeats the clean separation.
- Letting the planner skip writing the plan down. Unwritten plans exist only in a context window that's about to close.
- Letting scope creep slip in at execution time without cycling back through planning.

**Cross-references** — [Principle 1: Manage context aggressively](best-practices.md#1-manage-context-aggressively); [TDD with Separated Agents](#tdd-with-separated-agents)

### TDD with Separated Agents

**When to use** — Feature work or bug fixes where correctness matters and you want a structural guarantee that the agent isn't gaming its own tests. The safety property here is contextual separation: each agent sees only what it needs, so no single agent can unconsciously adjust its output to pass checks it also wrote.

**How it works**
1. **Spec/plan agent** — defines what must change, the acceptance criteria, and the contract (inputs, outputs, edge cases). Produces a written spec; does not write code or tests.
2. **Test-writing agent** (clean context) — receives the spec only. Writes failing tests that capture the acceptance criteria. Does not see the implementation or the planner's reasoning.
3. **Implementation agent** (clean context) — receives the spec and the failing tests. Makes the tests pass. Does not see the test-writing agent's transcript.
4. **Verification agent** (clean context) — receives the spec, the tests, and the diff. Checks that the implementation actually satisfies the spec and that the tests are not trivially circumvented.

**Common pitfalls**
- Letting any one agent see all four artifacts at once — the separation is the safety property; collapsing it turns this into ordinary single-session TDD.
- Giving the test-writer the implementation as a "hint." The tests will pass, and they will tell you nothing.
- Skipping the verification step because the tests are green. Green tests only confirm the implementation satisfies the tests, not that the tests satisfy the spec.
- Feeding all four agents' transcripts into a final reviewer. That re-pollutes the context and recreates the blind spots you separated to avoid.

**Cross-references** — [Principle 1: Manage context aggressively](best-practices.md#1-manage-context-aggressively); [Principle 5: Tests are leverage](best-practices.md#5-tests-are-leverage); [Plan-Then-Execute](#plan-then-execute)

### Cross-Model Code Review

**When to use** — Before merging a meaningful change; when stakes are high; when you want a second opinion on a plan or spec before execution begins. The more context the producing agent accumulated, the more useful this is — a session that spent many turns designing an approach has deep framing that shaped every decision, and it cannot see past that framing on its own.

**How it works**
1. Start a fresh agent session. A different model is ideal but not required — the key property is a clean context with no stake in prior decisions.
2. Give it only the spec or plan, the readme, and the diff. No chat history, no prior reasoning, no helpful framing from the producing session.
3. Ask it to identify gaps, contradictions, missing requirements, or changes that don't match the spec. Do not ask "does this look good?" — ask "what's missing or wrong?"
4. Read the output critically. Agreement does not mean correctness; two models can share the same blind spot. Disagreement flags a decision for human attention.

**Common pitfalls**
- Coaching the reviewer: asking "does this look good?" steers it toward agreement rather than gap-finding.
- Giving it too much context — prior chat, the planning transcript, your own summary of intent — defeats the fresh-eyes property.
- Treating agreement as validation. Two models trained on similar data can fail the same way; consensus is a signal, not a guarantee.
- Running the review in the same session that produced the change. The framing is still there; you get corroboration, not review.

**Cross-references** — [Principle 3: Verify with fresh eyes](best-practices.md#3-verify-with-fresh-eyes); [Doc and PR Discipline](#doc-and-pr-discipline)

### Pin-Then-Modify

**When to use** — Before changing any code that lacks coverage, has unclear behavior, or carries implicit contracts the rest of the system depends on. This is most critical with legacy code, but applies any time the question "what will break?" cannot be answered by reading the code alone. The sequencing matters: characterization tests are locked in *before* the refactor, not bolted on after. Tests written after a change can only confirm the change passes — they can't tell you whether anything else broke.

**How it works**
1. Read the code and identify behaviors the rest of the system observes: return shapes, side effects, error handling, ordering guarantees.
2. Have an agent write tests that pin those behaviors against the *current* code. The tests should pass without any modifications.
3. Run the tests and confirm they are green. If they are not, the pinning is wrong — fix the tests, not the code.
4. Make the planned change.
5. Re-run the tests. Failures now pinpoint exactly which behaviors the change broke, giving the agent precise feedback to act on.

**Common pitfalls**
- Writing tests that only cover the new behavior — those belong in a separate test-writing pass; the pinning tests must cover what exists, not what you intend.
- Writing the pinning tests against the desired post-change behavior rather than the current behavior. Pinning tests should start failing only *after* the change, never before.
- Skipping the green confirmation before making the change. An already-failing test tells you nothing about what the change broke.
- Pinning so comprehensively that no refactor can pass — the goal is a change-safety layer, not a change-prevention layer. Pin what the system contracts on, not every internal detail.

**Cross-references** — [Principle 5: Tests are leverage](best-practices.md#5-tests-are-leverage); [Principle 4: Service boundaries enable agentic looseness](best-practices.md#4-service-boundaries-enable-agentic-looseness); [Run-and-Iterate Loop](#run-and-iterate-loop)

### Run-and-Iterate Loop

**When to use** — Whenever the agent needs to converge on working software, especially at integration, end-to-end, or UI layers where unit tests can't catch the real failure modes. If the question is "does this actually work against the running system?", use this pattern.

**How it works**
1. Give the agent the ability to *run* tests, not just write them: terminal access, a test runner, and whatever infrastructure the test tier requires (a live database, a container, a headless browser).
2. Let the agent run the full test suite or a targeted subset. The first run will likely fail — that's the point.
3. The agent reads the failure output, forms a hypothesis, makes a targeted change, and reruns. Real failure messages (stack traces, assertion diffs, HTTP errors) are more informative than any mock can be; the agent uses them directly.
4. The loop continues until tests are green, the iteration budget is exhausted, or the agent surfaces a blocker for human review.
5. Time-box the loop: set a maximum iteration count or wall-clock limit before the session starts. Without a hard stop, the agent can spin on a fundamentally wrong assumption indefinitely.

**Variants**

- *Unit loop* — fast, tight feedback; the agent runs in-process tests and fixes logic errors in a single file or module.
- *Integration loop* — tests run against real or test-container databases and services; catches contract mismatches between components that unit tests mock away.
- *Browser loop* — a headless browser (Playwright, Selenium) drives the actual frontend; catches UI regressions, selector drift, and rendering issues that no other tier can surface.
- *Docker loop* — the full application starts inside a container; catches environment and configuration problems that only appear outside a developer's local setup.

**Use the variants as tiers, not alternatives.** A meaningful change usually wants progressive verification: build/typecheck → relevant unit tests → integration tests for the touched contracts → a smoke test against the running system. Each tier earns trust in the next. Skipping straight to the slowest tier wastes time on issues a faster tier would have caught; skipping the slowest tier ships changes that compile and pass unit tests but break in production-like conditions.

**Common pitfalls**
- Letting the agent iterate without observable progress — if the same test fails in the same way after three attempts, stop and diagnose rather than continue the loop.
- Not capturing test output for human review. The loop's value is the failure trail; preserve it.
- Allowing destructive operations inside the loop without guardrails. The loop is high autonomy by nature; pair it with strict allow/deny lists so a runaway iteration can't corrupt data or infrastructure.
- Mocking at the boundaries the agent should be testing against — this converts an integration loop into a unit loop and defeats the "real failures" property entirely.
- Looping the agent over mechanical work that could be a deterministic script. If the same sequence of operations runs identically every iteration, extract it. See [Use scripts where agents don't belong](working-with-ai.md#use-scripts-where-agents-dont-belong).
- **Running tests that the agent's environment can't actually run.** Some test suites need Docker socket access, real network egress, or external services that an agent sandbox doesn't provide. Document which tests are sandbox-safe and which require a local or CI run; route the agent to the right tier rather than failing through 'environment unavailable' errors.

**Cross-references** — [Principle 5: Tests are leverage](best-practices.md#5-tests-are-leverage); [Principle 2: Match autonomy to phase and blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius); [Pin-Then-Modify](#pin-then-modify)

### Long-Lived Agent Configuration

**When to use** — Any project you work on repeatedly, and any rule you want applied across all your projects. The more you find yourself re-explaining context or preferences at the start of each session, the more you need this.

**How it works**

Project-level config files (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`, and equivalents) live in the repo and are loaded when an agent starts in that directory. User-level config lives in your home directory and applies everywhere. Together they let you encode two categories of guidance:

- **Prose guidance** — judgment calls the agent should make: voice and formatting conventions, when to ask before proceeding, what counts as a "safe" change in this codebase, which directories to treat as off-limits by default.
- **Hard rules** — never put these in prose. "Always run lint before committing" and "never delete migration files" sound like config, but they are enforcement problems. If the rule must hold regardless of context, it belongs in a hook (see [Hook-Based Enforcement](#hook-based-enforcement)), not a sentence in a markdown file.

**One canonical file in multi-vendor environments.** When you use multiple agents (Claude Code, Codex, Cursor, etc.), maintaining the same rules across `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, and any other vendor-specific file is a recipe for drift — they will diverge, and which one is right won't be obvious. Pick one canonical file (we recommend `AGENTS.md` because it's the more widely-used cross-vendor convention) and have the others reference it with a single line: "see AGENTS.md." A starter `AGENTS.md` is provided at [`starters/AGENTS.md`](starters/AGENTS.md).

**Common pitfalls**

- **Prose-rule bleed** — instructions in `CLAUDE.md` get evicted as context fills past roughly 50%. An agent that followed the rule for the first half of a long session may quietly stop following it in the second half. This is the symptom behind "why did the agent ignore the rule in `CLAUDE.md`?" If the rule is safety-critical, prose is not a sufficient enforcement mechanism.
- Over-loading config files with everything you've ever wanted an agent to do — long files get skimmed or partially loaded. Keep project config focused on what is genuinely specific to that project.

**Cross-references** — [Principle 6: Enforce with hooks and config, not prose](best-practices.md#6-enforce-with-hooks-and-config-not-prose); [Hook-Based Enforcement](#hook-based-enforcement)

### Hook-Based Enforcement

**When to use** — Any rule that must hold regardless of what the agent remembers. If you have ever worded a rule more carefully hoping the agent would follow it better, and it didn't, that rule belongs in a hook.

**How it works**

Hooks are scripts or configurations that the tool, not the agent, executes at defined lifecycle points — before a tool call, after an edit, at session start. The agent cannot opt out of them by forgetting them or reasoning past them.

Concrete examples of rules that belong in hooks rather than prose:
- Block `rm -rf` outside a specific scratch directory.
- Require human approval before any `git push`.
- Auto-run `npm run typecheck` after any edit to a `*.ts` file.
- Restrict filesystem reads to a whitelisted directory tree.
- Scope CI checks to changed file types — docs-only PRs run markdownlint only; code PRs run lint and tests. Reduces the false-failure rate that erodes trust in CI.
- Validate configuration at load time, not at runtime. A config file with a missing required field should fail loudly when the application starts, not produce a confusing error fifteen minutes into a job.

Claude Code has a first-class hooks system with pre-tool-use, post-tool-use, and session-lifecycle hooks. Codex exposes similar enforcement via permissions profiles. The mechanism differs, but the principle is the same across vendors: when your tool offers programmatic guardrails, use them for rules that must hold — and use prose only for rules that can reasonably flex with context.

**Common pitfalls**

- Engineering hooks for judgment calls — if the right answer depends on context, a hook that blocks the action is more likely to frustrate than protect. Hooks are for invariants.
- Silent failures — a hook that blocks an action without emitting visible output leaves the agent (and you) with no signal. Always emit a clear message stating what was blocked and why.
- Performance drag — hooks that run a slow process on every file edit will make every session painful. Scope triggers narrowly (e.g., only on `*.ts` edits) and keep hook scripts fast.

**Cross-references** — [Principle 6: Enforce with hooks and config, not prose](best-practices.md#6-enforce-with-hooks-and-config-not-prose); [Permissions and Allowlists](#permissions-and-allowlists)

### Permissions and Allowlists

**When to use** — Every project where an agent has terminal access. This is non-negotiable when the CLI surface includes destructive operations: cloud infrastructure commands, edge compute platforms, database CLIs, container orchestration. The blast radius of a single misfire in those contexts is large enough that relying on the agent's judgment is not an acceptable control.

**How it works**

Maintain explicit **allow** and **deny** lists at the project level — both, not one or the other. An allowlist alone tells the platform what's safe; a denylist adds a named wall around what you know is dangerous. Belt and suspenders.

Build the lists *with AI help before kicking off real work*. The enumeration problem — "what commands does this tool even have?" — is tedious for humans and trivial for a model. Ask the agent: "For tool X, list every subcommand. Mark each one as safe, mutating, or destructive." Review the output and codify it in your project config. You'll cover the full surface in minutes rather than discovering gaps by accident mid-session.

Concrete example: the Fastly CLI has subcommands that purge cache globally, modify routing configs, and delete services. A reasonable per-project list allows `fastly stats`, `fastly log-tail`, and `fastly service list`; it explicitly denies `fastly purge --all` and `fastly service delete`. The same enumeration exercise applies to any CLI with destructive subcommands — cloud providers, DNS tools, database clients.

The secondary goal of allowlisting is often overlooked: suppress friction for routine commands. Every time the agent asks permission to run `git status`, `ls`, `pytest`, or `npm run lint`, that's a prompt that slows you down without adding safety. Add the safe, read-only, idempotent commands to your allowlist so the agent can move without interruption on low-risk actions.

**Wrap risky patterns in scripts; allow the script, not the wildcard.** When an agent repeatedly asks to run a tool with broad arguments — for example `python3 *` — granting wildcard access is dangerous: the agent could run `python3 -c "drop production database"` and the allowlist wouldn't catch it. The safer pattern: ask the agent to write the recurring operation as a small reusable parameterized script, review the script, and allow only that named script (not the underlying tool). The agent stops asking for permission on every call, the action is now reviewable in advance, and the blast radius is bounded by what the script accepts as parameters. This is one of the highest-leverage moves for projects where the agent runs interpreters, shell wrappers, or other powerful tools repeatedly.

Use the platform's permission system to enforce the lists — don't rely on the agent reading them and complying. Claude Code has first-class allow/deny configuration in its settings. Codex exposes permissions via profiles. The mechanism differs, but the principle holds: hard configuration beats good intentions.

On `--dangerously-skip-permissions`: this flag exists for a reason. It is useful in fully isolated environments — a disposable worktree, test-only data, no production credentials anywhere in the environment. It is not a substitute for configuring proper lists, and it should never be your default just to stop being prompted. A denylist as the only safety net, with all permission checks disabled, is not belt-and-suspenders; it's a single point of failure.

**Common pitfalls**

- Treating list maintenance as a one-time task. When you adopt a new CLI or upgrade a tool's major version, schedule a review pass. Allowlists that grow unchecked become equivalent to no list.
- Building only a denylist and assuming you've covered the surface. Obscure subcommands, aliases, and environment-specific variants will not be in a manually written denylist.
- Using `--dangerously-skip-permissions` as a daily convenience. If you find yourself reaching for it routinely, the fix is a better allowlist, not a permanent bypass.
- Adding commands to the allow list reactively, one permission prompt at a time, without ever doing the full enumeration. You end up with a sparse list that still interrupts constantly.
- Granting wildcard access on a powerful tool (e.g., `python3 *`, `bash -c *`) just to stop being prompted. Wrap the recurring use case in a named script and allow that instead — same convenience, much smaller blast radius.

**Cross-references** — [Principle 2: Match autonomy to phase and blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius); [Principle 6: Enforce with hooks and config, not prose](best-practices.md#6-enforce-with-hooks-and-config-not-prose); [Hook-Based Enforcement](#hook-based-enforcement)

### Parallel Agents Without Collisions

**When to use** — Whenever you want two or more agents working on related work simultaneously and need a structural guarantee they won't fight over the same files. If the work is tightly coupled — agents writing to the same module, coordinating on the same API surface — sequence it instead. Parallelism only pays when the task partitions cleanly.

**How it works**

Git worktrees are the cleanest mechanism. Each agent gets its own working directory pointing at the same repo, on its own branch. No file conflicts, no shared state in the working tree.

**Prefer visible sibling directories over hidden subfolder worktrees.** Lay them out at the same filesystem level: `core/`, `core-feature-x/`, `core-investigation-y/` side by side. The `git worktree add` syntax for siblings is `git worktree add ../core-feature-x feature-x` from the main checkout. The hidden-subfolder pattern (`core/.worktrees/feature-x`) is technically equivalent but creates a real failure mode: when you bounce between worktrees, both humans and agents lose track of which one is actually active. The directory tree from `ls` doesn't tell you anything useful, and a terminal pane in `core/.worktrees/feature-x` looks indistinguishable from one in `core/.worktrees/feature-y`. Visible siblings make the active workspace self-evident and reduce the chance of running a command in the wrong context. Separate clones are a valid fallback when your tooling doesn't play well with worktrees.

One agent per branch. Agents never write to each other's branches mid-flight. Merge conflicts get resolved by a human or a coordinating agent at integration time, after each branch's work is complete. For long-running tasks, pair a clean-context coordinator agent with the workers: it reviews each branch's output periodically and manages final integration.

**Workstream isolation extends past the working tree.** Shared resources — job queues, cache namespaces, search indexes, scheduled-task targets — also need workstream-scoped names. A job enqueued from workstream B that gets picked up by a backend running on workstream A's code is a real failure mode that visible directories alone won't prevent. Tag queue names, cache prefixes, and similar namespaces with the workstream so they route correctly even when the underlying database or service is shared.

**Common pitfalls**

- **Shared state outside the working tree** — env files, local databases, and port bindings are not isolated by worktrees. Agents will fight over them. Fix: per-worktree config, explicit port assignment per agent, isolated databases.
- **Forgetting cleanup** — worktrees that outlive their branches accumulate. Prune after merge: `git worktree remove`.
- **Parallelizing tightly-coupled work** — if two agents must read each other's output to make progress, that's a sequential dependency, not a parallelism opportunity. Sequencing is the right answer.
- **Treating trivial or docs-only edits as exempt from the worktree workflow.** The discipline only works when it's universal — 'just a tiny fix on main' is how the pattern erodes. If main is sacred for code, it's sacred for docs and config too.
- **Running too many parallel streams across *unrelated* contexts.** Same-work divide-and-conquer is usually fine; the dangerous pattern is one stream on the dashboard, one on a backend service, one on an investigation, one on a side tool — different mental models, different decision points, all needing your attention at once. The 4th cross-context stream is the one that gets the careless approval. See [Don't dilute your attention](working-with-ai.md#dont-dilute-your-attention) for the judgment side.

**Cross-references** — [Principle 1: Manage context aggressively](best-practices.md#1-manage-context-aggressively); [Plan-Then-Execute](#plan-then-execute); [Cross-Model Code Review](#cross-model-code-review)

### Workstream-Aware Configuration

**When to use** — Any project where multiple parallel workstreams need different ports, databases, env vars, or service configurations. This pattern bridges the *infrastructure* of parallel workstreams — isolated working trees, separate database instances — with the *runtime behavior* the agent must honor. Without it, an agent in workstream B reaches for hardcoded defaults that belong to workstream A, and collisions happen even when the underlying infrastructure is properly isolated.

**How it works**
1. Each workstream has an override file at a known path — e.g., `CLAUDE.workstream.md` or `.workstream/config.md` — capturing workstream-specific values: ports, database names, env var overrides, service URLs. The path and format are your call; the discipline is that every workstream has one and every agent knows to look there.
2. The agent reads this file at session start, before reaching for any hardcoded value. Make this an explicit instruction in your `CLAUDE.md` or `AGENTS.md` — it is too easy to skip otherwise (see [Long-Lived Agent Configuration](#long-lived-agent-configuration)).
3. If no override file is present, fall back to project defaults. Absence of an override is valid; it just means "use the defaults."
4. Commit the override file alongside the workstream branch so other agents and collaborators in that workstream see the same configuration.
5. Treat hardcoded values that bypass the override as bugs — flag them for cleanup.

**Common pitfalls**

- **Agent skipping the override-read step** — the most common failure. Fix it in your project-level config file, not by trusting the agent to remember.
- **Hardcoded ports or database names in source code** — they don't consult the override and will route to whichever workstream owns that value.
- **Override files not committed to the branch** — if the file lives only in the working tree, agents starting fresh in that workstream won't have it.
- **Override file diverging from what infrastructure provides** — infrastructure assigns port 8081 but the override still says 8080. Keep them in sync; a mismatch is a configuration bug.

**Cross-references** — [Principle 1: Manage context aggressively](best-practices.md#1-manage-context-aggressively); [Parallel Agents Without Collisions](#parallel-agents-without-collisions); [Long-Lived Agent Configuration](#long-lived-agent-configuration)

### Parallel Workstream State Sharing

**When to use** — Multiple agents running in parallel workstreams who need to see each other's activity without stepping on each other; long-running work where future sessions need a visible "what's in flight / what was done" record beyond git history; humans monitoring several streams who want a per-stream snapshot without reading every commit.

- *When it's overkill:* Single-agent tasks; throwaway or experimental work; projects where git log and PR descriptions already serve the visibility role. Be honest about whether anyone will actually read the file — if the answer is no, you're paying for nothing.

**How it works**
1. Each active workstream-agent owns a state file at a known path — e.g., `.claude/session-state.md` or `.workstream/state.md`. One file per workstream; agents don't share a file.
2. The file captures: current goal, recent meaningful actions, blockers, open decisions, last-updated timestamp. Prose-y, narrative, not a log of every edit.
3. The agent updates at *task transitions and decision points* — not on every turn or every file edit. The right cadence makes the file readable; updating on every action makes it noise.
4. Other agents and humans read the file to understand what's running and where coordination is needed.
5. On completion, the file becomes part of the audit trail: commit it, archive it, or fold it into the PR description.

**Common pitfalls**

- **Updating on every turn** — overhead crushes value. Update at meaningful checkpoints: goal shifts, blockers, significant decisions.
- **Letting it go stale** — a stale state file is worse than no state file because readers can't trust it.
- **Duplicating what git already tracks** — file paths changed, commit messages. Let git own that; the state file carries what git cannot: decisions made, context behind a choice, what's actively blocked.
- **Treating it as the agent's own memory** — this is a coordination artifact for *other* readers. Internal reasoning belongs in context curation, not here.

**Cross-references** — [Parallel Agents Without Collisions](#parallel-agents-without-collisions); [Workstream-Aware Configuration](#workstream-aware-configuration); [Doc and PR Discipline](#doc-and-pr-discipline)

### Choosing Model, Vendor, Effort

**When to use** — Every non-trivial task. Defaulting to your familiar model at default settings is rarely optimal; matching model class and effort level to the task is one of the highest-leverage adjustments you can make without changing anything else about your workflow.

**How it works**

The default heuristic: top-tier reasoning models with extended thinking on planning and design; mid-tier or fast models on well-defined execution. The cost of a wrong plan compounds through every downstream step, so under-resourcing the planner is the failure mode that hurts most. Over-resourcing routine execution is wasteful but rarely produces worse work.

Map task type to model class. Discovery, planning, and open-ended architectural thinking benefit from top-tier reasoning models with extended-thinking enabled — the cost is worth it when a bad plan compounds through every downstream step. Routine code edits where context and requirements are already clear run well on mid-tier models. Bulk transformations, simple lookups, and mechanical reformatting belong on fast, cheap models.

Set effort to match stakes, not habit. Pour premium effort into the planning agent; dial it down on simple executors. Burning maximum effort on a three-line fix wastes money and time; under-resourcing the planning phase creates errors that multiply through execution.

Mix models within a single task when the stakes justify it. A consensus loop runs the same prompt past two or three models and compares responses — disagreement surfaces decisions that need human attention. Different model families also have different failure modes, so cross-model review catches what same-model self-review misses.

The underlying tradeoff is cost-quality-speed: pick two, know which two you picked, and revisit when the task changes shape.

**Common pitfalls**
- Loyalty to one vendor — you'll miss the others' strengths and blind spots.
- Always-max-effort — expensive, slow, and no better than mid-tier on routine work.
- Never-max-effort — under-resourcing the planning phase, where errors compound the most.
- Choosing by familiarity rather than fit.

**Cross-references** — [Principle 1: Manage context aggressively](best-practices.md#1-manage-context-aggressively); [Principle 3: Verify with fresh eyes](best-practices.md#3-verify-with-fresh-eyes); [Cross-Model Code Review](#cross-model-code-review)

### Doc and PR Discipline

**When to use** — Every change that ships: features, refactors, dependency bumps, config changes. There is no category of change small enough to skip this. If code merges without a useful PR description and up-to-date docs, the next person to touch that code starts from a deficit.

**How it works**

Treat doc updates and PR descriptions as part of the work, not cleanup afterward. When the agent edits code, it should also update the relevant README sections, ADRs, and inline comments in the same pass. Documentation that lags behind code is not neutral — it actively misleads.

A PR description should contain six things: the goal, a summary of what changed, any non-obvious decisions and why they were made, what was explicitly *not* changed and why, a link to the spec or plan, and any open questions that didn't get resolved. The agent can draft this directly from the diff and the plan; the human edits for accuracy and tone. This takes minutes and pays for itself in reviewer time.

Detailed PR descriptions are leverage for reviewers. A reviewer who can read "we chose approach A over B because of constraint C" in thirty seconds is a faster, more useful reviewer than one who has to reconstruct that reasoning from the diff. They also catch decisions early — a bad call that surfaces in PR review costs far less than one that surfaces in production.

**Common pitfalls**

- Generic descriptions ("updates X to Y") that summarize the diff without explaining the decisions — they add no value and reviewers skip them.
- Doc updates that lag behind code, accumulating until the documentation actively contradicts the system.
- The agent inventing decisions the human didn't make — draft descriptions need a human review pass before they go up.
- Treating the PR description as a formality rather than a tool.

**Cross-references** — [Principle 6: Enforce with hooks and config, not prose](best-practices.md#6-enforce-with-hooks-and-config-not-prose) — a PR template hook can require sections before merge; [Cross-Model Code Review](#cross-model-code-review)

### Job Run State and Audit Trail

**When to use** — Recurring or long-running orchestrated agentic workflows — environment provisioning, data migrations, scheduled jobs, multi-step onboarding — where you need to debug failed runs, tune the workflow over time, and decide when to loosen approval gates. Without it, a failed run means reconstructing events from scattered logs, and loosening an approval gate is a guess rather than a data-driven call.

- *When it's overkill:* One-off tasks; workflows you've run two or three times with no plans to scale. Don't build audit infrastructure before the workflow is stable — add the audit layer *after* it has run enough times that you know what shape it has.

**How it works**
1. Each job run owns a state record — a file, a database row, a document — with structured per-phase data: phase name, status (pending / running / completed / failed), inputs, outputs, timestamps, cost, human touchpoints, and error details.
2. The orchestrating agent updates the record at every *phase transition*, not on every action. A phase transition is a meaningful checkpoint: a step completes, fails, or blocks on a human.
3. The record persists after the run as historical data, aggregable across runs to show trends.
4. Reviewers query the aggregated data: which phases are most expensive? where do humans get pulled in? which steps fail most? has this gate earned loosening?
5. Failed-run records become debugging input: replay the inputs, inspect the failing phase, fix the workflow.

**Common pitfalls**

- **Tracking metrics no one looks at** — capture only data that will inform actual decisions. Everything else is maintenance with no return.
- **Letting state diverge from reality** — recording "completed" when the job crashed. A record that lies is worse than no record.
- **Over-engineering before the workflow is stable** — instrument incrementally as the process stabilizes.
- **Mixing run-state with operational alerts** — the audit trail is for tuning and debugging; real-time failure alerts belong in your monitoring system.

**Cross-references** — [Repeatable automation — onboarding a new retailer](#repeatable-automation--onboarding-a-new-retailer); [Principle 2: Match autonomy to phase × blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius) (the audit data tells you when blast radius shrinks enough to loosen gates); [Run-and-Iterate Loop](#run-and-iterate-loop)

## 3. Playbooks

Each playbook is an annotated end-to-end walkthrough with named pattern references at each step.

### Designing and building a new feature

Use this playbook when building something new end-to-end: a feature that adds files, introduces a new API surface, or requires design decisions before a line of code is written. It covers the full arc from fuzzy requirement to merged PR, and it is overkill for a two-line fix. Apply it when the work is large enough that a wrong early decision would compound through every downstream step.

**Phase 1 — Discovery** *(human-in-the-loop iteration; no pattern yet)*
Open a planning session and state the goal and known constraints. Expect back-and-forth: the first statement of requirements is rarely the right one. Narrow scope explicitly — list what is *not* in scope as clearly as what is. The goal here is stable requirements, not a plan; don't rush to architecture until the feature boundary stops moving.

**Phase 2 — Plan capture** *(planning agent produces written artifact)*
Once requirements are stable, direct the planning agent to write a spec: goals, non-goals, architecture, affected files, and acceptance criteria. Read it and push back on anything vague or missing. The spec is the contract between planning and execution; gaps here become bugs later. Do not end this phase until you can approve the spec in writing.

**Phase 3 — Plan handoff** *([Plan-Then-Execute](#plan-then-execute))*
Close the planning session. Open a fresh execution session and hand it only the finalized spec — not the planning conversation, not "for context" summaries. The separation is the point: the executor must work from the spec, not from the accumulated framing the planner developed. A subagent fork works; so does a manual copy-paste.

**Phase 4 — Test scaffolding** *([TDD with Separated Agents](#tdd-with-separated-agents))*
Inside the execution session, or as a sub-session with clean context, draft tests against the acceptance criteria before writing any implementation. The test-writing agent should see the spec and nothing else — no partial implementation, no hints about how you plan to build it. Failing tests that express the spec's intent are the artifact; tests that are already green before any implementation exists mean nothing.

**Phase 5 — Implementation loop** *([Run-and-Iterate Loop](#run-and-iterate-loop))*
The implementation agent receives the spec and the failing tests. It implements, runs the test suite, reads failures, and adjusts — real failure output is more informative than any mock. Set a time-box or iteration cap before the loop starts. If the same test fails three attempts in a row without progress, surface the blocker for human review rather than continuing the loop.

**Phase 6 — Cross-model review** *([Cross-Model Code Review](#cross-model-code-review))*
Before the PR, run the spec and the diff past a fresh-context agent — a different model is ideal. Give it only the spec, the readme, and the diff; ask what's missing or wrong, not whether it looks good. Pay attention to disagreements: they flag decisions that need a human look. Agreement is a signal, not a guarantee.

**Phase 7 — PR opening** *([Doc and PR Discipline](#doc-and-pr-discipline))*
Have the agent draft the PR description from the spec and diff: goal, summary of changes, non-obvious decisions and why, what was explicitly not changed, and a link to the spec. You edit for accuracy and tone, then open the PR. A reviewer who can read your reasoning in thirty seconds is faster and more useful than one reconstructing it from the diff.

**Phase 8 — Merge**
Standard human review on the PR. No agent involvement at this phase. Merge when green.

**Principles invoked:** [1 — Manage context aggressively](best-practices.md#1-manage-context-aggressively); [2 — Match autonomy to phase and blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius); [3 — Verify with fresh eyes](best-practices.md#3-verify-with-fresh-eyes); [5 — Tests are leverage](best-practices.md#5-tests-are-leverage); [6 — Enforce with hooks and config, not prose](best-practices.md#6-enforce-with-hooks-and-config-not-prose)

### Modifying legacy code safely

Use this playbook when you need to change code that lacks adequate tests, has unclear intent, or carries implicit contracts the rest of the system depends on. "Legacy" here means anything where the honest answer to "what will break?" is "I'm not sure." The playbook is heavier than editing well-tested code — it front-loads understanding and test-writing before touching anything. That overhead pays for itself the first time a pinning test catches a behavior you didn't know existed.

**Phase 1 — Understand current behavior** *(no pattern yet; human-in-the-loop reading)*
Read the legacy code and its downstream callers before forming any plan to change it. List what the code currently does, including the non-obvious: return shapes, side effects, error handling paths, ordering guarantees, and anything callers may be depending on implicitly. Write this list down — it is the contract you are about to work against. Do not plan the change until this inventory is stable.

**Phase 2 — Pin behavior with tests** *([Pin-Then-Modify](#pin-then-modify))*
Have the agent write tests that exercise the current behaviors from the inventory. These tests codify what exists, not what you intend. They should pass against the unchanged code. If you are unsure whether a behavior is intentional or accidental, pin it anyway — you will triage later. The goal is a safety net before you touch anything.

**Phase 3 — Confirm green** *([Run-and-Iterate Loop](#run-and-iterate-loop))*
Run the new pinning tests against the unmodified code. They must all pass. If any fail, the tests are wrong or the reading was wrong — investigate and fix the tests before proceeding. A failing pinning test before any change has been made tells you nothing about regressions; it tells you the inventory was incomplete. Do not proceed until the suite is clean.

**Phase 4 — Make the planned change** *([Plan-Then-Execute](#plan-then-execute))*
Close the reading and test-writing session. Open a fresh execution session and hand it only the plan and the pinning test suite — not the prior transcript, not the behavior inventory narrative. The clean context matters: the executor should work from the spec and the tests, not from the accumulated framing of the session that wrote them.

**Phase 5 — Run tests** *([Run-and-Iterate Loop](#run-and-iterate-loop))*
Run the full test suite after the change. Expect some pinning tests to fail. Before acting on any failure, triage it: some failures are the point of the change — behavior that was intentionally altered — and some are regressions you did not intend. These two categories require completely different responses. Do not fix failing tests until every failure has been labeled.

**Phase 6 — Resolve failures**
For each failing test: determine whether it represents an intended behavior change or an unintended regression.

- *Intended change* — update or delete the test, and document the behavior change explicitly in comments or a changelog entry. The documentation matters: future readers need to know the behavior changed deliberately and when.
- *Regression* — fix the code. The pinning test is telling you something broke that was supposed to be preserved. Do not update the test to match the broken behavior.

This triage step is the unique value of the pinning approach. Without it, a suite of red tests is noise; with it, each failure is a clear decision.

**Phase 7 — Cross-model review** *([Cross-Model Code Review](#cross-model-code-review))*
Legacy code is where cross-model review matters most. The agent that made the change has accumulated framing about what the code should do — framing that may have subtly colored which behaviors it treated as intentional versus incidental. A fresh-context agent with a different model sees the diff without that history. Give it the original behavior inventory, the diff, and the list of behavior changes; ask what was missed or misread, not whether the change looks correct.

**Phase 8 — PR with explicit behavior-change notes** *([Doc and PR Discipline](#doc-and-pr-discipline))*
The PR description for a legacy change needs two explicit sections that new-feature PRs often skip: which behaviors changed (with before/after descriptions) and which behaviors were explicitly preserved. Reviewers cannot diff behavior from code alone; you have to surface it. A reviewer who can read "function X no longer swallows errors from Y — it now propagates them" takes thirty seconds to evaluate that decision. One reconstructing it from the diff takes much longer and may miss it entirely.

**Principles invoked:** [1 — Manage context aggressively](best-practices.md#1-manage-context-aggressively); [2 — Match autonomy to phase and blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius); [3 — Verify with fresh eyes](best-practices.md#3-verify-with-fresh-eyes); [5 — Tests are leverage](best-practices.md#5-tests-are-leverage)

### Multi-agent code review on a PR

Use this playbook when you want structural assurance that a PR is correct before it merges — not just "does this look reasonable?" but "does this match the spec, and what did the producing agent miss?" The playbook works precisely because a long implementation session accumulates framing: every decision the agent made shaped how it reads its own output. Two reviewers in clean contexts, ideally on different models, don't share that framing. They find things the original session can't see.

**Phase 1 — Prepare review artifacts** *(no pattern yet; human-in-the-loop prep)*
Gather the spec or plan that motivated the change, any relevant readme sections, and the full diff. Strip everything else: no chat history, no planning transcripts, no "for context" summaries. Each reviewer session must start from only these artifacts. If you include prior reasoning, you import the framing you are trying to escape. Prepare the artifacts once and hand them identically to both reviewers.

**Phase 2 — Reviewer A (same model, fresh context)** *([Cross-Model Code Review](#cross-model-code-review))*
Open a clean session of your primary model. Provide spec and diff only. Ask: "What's missing? What's contradicted? What decisions does this code make that aren't in the spec?" Don't ask whether the change looks good — that steers toward agreement. Ask for gaps and contradictions. Capture the full output before closing the session.

**Phase 3 — Reviewer B (different model, fresh context)** *([Cross-Model Code Review](#cross-model-code-review))*
Open a clean session on a different model. Give it the same artifacts and the same prompt as Reviewer A, verbatim. The different model family is the point: different training, different blind spots, different failure modes. Capture the full output.

**Phase 4 — Compare reviews** *(human judgment)*
Read both outputs against each other. Where reviewers agree: treat it as a real finding and prioritize it — two independent sessions converging on the same gap means something. Where they disagree: flag the item for the engineer with reasoning from each side. Don't resolve disagreements by picking the reviewer you trust more. Consensus is a signal, not a guarantee; disagreement is always a flag.

**Phase 5 — Triage** *(human judgment)*
Sort every finding into one of four categories: **must-fix** (a correctness or safety issue that blocks merge), **should-fix** (real improvement, not blocking), **won't-fix** (acknowledged but declined, with reasoning), or **decision-required** (the finding surfaces a tradeoff that the engineer needs to call). Apply must-fixes immediately. Document won't-fix decisions in the PR description — a reviewer who sees a gap and finds it addressed in the description is a reviewer who doesn't have to ask.

**Phase 6 — Re-run reviewers if changes are substantial** *([Cross-Model Code Review](#cross-model-code-review))*
If the must-fix changes altered meaningful logic — not just formatting or naming — run both reviewers again on the updated diff. Fresh context each time; don't continue the prior reviewer sessions. Accumulated context loses the fresh-eyes property that makes this worth doing. If the changes were minor, skip this step.

**Phase 7 — Open or update the PR with notes** *([Doc and PR Discipline](#doc-and-pr-discipline))*
The PR description should document the review process explicitly: which reviewer findings were applied and which were rejected, with reasoning. A reviewer who sees "Reviewer B flagged the error-handling path in X; we changed approach; Reviewer A flagged the caching strategy as undocumented — won't-fix because the behavior is intentional and now noted in the readme" can evaluate those calls in seconds. Without this record, every finding disappears into the diff and reviewers must reconstruct the reasoning from scratch.

**Principles invoked:** [1 — Manage context aggressively](best-practices.md#1-manage-context-aggressively); [3 — Verify with fresh eyes](best-practices.md#3-verify-with-fresh-eyes); [6 — Enforce with hooks and config, not prose](best-practices.md#6-enforce-with-hooks-and-config-not-prose)

### Repeatable automation — onboarding a new retailer

Use this playbook when you need to automate a recurring multi-step process that currently runs manually. Onboarding a new retailer is the running example — it's a real recurring task with config to set, validation to run, and a clear definition of done — but the pattern applies equally to any process you run more than twice: customer environment provisioning, data migrations, certificate renewals, partner integrations. If you find yourself following the same checklist from memory, that's the signal. The goal is an agent-executable spec that a fresh session can run end-to-end with minimal interruption, backed by hard permissions so autonomy and blast radius stay in proportion.

**Phase 1 — Document the process once, manually with AI help**
Run through the full process with an agent observing alongside — shared screen, transcript, or a parallel chat session where you narrate every action as you take it. The agent's job here is documentation: capturing each decision, each config key touched, each validation step run, and each output you use to confirm success. Don't write the spec in your head; let the agent draft it from observation. You edit it; the agent does the mechanical capture.

**Phase 2 — Refine the spec**
Iterate on the draft until a fresh agent could execute it without you in the room. Vague language like "configure the retailer settings" is not a step; "set `retailer.tier` to `standard` in the tenant config and confirm the value returned by `GET /api/config/{id}` matches" is. Pin every data shape, every config key, every assertion. If you think "an engineer would know what to do here," that's a gap — close it.

**Phase 3 — Build the allow/deny list** *([Permissions and Allowlists](#permissions-and-allowlists))*
Enumerate every CLI command and API call the process uses. Mark each one: safe, mutating, or destructive. Use an agent for the enumeration — feed it the spec and ask it to list every external call; review and extend manually. Codify the result in your project permissions before any automated execution begins. You're building a hard wall around the blast radius, not a best-effort list of things you remember to check.

**Phase 4 — Dry-run on a test environment** *([Plan-Then-Execute](#plan-then-execute); [Run-and-Iterate Loop](#run-and-iterate-loop))*
Hand the finalized spec to a fresh agent with the allow/deny list, test credentials, and test data — nothing else. Watch it execute. Capture every failure, every point where it pauses for clarification, every step it skips or misreads. The failures are the output you want; they are the gap map for Phase 5.

**Phase 5 — Iterate the spec**
Each failure from the dry-run is a gap in the spec, not a failure of the agent. Fix the spec and retry. If the same step fails across multiple iterations, the spec is structurally ambiguous — go back to Phase 2 and rework that section until the gap closes. Repeat until the dry-run is clean end-to-end.

**Phase 6 — Production execution with HITL approval gates** *([Hook-Based Enforcement](#hook-based-enforcement))*
Even with a clean dry-run, the first production execution should pause at every mutating or destructive step and require explicit human approval. This is [Principle 2 — Match autonomy to phase and blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius) applied directly: the automation is new, the blast radius is real, and early runs are not yet validated at production scale. Implement the gates as hooks, not prose in the spec — a rule the agent can read is a rule the agent can reason past; a hook is not.

**Phase 7 — Loosen approval gates after several successful runs**
After a handful of clean production executions, the pattern is established. Pull approval gates off the mutating steps that have proven safe; keep them on anything irreversible — data deletes, credential rotations, downstream partner integrations. The remaining gates aren't bureaucratic overhead; they're acknowledgment that some actions have no rollback path.

**Principles invoked:** [2 — Match autonomy to phase and blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius); [4 — Service boundaries enable agentic looseness](best-practices.md#4-service-boundaries-enable-agentic-looseness); [5 — Tests are leverage](best-practices.md#5-tests-are-leverage); [6 — Enforce with hooks and config, not prose](best-practices.md#6-enforce-with-hooks-and-config-not-prose)

## 4. Security and Data Hygiene

### Committed v1 policies

1. **Work-tier accounts only.** All company AI work happens on paid or business-tier accounts that honor data-sharing opt-outs. Personal free-tier accounts are not acceptable for work, because free-tier conversations may be used to train models. The edge case worth naming: a personal account with a paid subscription still counts as personal-tier for this purpose — what matters is whether the account's data-sharing controls can be governed by the org, not just whether you're paying.

2. **No customer PII in prompts or sessions.** The same data-handling obligations that apply to logs, emails, and support tickets apply to AI sessions. Customer data does not belong in a chat window. This is not a gray area: if you're writing a prompt that includes a name, email, account ID, or any field that could identify a customer, stop and sanitize first. The fact that the model "won't remember it" is not a control.

3. **No production credentials in chat-style sessions.** ChatGPT.com, Claude.ai, and equivalent web interfaces maintain conversation history. Credentials pasted into those interfaces — API keys, tokens, database passwords, service account secrets — are compromised. They belong in a secrets manager, not a chat window. This applies equally to credentials embedded in config files you paste "for context." If you need an agent to work with a live system, use a CLI or SDK tool path where session logging is under your control; don't paste the credentials into a browser tab.

4. **Don't disable security tooling to streamline AI workflows.** Pre-commit hooks, secret scanning, linters enforcing security rules — these exist because the cost of skipping them once is too high relative to the time saved. AI speed gains do not change that math. If your agent keeps tripping a hook, the right response is to fix the underlying issue or adjust the workflow, not to turn the hook off. The friction is doing its job.

5. **No agent write access to production systems without explicit human approval.** Read-only access is acceptable; any mutation — database writes, config changes, deployments, file deletions — requires a named human to approve it first. This is [Principle 2 — Match autonomy to phase and blast radius](best-practices.md#2-match-autonomy-to-phase--blast-radius) applied directly to production: production mutations have the highest blast radius in the blast-radius calculus, and autonomy should be at its lowest there. Implement approval gates as hooks, not prose in a spec the agent can reason past.

If a less-restrictive setup is needed for a specific agent loop or experiment, escalate / discuss before deviating.

### Open security questions

These are genuinely unresolved — not deferred or forgotten, but actively open. If you have opinions, examples, or prior art from other teams, bring them in.

1. **Real credentials in shared configs.** When an agent is asked to debug a config file that contains real production credentials, is sharing that file with the agent acceptable? Under what conditions — work-tier account only? Config sanitized first? Does the answer change if the agent is running locally via CLI versus in a cloud-hosted session?

2. **Secrets on disk in agent read scope.** If an agent has read access to a working folder and a `.env` file or debug config with secrets is present in that folder, is that a problem? The answer may depend on the agent's actual file-access pattern during the session — a focused code-editing task probably never touches `.env`, but a broad "figure out why this is failing" task might. We don't yet have a clear policy here.

3. **Customer-data handling in AI-built tooling.** When AI helps build support tooling, analytics pipelines, or internal dashboards that process customer data, what handling rules govern the data the tooling operates on? This is distinct from "no PII in prompts" — this is about what the AI-written code does with customer data once it's running. Our existing data-handling policies likely cover this, but the intersection with AI-generated code hasn't been worked through explicitly.

4. **`--dangerously-skip-permissions` policy.** The [Permissions and Allowlists](#permissions-and-allowlists) pattern notes that this flag is useful in isolated environments — disposable worktrees, test-only data, no production credentials in scope — but doesn't answer when, if ever, it's acceptable in day-to-day work. Never? Only with test data? Only in combination with a denylist? We need a stated position.

## 5. Open Questions

These are workflow-level questions we haven't answered yet — about how the team operates around agentic work, not about any specific tool or technique. They're here to prompt discussion rather than to gather dust.

1. **How do we measure quality of agentic output across the team?** PR review time, defect rate, and self-reported confidence are all proxies, but none captures the full picture. Without a shared measurement approach, we can't tell whether the team is improving or just getting faster at producing the same error rate. Defining even one lagging and one leading indicator would be useful.

2. **What's the right cadence for revisiting these docs?** Quarterly review, triggered by an incident, or something else? The field moves fast enough that a doc written six months ago may be describing tools and patterns that no longer reflect what we actually use — but scheduled reviews have a way of becoming empty rituals. We should decide what triggers a meaningful revision.

3. **When does a pattern graduate to a hook?** When we've been applying a practice consistently across several projects, at what point do we encode it into automation — a hook, a permissions profile, a repo template — rather than continuing to rely on the team following a doc? There's a cost to premature encoding (inflexibility) and to delayed encoding (drift). We haven't articulated where that line is.

4. **What signals tell us that a doc has gone stale and needs refresh?** Someone asks a question the doc should answer but doesn't. A pattern in the field guide contradicts how we're actually working. A new tool renders an existing recommendation obsolete. These signals exist; we don't yet have a habit of acting on them.

Add others as they emerge.
