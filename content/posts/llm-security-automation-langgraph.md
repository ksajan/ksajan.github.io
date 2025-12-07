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

Security analysts were drowning in alerts, and manual triage slowed feature delivery. We needed automation that reduced noise but still kept humans in control.

## Approach
- Framed the workflow as a graph of agents: intake -> retrieval and enrichment -> reasoning -> suggested actions. LangGraph made the state explicit and debuggable.
- Combined hybrid search with custom rerankers to prioritize evidence before the reasoning step. This dropped irrelevant context and reduced token use.
- Served fine-tuned models on vLLM with tensor parallelism to hit sub-second latencies while keeping costs predictable.
- Added evaluation hooks and red-team prompts in CI to guard against regressions as we iterated.

## Impact
- 60-70% of threat triage is now automated, and analysis time dropped from hours to minutes.
- Feature teams ship ~40% faster because security checks are programmatic and repeatable.
- Customers get consistent summaries plus the raw evidence bundle, so trust stays high even as automation grows.

## Lessons learned
- Keep agents small and observable; most wins came from better retrieval and ranking, not bigger models.
- Invest early in evals and playbooks so on-call engineers can reason about failures quickly.
- Make handoffs obvious: every automated decision links back to source evidence so humans can override safely.
