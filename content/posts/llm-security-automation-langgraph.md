+++
title = "Cutting security threat analysis from hours to minutes with AI"
date = 2024-10-15T00:00:00-05:00
draft = false
summary = "How we automated most security alert triage with a graph of small agents, how the industry approaches the same problem today, and how agent swarms and feedback loops built on validated cases keep it improving."
tags = ["llm", "security", "langgraph", "rag", "automation", "platform", "agents", "evaluation"]
categories = ["engineering"]
authors = ["Sajan Kumar"]
cover = ""
+++

*Updated October 2026: expanded with how others in the industry now approach agentic triage, and how I think about agent swarms and feedback loops built on validated cases.*

Security analysts were drowning in alerts. Every alert needed someone to pull context, read logs and decide whether it mattered, and most of them did not. That is not unusual. The 2025 SANS Detection and Response Survey found that 73% of security teams name false positives as their top detection challenge ([via Vectra](https://www.vectra.ai/topics/alert-fatigue)). Manual triage also slowed our product teams down, since security checks sat in the path of shipping.

We wanted automation that removed the noise without taking humans out of the decisions that count. The system we built works across about 20TB of security data, and the analysis that used to take hours now takes minutes.

This post covers what we built, what the rest of the industry has converged on since, and where I think the real gains come from now: agents that fan out to gather evidence, and loops that turn every validated case into a better system.

## Make the workflow a graph

The first decision was to stop treating triage as one big prompt. We modeled it as a graph of small agents, each with one job:

1. **Intake** takes the alert and normalizes it.
2. **Retrieval and enrichment** pulls the evidence: related events, the asset, history.
3. **Reasoning** weighs that evidence and reaches a conclusion.
4. **Suggested actions** turns the conclusion into something an analyst can act on.

LangGraph made the state between those steps explicit. When something went wrong, we could see which step produced the bad output and what it was given, instead of guessing at a single opaque call.

## Most of the gains came from retrieval

The model was not the bottleneck. The context was. We combined hybrid search with custom rerankers so the reasoning step only saw the evidence that mattered. Dropping irrelevant context made the answers better and cut token use at the same time.

## Serving

We served models on vLLM with tensor parallelism, which got us sub-second latency with predictable cost. Some of those models were fine-tuned, but not to make them better at security reasoning. They were small models tuned for consistent output formats and for running cheap, high-volume steps. The reasoning came from a capable general model and from the evidence we put in front of it. I have since written more about why I think that split matters in [You Probably Don't Need Your Own Model](/blog/2026/10/you-probably-dont-need-your-own-model/).

## Guarding against regressions

Prompts, retrieval and models all change, and any of them can quietly make the system worse. We added evaluation hooks and red-team prompts to CI so a regression would show up before it reached analysts.

## What it changed

About 60-70% of threat triage is now automated. Feature teams ship around 40% faster because security checks are programmatic and repeatable instead of waiting on a person. Customers get a consistent summary along with the raw evidence behind it, so they can check the work rather than take it on trust.

## How the industry does this now

When we started, agentic triage was mostly an experiment. Now the big platforms ship it, and a few teams have written openly about what worked. They have converged on a surprisingly similar shape.

**Google** ships a [Triage and Investigation Agent](https://docs.cloud.google.com/chronicle/docs/secops/triage-agent) inside Google Security Operations. It runs searches to gather more context, enriches indicators with threat intelligence, analyzes command lines and reconstructs process trees. Then it returns a true or false positive verdict with an explanation that the analyst reviews inside the case.

**Microsoft's** [Phishing Triage Agent](https://learn.microsoft.com/en-us/defender-xdr/phishing-triage-agent) classifies user-reported emails and closes the false positives on its own. True positives stay open for an analyst. The interesting part is the feedback design. When an analyst changes a verdict, they explain why in plain language. The agent turns that explanation into a "lesson," shows it to the analyst for review, checks it against earlier lessons for conflicts, and only then stores it in memory. Every lesson can be audited and rejected later.

**Elastic** has written the most detailed account I have seen of running this in their own SOC. In [one post](https://www.elastic.co/security-labs/alert-triage-agentic-soc-elastic-workflows) they describe cutting triage from 30 minutes to under 3. Deterministic ES|QL queries close known false-positive patterns before any LLM is called. Specialized agents for endpoints, cloud and identity gather facts but do not give verdicts. A final review agent, which cannot run new queries, reads the case and decides. In [another post](https://www.elastic.co/security-labs/alert-triage-agentic-soc-self-correcting-agents) they describe taking verdict accuracy from 60% to 92% without changing the model. The change was entirely in context: the detection rule's investigation guide, user risk data, and the analyst closure reasons from the last 30 days of cases on the same rule.

**Research** is finding the same thing from the other direction. A recent paper, [From Investigation Failures to Reliable SOC Agents](https://arxiv.org/abs/2610.10608v1), replayed 1,247 alerts from a multi-stage attack through a live SIEM. Every existing approach it tested missed at least 40.4% of attack-related alerts. One of its most useful findings is that agents were more likely to dismiss a real attack when their searches came back empty. Their design separates the agent that proposes a verdict from a Judge that checks it against the evidence and can send it back for another round. That cut the false-negative rate to 3.1%, while escalating 18.4% of alerts to analysts.

Put these side by side and the pattern is clear:

- **Do the cheap, deterministic work first.** Anything a query or a rule can settle should not cost tokens.
- **Gathering evidence and deciding are separate jobs.** The agents that collect facts should not be the ones that rule on them.
- **Context beats model choice.** Investigation guides, user context and past cases moved accuracy far more than swapping models did.
- **Humans stay on the hard calls**, and their decisions feed back into the system.

That matches what we found with retrieval, and it is where I would push any team building this today.

## Swarms: fan out for evidence, not for verdicts

"Agent swarm" gets used loosely. The useful version is what Anthropic calls [orchestrator-workers](https://www.anthropic.com/engineering/building-effective-agents): a central model breaks the task down, hands the pieces to worker agents, and puts their results together.

Triage fits this well, because evidence gathering splits cleanly. For a suspicious sign-in, one worker checks the identity provider, one checks the endpoint, one checks cloud activity, and one pulls earlier cases on the same rule. They do not need each other's context, so they can run in parallel. Each one works in its own clean context window and returns a short, structured summary instead of raw logs.

Anthropic's write-up of its [multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) shows both why this works and what it costs. Their multi-agent setup beat a single agent by 90.2% on an internal research eval. It also used about 15 times more tokens than a chat interaction. They found multi-agent designs a poor fit when every agent needs the same shared context. A verdict needs exactly that kind of shared context.

So the shape I recommend:

1. **A deterministic orchestrator.** Routing by alert type, data source and OS does not need a model. Elastic made the same choice and runs orchestration as a workflow, because it is fast, predictable and costs no tokens.
2. **Narrow fact-finding workers in parallel.** Each has a small set of tools, a fixed procedure and a structured output. No worker gives a verdict.
3. **One decider.** It reads the gathered evidence and proposes a verdict with a confidence score and the evidence it relied on.
4. **An independent challenger.** A separate agent, with its own context, tries to argue the opposite case against the same evidence. If it finds a gap, the case goes back for more evidence instead of being closed.

The challenger matters more than it looks. The decider's failure is quiet: a plausible "benign" written over missing evidence. A second agent whose only job is to find what is missing catches that far more reliably than asking the first agent to double-check itself.

## Loops: learn only from validated cases

A triage system that does not learn is a snapshot. The question is what it should learn from, and the answer I keep coming back to is: **only from validated cases.** A case is validated when a human has confirmed the outcome. An agent's own verdict is not ground truth, and feeding it back in just reinforces whatever it already gets wrong.

There are three loops, at three speeds.

**The inner loop runs within one case.** The decider proposes, the challenger tests, and a missing piece of evidence triggers another round of gathering. This is Anthropic's [evaluator-optimizer pattern](https://www.anthropic.com/engineering/building-effective-agents) applied to investigation. It needs a hard limit on rounds, and one rule above all: **an empty search result is not evidence of benign activity.** "I found nothing" has to mean "I could not confirm," and that should escalate, not close.

**The case loop runs as analysts close cases.** Every closure records the agent's verdict, the analyst's verdict and the analyst's reason. When they disagree, that disagreement is the most valuable data the system produces. Two designs work well:

- **Retrieve similar past cases** at triage time, the way Elastic's agents read the last 30 days of closures on the same rule. The agent sees "this rule fired 50 times and 48 were benign, and here is why," including the times an earlier AI verdict was wrong.
- **Turn corrections into reviewed lessons**, the way Microsoft does. The analyst's reason becomes a written rule. A human approves it, it is checked against earlier lessons for conflicts, and it can be audited or removed later.

**The system loop runs weekly.** Validated cases become the evaluation set. Every change to a prompt, a tool, a retrieval step or a model runs against that set before it ships. Two numbers matter most: how often a true positive was missed, and how often a human had to redo the work. Accuracy on its own hides the misses. The system loop also feeds the layers below the agents:

- A benign pattern that keeps getting confirmed becomes a **deterministic rule or query.** It is cheaper, faster and auditable, and it never needs an LLM again.
- A noisy detection becomes a **tuning request** to whoever owns that rule.
- A recurring reasoning mistake becomes a fix to the **investigation guide or the agent's procedure.**

That last point is what makes the system improve over time. Each validated case pushes knowledge down to the cheapest layer that can hold it: rules first, then retrieval, then the agent's procedure. The expensive reasoning gets saved for what is actually new.

There is one catch. Analyst labels are not always right. In separate experiments, I have seen historical cases closed with the wrong resolution become the biggest limit on accuracy. The loop needs a way to challenge old labels too: review cases where the agent and the analyst keep disagreeing, and sample closed cases for a second look.

## Earning autonomy

None of this means letting agents close everything on day one. Autonomy should be earned per category and backed by numbers:

1. **Shadow mode.** The agent runs on every alert, but only humans decide. Compare the two on validated cases.
2. **Assist.** The verdict and evidence appear in the case. Analysts confirm with one click or correct it, and every correction feeds the case loop.
3. **Auto-close, narrowly.** Only alert categories where measured accuracy on validated cases has held up long enough, and only high-confidence false positives. Every automated closure is tagged, explained and easy to reopen.

True positives always go to a human. That is where the risk sits.

## What to watch out for

- **Attacker-controlled input.** Email bodies, command lines, file names and user agents are written by the attacker, and the agent reads all of them. Treat them as data, never as instructions. Keep tool permissions narrow so a manipulated agent cannot do much damage.
- **Missing evidence read as innocence.** Covered above, but it is the failure mode most worth testing for.
- **Cost growth.** Fan-out multiplies tokens. Deterministic filtering and narrow workers keep it in check. Elastic reported its narrow triage agent used about 50k tokens per run, against 500k or more for a general-purpose agent.
- **Lesson drift.** Accumulated feedback can contradict itself or go stale as the environment changes. Lessons need owners, conflict checks and an expiry review.
- **Summaries nobody reads.** Elastic found its analysts had stopped reading AI summaries that were wrong 40% of the time. A summary has to earn trust before anyone will use it.

## What I would tell someone starting this

- **Keep agents small and observable.** Most of our wins came from better retrieval and ranking, not bigger models, and the industry has since found the same thing.
- **Do deterministic work deterministically.** Queries and rules before agents, and an orchestrator that is code, not a model.
- **Fan out to gather facts, decide in one place, and challenge the decision.**
- **Build evals and playbooks early,** from validated cases, so on-call engineers can work out why something failed and catch regressions before analysts do.
- **Close the loop, carefully.** Learn from validated cases, review what gets learned, and push recurring knowledge down into rules.
- **Make the handoff to humans obvious.** Every automated decision links back to its source evidence, so an analyst can override it safely. Automation that people cannot check does not get trusted, and automation that is not trusted does not get used.
