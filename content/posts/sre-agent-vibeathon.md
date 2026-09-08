---
title: "An SRE Agent in Two Days: What Our Vibeathon Was Really Like"
date: 2026-09-07
description: "Artem Chernenko and I joined the AI Agents.PRO vibeathon with a rough idea for an SRE agent. A coding agent gave us a working system quickly. The real work was turning that first draft into something we could test, explain, and trust."
tags: ["AI Tooling", "Agents", "SRE", "Evals", "Claude Code"]
draft: false
---

This project started with a short message from [Artem Chernenko](https://github.com/timmy1496): "Should we form a team for the vibeathon?"

"Let's do it," I replied. We did not have a project idea yet, so we started looking at problems we knew from work. A few messages later, we had one: an SRE agent.

That was on August 17. At that point we had no architecture, no repository, and no plan. We just had a familiar problem: when an alert fires, an engineer spends the first part of the incident collecting the same context from the same places. Metrics, logs, recent deploys, runbooks, old postmortems. Much of that work is repetitive, but it still takes time, especially if you are on call at three in the morning and have never seen the service before.

A couple of weeks later, that chat became our final project for the [AI Agents.PRO](https://fwdays.com/en/event/ai-agentspro-course-2) course at fwdays Academy.

The format was a vibeathon: two people, two days, and a jury of two SREs and a CTO. Using coding agents was not a shortcut we had to hide. It was the point of the exercise.

The Slack quotes in this post are translated from Ukrainian.

## From a chat to a brief

On September 1, I shared a small Google Doc with Artem. It was not a specification. It was a rough list of things an SRE agent might do:

- react to alerts;
- look for a root cause;
- search company knowledge, postmortems, and runbooks;
- review logging and alerts;
- watch metrics after a release.

There were also three technology names at the bottom: LangChain, LangSmith, and Langfuse. That was about the full depth of the first draft.

Roughly half an hour later, I sent a proper brief to our Slack channel. It split the idea into five roles, described a demo environment, added evals and safety boundaries, and included a plan for one or two days. I wrote to Artem: "It came out like this. I haven't gone deeper yet. What do you think?"

His answer was basically: looks great, maybe we can use `kind`.

That exchange set the tone for the rest of the project. We did not spend hours trying to make the plan perfect. We wrote down enough to start, agreed on the important boundaries, and left room to change the implementation once it met reality.

One decision helped more than expected: we wrote down what to cut first if we ran out of time. Dashboards would go first, then PR delivery, then the service reviewer, then the release monitor. The knowledge base, incident investigation, evals, and tracing were the core. They stayed.

The original idea, the final brief, and the list of what we actually completed are all in the [public repository](https://github.com/timmy1496/ai-agents-pro-vibeathon).

## The first night: let the coding agent run

On the evening of September 2, Artem created the repository and started a coding agent against the brief. His Slack message was wonderfully casual: "I started the agent. Check what is there in the morning."

The next morning, I opened the repository and initially thought nothing had happened. I had misread the timestamps because I was half asleep. Artem replied: "Everything is in main."

The agent had spent about three hours generating the project. By morning there were thirteen commits: the demo environment, synthetic services and postmortems, five agent roles, test cases, an evaluation runner, and a green test suite.

Then Artem added the sentence that described the situation perfectly: "I haven't had time to test it by hand."

That is probably the most honest checkpoint in any AI-generated project. A lot of code exists. Its own tests pass. Nobody knows yet whether the thing works.

Around lunchtime, Artem ran the first complete incident. The demo service started returning errors, an alert fired, the bot investigated it, and a report appeared with the correct explanation. The test suite had 136 green tests. For a moment, it looked almost too easy.

The live environment also exposed four problems that the tests had missed. One container never started. The knowledge base could exist while containing no data. Two components were on incompatible versions. Tracing failed because a demo email address did not pass validation.

None of these problems was interesting by itself. What mattered was that the system could look healthy while quietly losing part of its context. The agent would still answer. It would simply answer with less evidence than we thought it had.

That changed the rest of our two days. From that point on, the job was no longer "build an agent." It was "find out where this agent can lie to us without technically failing."

## How we split the work

We never had a formal task board. The split happened naturally in Slack.

Artem kept the live path moving. He worked on the demo environment, incident flow, and the real Slack integration. He made alerts open their own threads, kept follow-up questions in the same context, and used Slack Socket Mode so the bot did not need a public URL or ngrok.

I focused on reviewing the repository against the brief, strengthening the evals, documenting the decisions, and turning the project into a story we could present. I also had a day full of meetings, so a lot of my work happened in short gaps with Claude reading the repository and me checking its findings.

At one point I dropped an AI-assisted repository review into Slack and immediately added: "Sorry, this is a bit AI, but I don't have time to do it another way."

The review found four real defects. The most serious one was in our main safety claim. We said destructive actions were blocked before a human saw them. In the actual execution order, the human approval step happened first. The README, the brief, and a green test all told a better story than the code did.

The other findings were less dramatic but just as real: a baseline window included the incident itself, the service reviewer could give a healthy service an F, and some tests passed only because another test file had left state behind.

I sent the findings. Artem asked whether I could tune the project. I said yes and asked the coding agent to help fix the defects and extend the evals. Meanwhile, Artem kept working on the demo and Slack.

By late evening our messages had become very short:

> "I think I'm more or less ready for the demo."
>
> "Pull my master."
>
> "The deck is almost okay. I'll finish it tomorrow."

That was our coordination process. One shared brief, one repository, one Slack channel, and frequent small checks that neither person's changes had broken the other's work.

## What we actually built

The result was an SRE assistant that takes over the first, repetitive part of an incident.

When an alert arrives, it gathers information about the service, checks what changed, compares the current metrics with the previous period, looks for common error patterns, and searches for similar incidents. It then posts a hypothesis, evidence, and suggested next steps in the Slack thread.

The system also answers questions from the knowledge base, reviews a service's logging and alert coverage, and checks metrics after a release. Internally, those jobs are split between several specialized agents and deterministic workflows. For the person on call, however, there is only one entry point and one thread.

The important boundary is simple: the agent can read, investigate, and suggest. It cannot change infrastructure on its own. Any action goes through a human.

That was enough technical scope for the vibeathon. We did not need every planned integration or a production-ready platform. We needed one end-to-end scenario that worked, showed its evidence, and failed in ways we could see.

## The part the coding agent did not solve

The coding agent gave us the skeleton quickly. Most of our time went into deciding whether we should trust it.

We built the evaluation process in two parts. A fast offline check ran on every change and verified that each test case contained enough evidence to support its expected answer. A second run used a model to grade the quality of the final reports.

That sounds like an implementation detail, but it became the most useful part of the project. The evals caught problems in the measuring system itself. One case punished the agent for giving the correct answer because its label was wrong. One metric had a name that did not match what it measured. A report could identify the right cause and still give advice too vague to help an engineer at three in the morning.

Our best demo came from changing one log field from `msg` to `message`. Nothing crashed. The metrics were still there, the tools returned successfully, and the agent still answered. But one important signal disappeared and the conclusion became `unknown`. The eval gate turned red.

That was a much better demonstration than another happy-path chatbot. It showed that the system could detect a quiet loss of quality, not just a visible exception.

## Building the pitch while building the product

The presentation changed how we thought about the project.

Instead of opening with architecture, we opened with the job the agent would take over: the first twenty minutes of an incident. The deck then explained the recurring cost of gathering context, what the engineer would see in Slack, why the answer could be checked, and where the human remained in control.

We deliberately framed it as a pilot proposal rather than a hackathon demo. The plan had three steps:

1. Run it only on the synthetic demo environment.
2. Put it in shadow mode for one tier-2 service, writing to a separate Slack channel while the on-call engineer works normally.
3. Move it into the real incident thread only if the shadow phase proves useful and safe.

We also defined the measurements before the pilot: time to the first grounded hypothesis, how often the on-call engineer confirms it, how often the report needs rewriting, and cost per resolved incident. Stopping after the shadow phase was a valid result, not a failure we planned to explain away later.

That became the real pitch. We were not asking anyone to believe that our two-day agent was ready for production. We were showing a controlled way to find out whether the idea deserved a production pilot.

## The last hours

The morning before the pitch was still busy. I finished the deck and added our links. Artem prepared the demo. I asked him to run it again from the latest `main` branch because I wanted to be sure my changes had not broken his setup.

About an hour before the pitch, the repository was finally frozen. The last bug was almost poetic: two parallel actions in the same Slack thread could overwrite each other's state. We found it because the bot had a `JOKE` intent and someone asked for a joke while an investigation was still running.

For a few minutes in the project's history, the bot automatically posted a context-aware joke during every incident. We quickly agreed that an agent volunteering humor at three in the morning was a different product. Jokes became request-only.

The final repository had 63 commits created in under 48 hours, more than 300 offline tests, 24 evaluation cases, 9 architecture decisions, and a full demo environment. Those numbers are fun, but they are not the part I remember most.

I remember how quickly a rough idea from our chat became a brief. How a coding agent produced more code overnight than we could safely review the next morning. How the first live run was both successful and full of failures. How Artem and I kept handing the project back and forth between meetings, tests, Slack messages, and the deck until it became one coherent story.

## What I took away from it

- **A shared problem is more useful than a clever agent idea.** We both understood the first twenty minutes of an incident, so we could make decisions quickly without explaining the domain to each other.
- **Coding agents compress implementation time, not verification time.** Three hours were enough to generate a large working skeleton. The rest of the vibeathon was about finding out what "working" actually meant.
- **A written brief makes parallel work possible.** We did not need constant meetings because the boundaries, priorities, and cut order were already written down.
- **Live runs tell a different truth than unit tests.** The first successful demo also revealed the failures that mattered most.
- **The presentation is part of the engineering.** Turning the demo into a pilot proposal forced us to explain the value, limits, and stopping conditions more clearly than the architecture diagram ever could.

Artem suggested that we enter the vibeathon, and together we chose to build an SRE agent. A coding agent wrote much of the first version. What made the project ours was everything that happened after that.

## Related reading

- [timmy1496/ai-agents-pro-vibeathon](https://github.com/timmy1496/ai-agents-pro-vibeathon): the source code, original brief, eval results, demo guide, and architecture decisions.
- [Treating AI Coding Tools Like a Platform Product](/posts/ai-tooling-as-a-platform/): why guardrails have to live in code and be tested at the point where they actually run.
- [Observability as Code](/posts/observability-as-code/): the thinking behind the service review and alert checks.
- [AI Agents.PRO at fwdays Academy](https://fwdays.com/en/event/ai-agentspro-course-2): the course that ended with this vibeathon.
