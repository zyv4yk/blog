---
title: "Treating AI Coding Tools Like a Platform Product"
date: 2026-06-05
description: "Notes on shipping AI assistant skills, agents, hooks, and MCP context as a versioned platform instead of letting everyone run their own prompts. With evals and a couple of numbers that actually moved."
tags: ["AI Tooling", "Claude Code", "MCP", "Developer Experience", "Platform Engineering", "Code Review"]
draft: false
---

Everyone on the team has an AI assistant in their editor now. Good, mostly. The problem is that everyone's assistant knows something slightly different. One person's Claude knows our migration conventions. Another's invents a new folder layout every time. One reviewer catches a missing Helm `--atomic` flag, another waves the same PR through. Same model, very different results.

This is the [distributed-pipelines problem](/posts/ci-cd-platform-at-scale/) again, one layer up. Back then it was copy-pasted `.github/workflows/` drifting across repos. Now it's copy-pasted prompts and knowledge that lives in people's heads. Same fix too: package it and ship it like any other internal tool, instead of leaving everyone to run their own prompts.

Here's roughly what that looks like.

## The shape of the toolkit

Modern AI coding assistants (Claude Code, Cursor, Codex) give you a few primitives you can build on instead of just chatting:

- **Skills**: focused, reusable procedures ("add a migration", "open a PR from our template") that the assistant pulls in on demand based on what you ask.
- **Agents**: subagents with their own system prompt and tools, dispatched for a bounded task (a code review, a research sweep) and returning a result.
- **Hooks**: deterministic code that runs around tool calls, so you can block or rewrite what the model is about to do.
- **MCP servers**: typed connections to your real systems (GitHub, Confluence, Kubernetes) so the assistant works from live data instead of guessing.
- **Commands**: thin named entry points (`/review`) that wire the above together.

The mistake is using these one developer at a time. The leverage is in packaging them and shipping them like any other internal tool.

## What you actually reach for

Those are abstractions. At some point you pick real products to implement them. Here's the kit I keep open, grouped by the job each one does:

| Job | What I reach for | Why it earns the slot |
|---|---|---|
| Agentic workhorse | Claude Code | Holds a plan across a dozen tool calls without losing the thread; the cleanest place to load skills, agents, and hooks |
| Unattended work | A background coding agent (Codex, GitHub's coding agent, Claude Code on the web) | Hand off an issue and get a PR back while my laptop's closed; the unit of work becomes a queued job, not a chat |
| Pairing editor | A JetBrains/IntelliJ IDE with an AI assistant plugin | For when I want to *see* the codebase while I steer, with inline completions and edits and a human firmly in the loop |
| Second opinion | Codex | Different model, different blind spots; handing the same diff to a second engine catches what a single-vendor habit won't |
| Always-on PR review | Copilot Code Review, plus a custom standards-grounded reviewer | The cheap smoke detector that flags the obvious before a human (or a heavier agent) ever looks |
| Live system access | MCP servers (GitHub, Confluence/Jira, Kubernetes/Helm, Slack) | Typed access to real state, so the assistant reads this quarter's reality instead of hallucinating last quarter's |
| Deterministic glue | `gh`, `kubectl`/`helm`, `jq`, plain `bash` | The fallback muscle for hooks and for every environment where MCP isn't wired up, which on a real laptop is most of them |
| One choke point | an LLM gateway in front of the model API | Keys, rate limits, model routing, and cost logging in one seam, so nobody pastes a raw key into their shell |

The bottom two rows are the ones you own. The runtimes and the model are interchangeable; the gateway and the glue are what turn a toy assistant into something the team can actually rely on. I pick by the job in front of me and don't get attached to a particular vendor. Honestly the model is the easy part.

## Distribution: a marketplace, not a wiki page

Instructions on a wiki page rot. Instructions packaged as a versioned plugin get installed, updated, and rolled back like code.

So you need a marketplace. Publish the toolkit to one, and any repo opts in with a single line:

```text
/plugin marketplace add org/ai-skills
```

Then version it like any other dependency: semver, a changelog, the ability to pin or roll back. An update becomes a deliberate bump, not something that lands in everyone's editor overnight. The exact mechanics matter less than the rule: one source of truth, shipped on a release cadence instead of by copy-paste.

The formats are converging on open standards, which helps. The same `SKILL.md` and the same MCP server increasingly work across assistants and vendors, not just one. So I build a capability once and it mostly travels: across editors, across the team, and across whichever model happens to be best this month.

In practice that means one reference in a service repo, and the team gets the same conventions, the same PR template, the same review bar. When the standard changes, it changes everywhere on the next update.

## The orchestration pattern: command, agents, skills

Honestly the part that took longest wasn't any single skill, it was getting them to compose. The pattern that stuck: a thin command fans out to one or several agents, and each agent pulls in whatever skills its slice of the job needs.

```text
/review                       # command: thin entry point, fans out
  ├─ security-reviewer        # agent, runs in parallel
  │    ├─ scan-dependencies   # skill
  │    └─ check-secrets       # skill
  ├─ infra-reviewer           # agent, runs in parallel
  │    ├─ lint-helm           # skill
  │    └─ check-migrations    # skill
  └─ open-pr                  # skill, runs once every review passes
```

The command is deliberately dumb. It routes and decides how many agents to spin up. Each agent owns one concern and runs in parallel: one reads the diff for security, another for infra, another for migrations. None of them carries the whole rulebook. Each loads only the skills and checklists its concern needs, then reports back. When every review passes, a final skill does the mechanical PR creation, and any other workflow can reuse that skill.

Keeping each agent narrow and pushing detail into skills it loads on demand is what keeps the fan-out fast instead of bloated. Each agent only gets the context for its own slice, so you're not paying for twelve checklists it never needed.

I've started running the same fan-out unattended: several background agents in their own git worktrees, working through a queue while I'm elsewhere, watched from one dashboard. Once "how many can I spin up" turns into a capacity question, you can't really skip the guardrails.

## Grounding the assistant in your standards

A general model knows general best practices. It doesn't know the naming convention your org settled on in an internal RFC, or that your deploy workflow has to request a specific set of permissions. That knowledge has to be injected, and MCP is how you do it without copy-pasting.

The review agent fetches the relevant internal standards and design docs live, then drops them into the prompt as context. I don't tell the model to "go find the rules" and let it loop around tool calls; the rules just arrive alongside the diff. It's simpler, faster, and the trace stays readable: fetch the standards, then make one well-grounded call.

Because these tools run in messy environments (laptops, CI, containers), every external dependency gets a fallback ladder:

```text
1. Required: hard stop if both the CLI and the MCP server fail
2. Degraded: reduced context, warn the user, keep going
3. Optional: skip silently
```

The review agent actually inverts the usual order and tries the `gh` CLI first, MCP second, because it runs where MCP often isn't configured. The degraded path is the one I spend the most time on. It's the difference between a demo and something people will leave running when they're not watching it.

## Guardrails: hooks, not vibes

The most reassuring thing you can add is a hook that makes a class of mistakes impossible, no matter what the model decides. A `PreToolUse` hook sees every command before it runs and can veto it:

```bash
# PreToolUse hook: block destructive git operations
input=$(cat)
cmd=$(echo "$input" | jq -r '.tool_input.command // empty')

if echo "$cmd" | grep -qE 'git (push --force|reset --hard|clean -fd)'; then
  echo "Blocked: destructive git command" >&2
  exit 2   # exit code 2 tells the assistant to block the call and explain why
fi
```

Pair that with permission modes (auto-accept edits, prompt the human for writes and shell) and you get an assistant that's fast on the safe stuff and asks before the scary stuff. The model's autonomy is a dial, not a switch.

The dial matters more once agents run without me watching. A background agent burns tokens at full speed whether or not it's getting anywhere, and a runaway session can rack up a frightening bill before anyone looks. So the unattended setup gets extra guardrails: a cap on how many run at once, hard termination conditions, per-session cost logging, least-privilege sandboxes, and a human gate before anything reaches `main`. It's the same choke-point instinct as the gateway, just turned up for code I never watched get written.

## Treat skills like code: evals and impact

If you ship a skill, you should be able to answer two questions: does it trigger when it should, and did it actually help?

Triggering is testable. Each skill carries a small eval set (positive and negative queries interleaved), and a runner checks whether the right skill fires:

```bash
bash skills/tests/run_evals.sh --parallel 5
```

A few things that measurably moved recall without hurting precision:

- **Interleave** positive and negative queries so train/test splits stay balanced.
- **List keywords and symptom phrases** in the description ("exit code 1", "permission denied", "why did this fail?"), not just a tidy summary.
- **Add explicit `DO NOT use for X` boundaries** so neighboring skills stop overlapping.

In practice the set runs with high precision (skills almost never fire when they shouldn't) and strong recall. Precision is the one I care about more, because a skill that fires when it shouldn't is worse than no skill at all.

I run that suite as a CI gate now, not a one-off I remember to check. A skill whose triggering regresses fails the build like any other test. I've started doing the same thing a level up: write the spec, let the agent execute, then keep the spec as the versioned artifact and the eval as its regression test. It's cheap insurance against the wall that ad-hoc prompting tends to hit a few months in.

Impact is the harder number, and the one that matters. The clearest signal came from rolling standards-grounded AI review across one repo and measuring before/after over several hundred merged PRs:

| Metric                    | Direction        |
|---------------------------|------------------|
| Mean time to merge        | <span class="trend trend-good">↓</span> roughly half   |
| Share merged within a day | <span class="trend trend-good">↑</span> several points |
| Review coverage           | majority of PRs  |

The cut in time-to-merge isn't really the model being clever. The boring routine comments just show up in seconds instead of waiting a day for someone to get to the PR, and people got to spend their review time on the parts that actually needed judgment.

## Lessons learned

The technical pieces are the easy part. Like the CI/CD platform, the real work is adoption.

1. **Make it the lazy option.** Nobody adopts a tool that's more work than doing it by hand. The plugin has to be genuinely faster than typing the prompt yourself, or people just won't reach for it.
2. **Design for the degraded path.** MCP will be down, the CLI will be unauthenticated. A tool that fails loudly and helpfully beats one that's brilliant only under perfect conditions.
3. **Put guardrails in deterministic code.** Trust comes from "it literally cannot force-push", not from "the prompt asks it not to."
4. **Measure, or it didn't happen.** Eval recall and time-to-merge are what turn "I think this helps" into something another team will actually buy.

None of this is new discipline. Versioning, fallbacks, guardrails, evals, a metric you can point at. It's the same stuff that made the CI/CD platform something people trusted. AI tooling is just another platform product with your own developers as the users, so I've been treating it that way and it's held up so far.

## Related reading

- [Building a Centralized CI/CD Platform for Microservices](/posts/ci-cd-platform-at-scale/): the platform-as-a-product playbook this post borrows its shape from.
- [Observability as Code](/posts/observability-as-code/): measuring impact with objective metrics instead of vibes.
