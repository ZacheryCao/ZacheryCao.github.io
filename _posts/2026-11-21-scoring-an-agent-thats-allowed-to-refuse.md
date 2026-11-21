---
layout: post
title: "Scoring an agent that's allowed to refuse"
date: 2026-11-21
tags: [evaluation, agents, methodology]
summary: "Define a correct refusal, allow several right answers per step, and sample humans from the pool the machine approved so you can estimate what it missed."
---

*Notes on building the evaluation layer for a production agent that acts on live interfaces.*

I work on the evaluation layer for an autonomous agent that takes actions on live web interfaces. Most of my time goes to one question: did the agent actually do the job?

That question turned out to be harder than shipping the agent. What follows is the method I'd hand to someone starting this work today, including the parts I still haven't measured.

## 1. Fix the scoreboard before you fix the agent

An agent acting on a live interface produces three outcomes. It acted correctly. It acted wrongly. Or it declined to act because it couldn't verify what it was looking at.

Logs collapse those into two. Pass and fail. And that collapse quietly punishes the behavior you want most.

Our action selector emits a confidence score, and below a threshold the agent abstains or escalates. A separate auditor model checks the selected action and escalates when it can't verify. Where structural evidence from the interface contradicts the model's read, the structural evidence wins. All of that is deliberate. An agent that guesses wrong takes an action someone has to undo.

So we made the regression gates score a deliberate abstention as a pass. That one decision changed what the team was chasing. Before it, a safety improvement could show up as a quality regression, and the numbers argued for a more reckless agent.

If you take one thing from this piece: decide what a correct refusal looks like, and encode it, before you write a single scoring function.

## 2. Ground truth needs room for several right answers

Early scoring assumed one recorded correct action per step. That's wrong often enough to matter.

Say the agent needs to tick a checkbox. Two different interface elements accomplish it: the box itself, and the text label bound to it. Both achieve the goal for the current stage and intention. A scorer that demands an exact match to one recorded element marks a perfectly good action as a failure.

We treat ground truth as a set of acceptable alternatives per step. The scope matters: these are per-step alternatives. The agent picks one of them, and the step is done.

This is rubric design, and it's where homegrown evaluation rigs quietly go wrong. Write down what counts as success for a step, in plain language, before you decide how to store it.

## 3. Build a test environment you can break on purpose

You can't run ablations against live traffic. So we built a synthetic environment covering a wide range of interface patterns. Layouts were collected from real interfaces. The content, the markup, the styling, and the interactive behavior were generated.

Alongside it sits a controlled test system. It replays recorded inputs, requests fresh model responses constrained to the agent's production action set, and compares the results against reference trajectories.

Fresh responses matter. If you cache and replay model outputs, you've built a very expensive unit test. You'll never see the variance that breaks you in production. Keeping the model live while keeping the inputs fixed is what makes an ablation mean something.

Separately, the agent records the action it took plus the rejected candidate pool, which produces hard negatives with no manual annotation. That's how the training-data loop downstream gets its examples.

## 4. Human judgment is the reference, so sample it deliberately

Here's the design I'd defend hardest, and the one I'd most want to finish.

A separate review agent audits the full dataset. A human reviewer then audits that agent's results. The human sample is drawn at random from the auditor's entire approved pool, which means it includes records the review agent flagged and records it passed.

That sampling population is the whole trick. If you only let humans check what the machine already flagged, you can measure precision and you can never measure what slipped through. Sampling from the input pool, including passes, is what makes an error-detection recall estimate possible against human judgment.

I want to be exact about the state of this: high recall is the design objective. I have not published a recall number, and I'd want to measure it properly before quoting one. The sample size, the disagreement handling, and the metric implementation are all things I'd want tighter than they are.

I'm saying that publicly because I think evaluation writing has a credibility problem. Everyone reports the number that flatters them. The useful thing to report is what your design can and cannot estimate.

## 5. Make the evaluation budget hold by construction

Exhaustive evaluation was unaffordable. The input space varies enormously, and evaluation resources don't scale with it.

So we shipped a sequential probability ratio test classifier, with a closed-loop tuning scheduler and downstream backpressure. Sample counts adapt as evidence arrives. Testing stops early when the answer is already clear.

The property I care about: early stopping removes real compute and real model-invocation spend. The budget holds because of how the system is built. Discipline drifts the first time a launch gets urgent.

Sequential testing is the cost-control lever. If your evaluation bill is a line item someone reviews monthly, that's where I'd start.

## 6. Pair the arms, and gate every change

A supervisor runs divergence detection over sliding-window comparison of interface state. Validation uses same-seed paired-arm A/B, so both arms see the same conditions. Automated regression gates sit in front of changes before they ship, and they preserve the abstention distinction from section 1.

Same-seed pairing is the part people skip. Agent behavior is noisy enough that unpaired comparisons will hand you a confident wrong answer, and you'll ship it.

There's a related trap in latency work. When you replace a model call in the hot path with a cheaper learned selector, the interesting question is how you compared them. Ours used predefined synthetic references plus manually collected reference trajectories on held-out inputs, with the comparison protocol written down as part of the work. I'm not going to quote a performance delta here, because I don't have one I'd stand behind. The protocol is the part I'd defend.

## 7. Attribute the failure, or the number is useless

A quality score you can't act on is decoration. When a run fails, the useful output is which layer caused it.

For our agent the candidate causes are distinct enough to separate. The prompt asked for the wrong thing. The planner sent the work to the wrong specialist. The perception model misread the interface. The selector picked a plausible but wrong action. The structural veto fired and overrode a correct model read. Or the agent abstained correctly and the scoring rig marked it wrong, which is section 1 again.

We built the planner as separate roles: a planner, task specialists, a grounder, and an auditor. That structure is what makes attribution possible. A single monolithic prompt gives you one blob to blame. Separate roles give you a trace you can read, and a trace you can read is a failure you can route to an owner with evidence attached.

That work also produced the cheapest win in the whole system. One specialist's prompt shrank to a fraction of its original length with no capability loss, found by working through what each part of the prompt was doing there. You don't find that by staring at an aggregate score.

I'd add one habit here: keep a coverage map. Write down which behaviors you test, which you test partially, and which you don't measure at all. The third column is the one that matters, and it's the one teams leave blank because it's embarrassing. An honest blank is worth more than a confident average over whatever happened to be easy to test.

## 8. What I haven't solved

One gap, stated plainly, because it's the one that matters for anyone building this.

**Judge calibration.** I have human-referenced recall estimation as described above. I have not built threshold calibration of an automated judge against human labels. I think judge error is the primary measurement risk in any pipeline that scores behavior with a model, and a judge you haven't calibrated is a number generator with good manners. That's the first thing I'd want to build next.

Here's why that belongs in writing. An evaluation number is a claim about measurement quality, and the reader can only check it if you say what the design can estimate. Section 4 is the example. Sampling humans from the approved pool makes a recall estimate possible. It doesn't make the estimate correct, and it says nothing about the raw interactions that never reached that pool. Both limits belong next to the number.

So publish what you can't measure yet. It's the cheapest credibility you'll ever buy, and it's the part of this work that compounds. The rubrics will get rewritten. The sampling plan will get replaced when the input mix changes. What survives is a team that knows which of its numbers to trust.
