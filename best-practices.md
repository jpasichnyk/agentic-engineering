# Best Practices for Agentic Engineering

> Status: draft. Living doc — expect change.

## 1. Why this doc exists

Most engineers using AI today live in chat windows and IDE autocomplete — useful, but a fraction of what AI agents can actually do. A smaller group has pushed past that into real agentic work: multi-step execution, clean-context separation, orchestrated workflows. None of what's in here is secret knowledge — anyone who spends enough time in the trenches with AI agents will arrive at most of it on their own. Writing it down is a head start, a way to compress the learning curve into something readable in an afternoon.

The field is multi-vendor by design. Different tasks call for different models, and the right answer at any given time depends on context, not loyalty. This doc is vendor-neutral on purpose: the principles here apply regardless of which assistant or IDE you're in. Alongside the principles, it sets guardrails around autonomy, security, and data hygiene — opinionated enough to be useful, not heavy enough to replace judgment.

This is a living doc. "Written down" does not mean "frozen." The field changes quickly, and anything here that stops reflecting reality should be updated. If you've found something that works — or something that doesn't — say so. For patterns, playbooks, and concrete how-to guidance, see [`field-guide.md`](field-guide.md). For a personal progression path from chat user to multi-agent work, see [`onboarding-ladder.md`](onboarding-ladder.md).

## 2. Core Principles

### 1. Manage context aggressively

Quality of agent output degrades as context grows. Long sessions degrade quality regardless of how big the window claims to be — models start forgetting rules from earlier in the session, conflate related concepts, and become more confident about things that aren't true. Task size is part of the same problem — one over-large task accumulates as much context as several sequential ones in the same session, with the same rule-forgetting and quality drops. Cost rises, turns get slower, and — critically — mixing planning, building, testing, and verification in one session creates failure modes that are hard to detect, like an agent adjusting tests to make itself succeed rather than catching a real regression. Stale, mixed-purpose context is worse than a smaller, curated one.

We default to clean separation across phases: plan in one session, build in another, verify in a third. Two implementations both work — subagent forking with curated context handed off explicitly, or human orchestration via copy-pasting the finished plan into a fresh session. For the how-to, see [Plan-Then-Execute](field-guide.md#plan-then-execute) and [TDD with Separated Agents](field-guide.md#tdd-with-separated-agents). The guiding instinct is "curate, don't pile on" — what you give the agent matters more than how big the window is.

Permissions are part of the same surface. Hand the agent the context it needs and the permissions it needs to act on that context — no more. Curated permissions reduce friction (fewer prompts on safe actions) and reduce risk (fewer routes to dangerous ones). See [Permissions and Allowlists](field-guide.md#permissions-and-allowlists).

### 2. Match autonomy to phase × blast radius

How much latitude to give an agent depends on two axes. The first is *phase of work*: design, discovery, and planning benefit from HITL — the back-and-forth is the output, not overhead to eliminate. Autonomous execution belongs once the plan is locked. The second axis is *blast radius*: how bad is the worst case? Irreversible operations — production deploys, schema migrations, force-pushes, hard deletes — demand tight controls regardless of how confident the agent sounds. Reversible local work — tests, feature-branch code, docs — earns more latitude.

The two axes multiply, and that multiplication is the insight worth internalizing. A destructive action inside a planning conversation is usually fine: the agent is describing what it would do, and nothing executes. The same action inside a long-running autonomous loop is dangerous. We set autonomy at the *intersection*, not at either axis alone. In practice that means enumerating explicit allow and deny lists per project before kicking off autonomous work — especially for CLIs with destructive subcommands. Prose rules drift as context fills; tool-level enforcement does not.

**In practice:**
- Planning conversations: HITL, back-and-forth — the conversation IS the work
- Execution from a finalized plan: autonomous, fresh session
- Production mutations (deploys, deletes, schema changes): named human approval, every time
- Read-only access (logs, queries, config inspection): open by default; friction here is a tax
- Encode these via permissions and hooks; written instructions drift as context fills

See [Permissions and Allowlists](field-guide.md#permissions-and-allowlists) and [Hook-Based Enforcement](field-guide.md#hook-based-enforcement).

### 3. Verify with fresh eyes

An agent that has worked through a long session develops blind spots. Mistakes made early shape the framing for everything that follows — a wrong assumption in the plan becomes an invisible constraint in the build. The agent cannot reliably catch its own gaps in the same context where it made them. The same reasoning that produced a mistake is the reasoning that evaluates whether it was one.

A fresh-context reviewer — same model or different — sees only what a human would hand it: the spec or plan, the readme, and the diff, with no in-session framing and no stake in prior decisions. Cross-model review is especially powerful: different models have different failure modes, so agreement is genuine confidence and disagreement surfaces a decision for the human. This applies to code review and plan review equally — fresh eyes catch ambiguity, missing requirements, and contradictions before they become bugs. The discipline is to not coach the reviewer into agreement; give it the same artifacts a human reviewer would get, nothing more. See [Cross-Model Code Review](field-guide.md#cross-model-code-review).

### 4. Service boundaries enable agentic looseness

The cleaner your service boundaries are, the more aggressive AI work can be. Strong contracts, published interfaces, integration tests, and production smoke or regression tests turn "did the agent write the right code?" into "does the service still honor its contract?" — a mechanical question with a mechanical answer. Inside a well-bounded service, internal style and detailed design choices matter less; the agent has room to refactor, rewrite, or add features without line-by-line human review via [Run-and-Iterate Loop](field-guide.md#run-and-iterate-loop) and [Pin-Then-Modify](field-guide.md#pin-then-modify).

The reverse is also true: tightly coupled, undocumented code is an AI tax. Every agent change risks cross-cutting effects no test catches, and human review becomes a bottleneck that scales with agentic output. This isn't a mandate to redo your architecture — it's a gradient. The cleaner your boundaries today, the more AI can multiply your effort tomorrow; the more tangled, the more carefully every AI change has to be inspected. For the concrete recipe of what "agent-friendly boundaries" looks like in practice, see [Agent-Readable Service Documentation](field-guide.md#agent-readable-service-documentation).

### 5. Tests are leverage

Tests are not a tax on agentic work — they are what makes agentic work safe and fast. Three common workflows each map to a different pattern:

- **Refactors and rewrites of existing code** — start with [Pin-Then-Modify](field-guide.md#pin-then-modify): capture current behavior in tests, make the change, confirm the pinning tests stay green. The agent can't reason about "what should still work"; the test suite can.
- **New features** — [TDD with Separated Agents](field-guide.md#tdd-with-separated-agents): a test-writing agent (clean context) writes failing tests against the spec; a separate implementation agent makes them pass. The separation prevents the implementer from gaming its own tests.
- **Bug fixes** — start by writing a test that captures the bug (it should fail in the same way the bug fails), then TDD the fix until that test passes alongside everything else. Bugs without a captured test recur; bugs with one don't.

Narrow unit tests give the agent a narrow window. To converge on working software, push the test surface further: integration tests, browser tests via Playwright or Selenium, Docker-based end-to-end loops. Real failures from real execution give the agent the feedback it needs. The [Run-and-Iterate Loop](field-guide.md#run-and-iterate-loop) is built around this: agent runs tests, sees failures, makes changes, reruns, converges. That loop is more powerful than any single round of code review because it is objective, repeatable, and immune to the same blind spots that produced the bug in the first place.

### 6. Enforce with hooks and config, not prose

Agents forget rules. As context fills with prior turns and partial results, instructions you wrote at the start of a session — in CLAUDE.md, AGENTS.md, or the prompt — drift out of attention. By the second half of a long session, the rule that was at the top of the file might as well not be there. This is the same context-growth phenomenon as Principle 1, applied to safety rules.

When a rule has to hold, encode it structurally rather than asking the agent to comply. Two layers, both stronger than instructions alone:

**What the agent can invoke** — hooks, permissions, allow/deny lists, scoped credentials, API keys with narrow permissions. The platform enforces what the agent is allowed to call. Cheap to set up; cheap to maintain. See [Permissions and Allowlists](field-guide.md#permissions-and-allowlists) and [Hook-Based Enforcement](field-guide.md#hook-based-enforcement).

**What the agent can reach** — restricted user accounts, filesystem boundaries, network access policies (no egress, or allowlisted destinations only), container or VM isolation, no secrets in environment scope. Higher setup cost; pays off when the blast radius is large or the stakes are high. See [Runtime and Environment Isolation](field-guide.md#runtime-and-environment-isolation).

The two layers compose. Layer 1 protects against the agent invoking a bad command. Layer 2 protects against the agent finding a way to do something bad regardless. For low-stakes work in a worktree, Layer 1 is usually enough. For production work, customer data, or infra mutations, both layers earn their cost.

## 3. Contributing and Sharing

The field changes fast enough that anything written here might be wrong by the time someone reads it. Velocity comes from posting what you learned — a prompt that worked unusually well, a setup that failed in an unexpected way, a tool that surprised you, a hook that caught a near-miss — not from discovering it alone and moving on. Discoveries that die in a chat DM or a personal notes file benefit no one. The norm here is simple: if you found something worth knowing, write it up and share it. That is how this doc improves.

This is opinionated guidance, not policy. The bar for adding a new pattern or playbook is low — write up what happened, what you tried, and what worked. The bar for changing a core principle is higher — a proposed change to §2 should include the reasoning, not just the edit.

**How to contribute:**

- If a prompt, setup, or workflow worked unusually well, write it up as a pattern or playbook in [`field-guide.md`](field-guide.md). A short description plus the steps you followed is enough to start.
- If something failed — a model got confused, an agent went sideways, a setup cost you time — document what went wrong and what you'd do differently. Failure notes are just as useful as wins. **For specific agent failures (missed a spec, didn't understand a package, repeated a mistake), follow [Closing the Agent Failure Loop](field-guide.md#closing-the-agent-failure-loop) — fix the docs/skills/instructions, then verify with a fresh agent.**
- If something here is out of date or wrong, open a PR with the correction and a one-line explanation. For patterns and playbooks, the bar is low; for a core principle in §2, include the reasoning.
- If you hit a near-miss that a hook or allowlist could have prevented, add it to the Hook-Based Enforcement or Permissions and Allowlists patterns in [`field-guide.md`](field-guide.md) so others can wire up the same guardrail.
- Do not let discoveries die in a chat DM. If it was worth typing once, it is worth landing somewhere that survives the conversation.
