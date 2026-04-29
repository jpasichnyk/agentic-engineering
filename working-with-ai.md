# Working with AI

> Status: draft. Living doc — expect change.

## Why this doc exists

The other docs in this repo are operational. `best-practices.md` gives you six principles for doing agentic work well. `field-guide.md` gives you patterns and playbooks you look up when you're mid-task. This doc sits above both of them — it's about the calls you make before you open an agent at all.

Those are different decisions, and they belong in a different doc. Should this work be an AI project at all right now? How many parallel streams should I actually be running? How big should the task I hand off be? Where does automation replace agents? These are judgment calls — they don't have checklists. They need a slightly different mode of thinking: more deliberate, more aware of org context, more willing to say "not yet" or "not this."

This doc is for engineers who are already doing agentic work and starting to develop opinions about it — and for anyone who leads or influences how engineering time gets allocated. The operational docs tell you how to run a clean session. This one is about whether to run it.

---

## Treat AI projects as real projects

Agentic work feels cheap in the moment in a way that distorts judgment. You — or your company — pay a real bill: SaaS plans, API spend, seats. But at the moment you decide "let me try this thing," that cost is invisible. It only surfaces later, as a line item on a card statement or in a monthly review you (or someone else) eventually scans. The bigger cost is harder to count: your time, your energy, the brain power you're spending here instead of elsewhere, and the work you would have been doing otherwise. None of that shows up anywhere until the thing you said you'd do — the roadmap item, the side project, the writing, the workout, the conversation — starts to slip.

If you're working against any kind of committed deliverables — your own roadmap, your team's, a client's, or just the personal project you said you'd ship by month-end — unplanned AI exploration is still unplanned work, and AI work is especially good at hiding how much of it you're signing up for. What started as "I'll just take a look for a couple of hours" becomes a couple of days, and then a couple of weeks, because the next promising lead always seems an hour away. Spend two weeks on an AI-driven detour that wasn't on the plan and the things that needed that time didn't get them. The AI didn't make those weeks free. It made them feel free. Initially.

The right pattern is *time-boxed* exploration on *big-bet* investigations — things that, if they pay off, could rightfully displace items already on your plan. Each gets a defined window: a shallow proof-of-concept inside that window, enough to validate the idea and write it up. Then it becomes a real proposal — scope, schedule, placement against committed work. The goal isn't fitting exploration around the edges of committed work; it's finding the few investigations that *should* end up demanding higher priority because the upside is that real. Everything that doesn't clear that bar gets dropped, even if it looks interesting in the moment. Don't chase every squirrel.

This isn't about discouraging experimentation. It's about being honest that "I just want to try this thing" has a cost — in money, time, attention, and the things you would have done otherwise — and that cost deserves the same prioritization as everything else you've committed to. Treating AI work as a real project, not just something you'll squeeze in, is what keeps exploration from silently crowding out the work already on your list.

---

## Don't dilute your attention

There are two kinds of parallelism in agentic work, and they have very different costs. Running multiple agents on parts of *the same* piece of work — frontend changes here, backend changes there, both feeding one feature — is divide-and-conquer. You hold one mental model; the agents just execute pieces of it. Running multiple agents on *different* pieces of work — one on a dashboard tweak, one on a backend service refactor, one on a website investigation, one on a tool you've been meaning to build — is something else entirely. Each stream is its own context, its own mental model, its own decision points.

The second kind is where the trap lives. It's incremental: you start Stream A on the dashboard. While waiting for an approval, you remember the backend thing and start Stream B. While B runs, the website investigation seems quick so you start Stream C. Now you're fielding questions and approving actions across three or four unrelated contexts, and none of them are getting the consideration they deserve. The approval click on Stream 3 at the moment you're mentally in Stream 1 is the one that gets the careless read. Rubber-stamping — the reflexive "LGTM" on something you didn't really look at — isn't a hypothetical failure mode; it's the predictable result of attention spread across four different problems.

Same-work parallelism is usually fine. Cross-context parallelism is usually not. Match concurrency to actual available attention — and especially to the number of distinct contexts you're trying to hold at once. A stream that needs frequent human approval needs focused attention, which means fewer concurrent contexts, not more.

A useful proxy: if a stream doesn't deserve to be on your screen — visible, in your peripheral vision, ready for you to engage with it — you probably aren't really engaged with it, and your decisions there will be worse. Tools like tmux with split panes make this honest: if you can hold a 2-pane split in your head and stay sharp, two is your number. If three is a stretch, two is still your number. Buried in tab 17 isn't running in parallel; it's running unsupervised. The number of streams that fit on screen at once is rarely a bad cap on how many to run.

For the operational mechanics of parallel agents, see [Parallel Agents Without Collisions](field-guide.md#parallel-agents-without-collisions). This essay is the judgment side: how many *contexts* to run in parallel is a decision about your attention, not about the number of worktrees you can open.

---

## Right-size what you delegate

Task granularity is a real decision. Too small and you're paying coordination overhead — short turns, constant permission prompts, handoff cost that swamps the actual work. Too big and several things degrade at once: the goal drifts mid-stream, scope creep enters at execution, tokens go in the wrong direction — and accumulating context starts evicting the rules the agent was operating under. A long session, or several different sequential tasks crammed into one session, is where prose guidance (CLAUDE.md, AGENTS.md, the decisions established at the start) quietly stops being honored. Not because the agent is being defiant — because the part of the window that held those rules has been pushed aside by everything that came after.

The right size is roughly "one clear deliverable an agent can produce in a focused session without losing track of the goal." Reliable tells at both ends: too small when you're approving every tool call or the session ends before setup amortizes; too big when the work goes inconsistent mid-stream, the plan-then-execute discipline breaks down because the executor starts re-planning, or scope quietly shifts between the first turn and the last.

Neither failure mode announces itself loudly. You notice them in retrospect: the session that felt like constant interruption, or the one that produced something you had to redo because it drifted gradually off course. Calibration happens through doing the work — noticing where friction lands and adjusting. This is judgment, not formula.

The fix when something is too big is the same shape as Principle 1: have an agent write the plan down — incrementally if the work supports it — then hand each chunk to a fresh session. Manage the handoff manually or via an orchestrator dispatching subagents; what matters is that the executor doesn't carry the planning session's accumulated context. Each chunk gets a clean window and the relevant slice of the plan, nothing more. The rules the agent operates under are honored because they were just loaded, not buried under thirty thousand tokens of prior turns. For the operational pattern, see [Plan-Then-Execute](field-guide.md#plan-then-execute).

---

## Use scripts where agents don't belong

Agents are for judgment, exploration, and natural-language reasoning. Scripts are for deterministic, repeatable, mechanical work. When you find yourself rinsing and repeating an agent through the same kind of task, that's the signal: the deterministic parts want to be extracted.

A script gives the same result every time, costs nearly nothing per call, and is testable and auditable. An agent loop over the same mechanical task is token-expensive, non-deterministic, and brittle at the edges. When something goes wrong with a script, you can trace exactly what happened. With a freeform agent loop, you have a transcript.

The pattern: run the task with an agent a couple of times in close HITL mode — you guiding, the agent executing, both of you watching what actually happens — until you understand it well enough to know what's mechanical and what isn't. You can't extract the deterministic parts until you've seen them in action. Identify what's genuinely mechanical: the file transformations, the API calls with known inputs, the templating, the validation checks. Write scripts for those. Then let the agent orchestrate at the level where judgment matters, calling the scripts rather than re-deriving the mechanics each time. This is true even when an agent is orchestrating other agents: a few tokens to invoke a script and read its result is dramatically cheaper than the hundreds or thousands an agent burns doing the same mechanical work itself — and the script's output is deterministic where the agent's is variance. The agent gets smaller, cheaper, and more reliable.

This is a corollary to [Principle 6](best-practices.md#6-enforce-with-hooks-and-config-not-prose): "enforce with hooks, not prose" generalizes to "do mechanical work with scripts, not agent turns." See [Run-and-Iterate Loop](field-guide.md#run-and-iterate-loop) and [Repeatable automation — onboarding a new retailer](field-guide.md#repeatable-automation--onboarding-a-new-retailer) for where this pattern shows up in practice.
