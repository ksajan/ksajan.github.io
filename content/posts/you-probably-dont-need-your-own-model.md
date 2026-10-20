+++
title = "You Probably Don't Need Your Own Model"
date = 2026-10-19T00:00:00-05:00
draft = false
summary = "Why domain-tuned security models rarely beat a capable general model with the right context, tools, skills and scope, and where training still earns its cost."
tags = ["llm", "security", "fine-tuning", "agents", "evaluation"]
categories = ["engineering"]
authors = ["Sajan Kumar"]
cover = ""
+++

Every few weeks someone announces a model trained specifically for their industry. Finance, legal and healthcare have all had their share, and in security there has been a wave of them lately. Each time, the same question comes up, in conversations and in threads I read: should we be building our own model too?

I have been building AI and data systems for quite a while now, across different sectors, so I think about this a lot from that perspective. My honest answer, for most of what people want these models to do, is no. Not because training is bad, but because the problem it is supposed to solve mostly moved somewhere else.

## What changed

A few years ago, every useful model was its own project. Phishing classifier, entity extractor for threat reports, malicious command line detector. Each one had its own dataset, its own training pipeline, and its own drift problem. If you wanted a model to do something, you trained it.

LLMs changed where specialization lives, and open models have matured enough that this holds for them too, not only for the closed ones. A capable instruction-following model already knows what a CloudTrail event is, can read a PowerShell one-liner, and can map behavior to ATT&CK. What it does not know is your environment and your task. You give it that through:

- **Context**: the actual events, the asset, the user's history, the detection that fired.
- **Tools**: query the data, look up the identity, check what else the host did.
- **Skills**: written procedures for a specific job, the way a good analyst would actually work the case.
- **Scope**: one narrow task with clear inputs and a clear definition of done.

All four of those can change in an afternoon without a GPU. Weights cannot.

This matters more in security than most places. It is a constant chase between attackers and defenders, and the work has layers: detection, triage, threat hunting, response. Each layer has its own knowledge and all of it shifts week to week. Fraud has the same problem, where a pattern changes the moment it gets caught. In domains like these, weights are the wrong place to keep what you know.

## Not all fine-tuning is the same

"We trained our own model" can mean very different things, and they are not equally useful. Here is how I think about the main kinds.

| Kind | What it changes | Where it helps |
| --- | --- | --- |
| Continued pretraining + SFT on domain text | Vocabulary, tone, familiarity with the domain | Rarely worth it now. The base already read the same public corpus. |
| LoRA / adapters | Output style, format, small behaviors | Cheap way to get consistent formatting or schema from a small model |
| RL on tasks with checkable outcomes | Behavior on one narrow, verifiable task | The most defensible kind, but only once the task is already well scoped |
| Distillation into a small model | Size, cost and latency | Running a narrow task at high volume or where big models cannot go |

Most of the industry models I see announced are the first kind: an open model, often Qwen or Kimi, with more domain text poured in. That is the weakest of the four for reasoning. The useful kinds all have one thing in common: they assume you already know exactly what task you want done.

## When training is worth it

I still train models. The question is always what job the model is doing.

Training earns its cost when:

- **Volume and latency rule out an LLM.** Scoring billions of events a day, inline, in milliseconds. A small trained classifier wins and it is not close.
- **The signal is statistical.** Baselines, outliers, beaconing intervals, rare process trees, volume spikes. Gradient boosting, isolation forests and time series models are built for this. An LLM is the wrong tool.
- **You have labeled data nobody else has**, at real scale, for a behavior you cannot describe in a prompt or a playbook.

There are also honest reasons people fine-tune in security that have little to do with reasoning:

- **Refusals.** General models often refuse offensive security work, like writing a detection test or explaining an exploit chain. An unrestricted open model fixes that. This is the one model choice I think really matters: a model liberated enough to do the work it is supposed to do. You get that by picking the right open model, not by training one.
- **Data can't leave.** Some teams need to self-host. Fair, but again that is a deployment decision. A good open model with the right harness covers it.
- **Cost per call at scale.** Real, and the answer is usually distillation of a narrow task, not a general security model.

What is not on any of these lists: "reason about security better than a good general model already does."

## What I've seen in practice

I have been involved in some experiments on a pretty ordinary problem: deciding early whether a security finding is likely benign, so analysts can spend time on the rest. These are early and small, so treat them as observations, not results. But a few things stood out.

**General LLMs worked best as a signal, not as the decider.** On their own, asking a model "is this suspicious?" was not great. Feeding that model's score into a traditional gradient boosted model, next to the features we already had, beat the traditional model alone. The LLM score ended up being one of the most important features in the whole model.

**A security-tuned model did worse.** We tried one of the publicly available security models in the same setup. It came in below the general models. Fine-tuning it further on our own data made it worse, not better.

**Labels mattered more than the model.** The biggest limit was not which model we used. It was that some of the historical labels were simply wrong, cases closed with the wrong resolution. No model fixes bad ground truth.

**More context helped, a little.** Adding outside enrichment like reputation data moved things slightly. Useful, but not a fix on its own.

None of this proves anything broadly. But it lines up with what I keep seeing: the gains come from how the problem is framed and the data around it, not from a model that has read more security text.

## What I build instead

The pattern I keep coming back to splits the work by what each piece is good at. Take an impossible travel alert: a user signs in from two countries an hour apart.

1. **Traditional ML filters first.** A cheap model scores every sign-in against the user's baseline. Most never go further. This is where training belongs, and it keeps LLM cost sane.
2. **Rules catch the known.** VPN egress ranges, known travel, service accounts. Deterministic and boring, on purpose.
3. **A scoped agent works what is left.** One job: decide if this sign-in is a real compromise. It has tools to pull the user's recent activity, the device, MFA events and IP reputation. It follows a skill that describes how a good analyst works this exact case. It returns a verdict with the evidence it used.
4. **A human makes the hard call**, starting from gathered evidence instead of a raw alert.

None of the intelligence here sits in custom weights. It sits in the data being clean and complete, in how tightly the task is scoped, and in how good the skill is. If the agent gets it wrong, I fix the skill or the tools, not a training run. As long as the model is capable and willing to do the work, I can swap it without rebuilding anything.

This is not free either. Skills need upkeep, evals need maintaining, and agents get expensive if you point them at everything. That is exactly why the ML layer in step 1 matters.

## The evaluation gap

The part that bothers me most as an engineer: there is no good way today to compare a domain-tuned model against a well built agentic setup.

Most published evaluations are knowledge quizzes or CTF-style puzzles. They tell you whether a model knows what a technique is. They do not tell you whether it closes real cases correctly. And comparisons are often uneven. The tuned model gets a full harness of tools, retrieval and prompts. The general model gets a bare prompt. Give both the same harness and they are essentially the same system with different weights in the middle, which is the only comparison that actually means anything.

A fair test, in my view:

- Real cases with known outcomes, and labels you have checked.
- Same tools, same data access, same skills for every model.
- Measure what matters on the job: verdict accuracy, missed true positives, time to verdict, cost per case, and how often a human had to redo the work.
- Rerun it whenever either side changes.

I am thinking about putting together a benchmark along these lines: scoped security tasks, a shared harness, and models plugged in on equal terms. Nothing to share yet, but if that is something you would use or want to help shape, I would like to hear from you.

## My take

Training your own model used to be the only way to get a model to do something specific. Now it is one tool among several, and for reasoning in a fast moving domain it is rarely the best one.

If I were starting fresh: train where the math demands it, meaning high volume, low latency and statistical signals. Everywhere else, scope the task tightly, give the agent the right context and skills, fix your labels, and keep all of it current as the domain shifts. The model still matters, just less than it used to. Open models are mature enough now that what you really need is one liberated enough to do the work. After that, the results come from everything around it.
