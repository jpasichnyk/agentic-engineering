# Best Practices for Agentic Engineering

> Status: draft. Living doc — expect change.

## 1. Why this doc exists

Most engineers using AI today live in chat windows and IDE autocomplete — useful, but a fraction of what AI agents can actually do. A smaller group has pushed past that into real agentic work: multi-step execution, clean-context separation, orchestrated workflows. None of what's in here is secret knowledge — anyone who spends enough time in the trenches with AI agents will arrive at most of it on their own. Writing it down is a head start, a way to compress the learning curve into something readable in an afternoon.

The field is multi-vendor by design. Different tasks call for different models, and the right answer at any given time depends on context, not loyalty. This doc is vendor-neutral on purpose: the principles here apply regardless of which assistant or IDE you're in. Alongside the principles, it sets guardrails around autonomy, security, and data hygiene — opinionated enough to be useful, not heavy enough to replace judgment.

This is a living doc. "Written down" does not mean "frozen." The field changes quickly, and anything here that stops reflecting reality should be updated. If you've found something that works — or something that doesn't — say so. For patterns, playbooks, and concrete how-to guidance, see [`field-guide.md`](field-guide.md). For a personal progression path from chat user to multi-agent work, see [`onboarding-ladder.md`](onboarding-ladder.md).

## 2. Core Principles

### 1. Manage context aggressively

Quality of agent output degrades as context grows. Even at 1M tokens, degradation is reliable past roughly 50% fill; rules set in CLAUDE.md / AGENTS.md get forgotten past ~100k tokens. Task size is part of the same problem — one over-large task accumulates as much context as several sequential ones in the same session, with the same rule-forgetting and quality drops. Cost rises, turns get slower, and — critically — mixing planning, building, testing, and verification in one session creates failure modes that are hard to detect, like an agent adjusting tests to make itself succeed rather than catching a real regression. Stale, mixed-purpose context is worse than a smaller, curated one.

We default to clean separation across phases: plan in one session, build in another, verify in a third. Two implementations both work — subagent forking with curated context handed off explicitly, or human orchestration via copy-pasting the finished plan into a fresh session. For the how-to, see [Plan-Then-Execute](field-guide.md#plan-then-execute) and [TDD with Separated Agents](field-guide.md#tdd-with-separated-agents). The guiding instinct is "curate, don't pile on" — what you give the agent matters more than how big the window is.

Permissions are part of the same surface. Hand the agent the context it needs and the permissions it needs to act on that context — no more. Curated permissions reduce friction (fewer prompts on safe actions) and reduce risk (fewer routes to dangerous ones). See [Permissions and Allowlists](field-guide.md#permissions-and-allowlists).

### 2. Match autonomy to phase × blast radius

How much latitude to give an agent depends on two axes. The first is *phase of work*: design, discovery, and planning benefit from HITL — the back-and-forth is the output, not overhead to eliminate. Autonomous execution belongs once the plan is locked. The second axis is *blast radius*: how bad is the worst case? Irreversible operations — production deploys, schema migrations, force-pushes, hard deletes — demand tight controls regardless of how confident the agent sounds. Reversible local work — tests, feature-branch code, docs — earns more latitude.

The two axes multiply, and that multiplication is the insight worth internalizing. A destructive action inside a planning conversation is usually fine: the agent is describing what it would do, and nothing executes. The same action inside a long-running autonomous loop is dangerous. We set autonomy at the *intersection*, not at either axis alone. In practice that means enumerating explicit allow and deny lists per project before kicking off autonomous work — especially for CLIs with destructive subcommands. Prose rules drift as context fills; tool-level enforcement does not. See [Permissions and Allowlists](field-guide.md#permissions-and-allowlists) and [Hook-Based Enforcement](field-guide.md#hook-based-enforcement).

### 3. Verify with fresh eyes

An agent that has worked through a long session develops blind spots. Mistakes made early shape the framing for everything that follows — a wrong assumption in the plan becomes an invisible constraint in the build. The agent cannot reliably catch its own gaps in the same context where it made them. The same reasoning that produced a mistake is the reasoning that evaluates whether it was one.

A fresh-context reviewer — same model or different — sees only what a human would hand it: the spec or plan, the readme, and the diff, with no in-session framing and no stake in prior decisions. Cross-model review is especially powerful: different models have different failure modes, so agreement is genuine confidence and disagreement surfaces a decision for the human. This applies to code review and plan review equally — fresh eyes catch ambiguity, missing requirements, and contradictions before they become bugs. The discipline is to not coach the reviewer into agreement; give it the same artifacts a human reviewer would get, nothing more. See [Cross-Model Code Review](field-guide.md#cross-model-code-review).

### 4. Service boundaries enable agentic looseness

The amount of human review an AI-generated change requires scales inversely with how well-bounded that change is. When a service has strong contracts — explicit API surfaces, integration tests, and production smoke or regression tests — the question shifts from "did the agent write the right code?" to "does the service still honor its contract?" That second question is mechanical. The first requires a human to hold the full intent of the change in their head and reason through every line. Architectural investment in service boundaries is, directly, an investment in how fast and safely agentic work can move.

Inside a well-bounded service, internal style and detailed design choices matter less — the agent has room to operate via [Run-and-Iterate Loop](field-guide.md#run-and-iterate-loop) and [Pin-Then-Modify](field-guide.md#pin-then-modify) without close oversight of every decision; the contract tests answer what matters. Without that isolation, every change risks cross-cutting effects, and human review becomes a bottleneck that scales with agentic output. The investment in boundaries and contract coverage is not just good software hygiene — it is the enabling investment for agentic engineering at scale.

### 5. Tests are leverage

Tests are not a tax on agentic work — they are what makes agentic work safe and fast. The most important habit is [Pin-Then-Modify](field-guide.md#pin-then-modify): before changing existing code, capture its current behavior in tests. Then make the change. Then confirm the tests are still green. An agent cannot reason about "what should still work" — it doesn't have that institutional knowledge — but a test suite does. Pinning behavior before modification converts that unknown into a mechanical check the agent can act on.

Narrow unit tests give the agent a narrow window. To converge on working software, push the test surface further: integration tests, browser tests via Playwright or Selenium, Docker-based end-to-end loops. Real failures from real execution give the agent the feedback it needs. The [Run-and-Iterate Loop](field-guide.md#run-and-iterate-loop) is built around this: agent runs tests, sees failures, makes changes, reruns, converges. That loop is more powerful than any single round of code review because it is objective, repeatable, and immune to the same blind spots that produced the bug in the first place.

### 6. Enforce with hooks and config, not prose

Prose rules in CLAUDE.md, AGENTS.md, and system prompts are advisory. They are read once at context load, then diluted as the session accumulates tokens — the same degradation described in Principle 1. A rule written at the top of your project file has no guarantee of being honored in turn 80. The agent is not being deceptive; the rule just no longer has weight in a context that has moved on. This is the prose-rule-bleed problem, and it is invisible: you will not know the rule was forgotten until something breaks.

Hooks and permission settings are deterministic. A pre-commit hook that blocks `git pull --rebase` fires every time, regardless of what the agent remembers. An allowlist that restricts file access to `~/projects` does not depend on the agent's current context window. When you can encode a rule mechanically — don't touch files outside the project root, always run the linter before commit, block destructive subcommands of a CLI — encode it that way and remove it from prose. Reserve prose for genuinely contextual judgment calls: style preferences, voice, when to pause and ask. The reliability gap matters: a hook that fails to fire is a bug you can detect; a prose rule that silently fails is invisible. See [Hook-Based Enforcement](field-guide.md#hook-based-enforcement) and [Permissions and Allowlists](field-guide.md#permissions-and-allowlists).

## 3. Contributing and Sharing

The field changes fast enough that anything written here might be wrong by the time someone reads it. Velocity comes from posting what you learned — a prompt that worked unusually well, a setup that failed in an unexpected way, a tool that surprised you, a hook that caught a near-miss — not from discovering it alone and moving on. Discoveries that die in a chat DM or a personal notes file benefit no one. The norm here is simple: if you found something worth knowing, write it up and share it. That is how this doc improves.

This is opinionated guidance, not policy. The bar for adding a new pattern or playbook is low — write up what happened, what you tried, and what worked. The bar for changing a core principle is higher — a proposed change to §2 should include the reasoning, not just the edit.

**How to contribute:**

- If a prompt, setup, or workflow worked unusually well, write it up as a pattern or playbook in [`field-guide.md`](field-guide.md). A short description plus the steps you followed is enough to start.
- If something failed — a model got confused, an agent went sideways, a setup cost you time — document what went wrong and what you'd do differently. Failure notes are just as useful as wins.
- If something here is out of date or wrong, open a PR with the correction and a one-line explanation. For patterns and playbooks, the bar is low; for a core principle in §2, include the reasoning.
- If you hit a near-miss that a hook or allowlist could have prevented, add it to the Hook-Based Enforcement or Permissions and Allowlists patterns in [`field-guide.md`](field-guide.md) so others can wire up the same guardrail.
- Do not let discoveries die in a chat DM. If it was worth typing once, it is worth landing somewhere that survives the conversation.
