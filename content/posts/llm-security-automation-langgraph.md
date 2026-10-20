+++
title = "Cutting security threat analysis from hours to minutes with AI"
date = 2024-10-15T00:00:00-05:00
draft = false
summary = "Multi-agent systems that process 20TB of security data, trim noise, and push actionable findings fast."
tags = ["llm", "security", "langgraph", "rag", "automation", "platform"]
categories = ["engineering"]
authors = ["Sajan Kumar"]
cover = ""
+++

Security analysts were drowning in alerts. Every alert needed someone to pull context, read logs and decide whether it mattered, and most of them did not. Manual triage also slowed the product teams down, since security checks sat in the path of shipping.

We wanted automation that removed the noise without taking humans out of the decisions that count. The system we built works across about 20TB of security data, and the analysis that used to take hours now takes minutes.

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

## What I would tell someone starting this

- **Keep agents small and observable.** Most of our wins came from better retrieval and ranking, not bigger models.
- **Build evals and playbooks early.** When something fails, the on-call engineer needs to work out why quickly, and that is hard to bolt on later.
- **Make the handoff to humans obvious.** Every automated decision links back to its source evidence, so an analyst can override it safely. Automation that people cannot check does not get trusted, and automation that is not trusted does not get used.
