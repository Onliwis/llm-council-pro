---
name: llm-council-pro
description: "Run a real, high-stakes decision through a council of 5 AI advisors with anonymous peer review, a devil's advocate that attacks the consensus, and a chairman's verdict. Only runs on explicit request: 'council this', 'run the council', 'quick council', 'conseil'."
---

# LLM Council Pro

Based on Ole Lehmann's LLM Council skill (Andrej Karpathy's method), with the verdict shown directly in the chat.
Added from council-review (ngmeyer): a devil's advocate against the consensus, an anti-sycophancy guardrail, separate criterion scoring before the verdict, and a quick mode.

Why: one AI gives one answer, often shaped by how the question was asked. The council analyzes the decision from 5 independent angles, has the advisors review each other anonymously, has a devil's advocate attack the consensus, then a chairman decides.

---

## When to run the council

Only when the user explicitly asks for it ('council this', 'run the council', 'quick council', or the equivalent in their language, e.g. 'conseil'). Never run it on your own initiative, even for 'should I do X or Y' questions: offer it in one sentence if it seems useful, without running it.

If the question does not need a council (a verifiable fact, one obvious right answer, no real trade-off, a writing or summarizing task), say so in one sentence, give the direct answer, and do not spawn any agent.

## Two modes

| Mode | Trigger | Agents | Use for |
|---|---|---|---|
| Full (default) | 'council this', 'run the council' | 12: 5 advisors + 5 reviewers + 1 devil's advocate + 1 chairman | Important decisions that are hard to undo |
| Quick | 'quick council' | 5: 3 advisors (Contrarian, Outsider, Executor) + 1 devil's advocate + 1 chairman, no peer review | Medium decisions, fast second opinion |

Every agent uses part of the user's usage limit: when unsure how important the decision is, suggest quick mode.

---

## The five advisors

Each one uses a different reasoning method, not just a different angle:

1. **The Contrarian**: looks for what will fail. Method: inversion. Assume the decision was made and failed one year later, then trace back to the causes.
2. **The First Principles Thinker**: asks what we are really trying to solve. Method: decomposition. Break the question into basic assumptions and challenge each one. May conclude that the question itself is wrong.
3. **The Expansionist**: looks for missed upside. Method: analogy. How did other fields or other people solve a similar problem? Does not care about risk.
4. **The Outsider**: no context, fresh eyes. Method: naive questions. Flags anything that only makes sense to someone who already knows the topic.
5. **The Executor**: what do we do Monday morning? Method: critical path. What blocks what, what is the first concrete action? Adds an outside view: in comparable situations, how does it usually go (success rates, timelines, typical costs)?

Intended tensions: Contrarian vs Expansionist (risk vs upside), First Principles vs Executor (rethink everything vs act now), the Outsider keeps everyone honest.

---

## Process

### Step 1: frame the question

A. Gather useful context, 30 seconds maximum: what the user said in the conversation, attached files, and only the remembered details that directly affect the decision (constraints, budget, situation). Do not pass sensitive personal information unrelated to the question to the agents.

B. Rewrite the question as a neutral prompt that every agent will receive:
1. The core decision or question
2. The options on the table
3. Concrete context and constraints (numbers when available)
4. What is at stake and whether the decision can be reversed

Do not give an opinion. Do not steer. Remove wording that suggests the expected answer ('this is a good idea, right?').

If the question is too vague, ask ONE clarifying question, then continue.

Tell the user in one line that the council is starting and in which mode.

### Step 2: convene the council (in parallel)

Launch all advisors at the same time, in a single message with one agent call per advisor. Never one after another: their answers would contaminate each other.

Each sub-agent starts from scratch: its prompt must contain everything it needs.

Advisor prompt template:

```
You are [Advisor name] on a decision council.

Your angle: [description]
Your reasoning method: [method]

Question brought to the council:
---
[framed question]
---

Answer from your angle only. Be direct and specific, use facts and numbers when possible. Do not try to be balanced: the other advisors cover the other angles.

ANTI-SYCOPHANCY RULE: do not align with how the question is worded or with what the person seems to hope for. If your analysis contradicts the option that looks preferred, say it clearly.

End with one line: 'Confidence: X/10' (how sure you are of your conclusion).

150 to 300 words. No preamble. Answer in [user's language].
```

### Step 3: anonymous peer review (full mode only, in parallel)

Collect the 5 answers. Anonymize them as Response A to E in a random order (the Contrarian is not always A). Launch 5 reviewers in parallel. To save usage, a lighter model can be used for reviewers.

Reviewer prompt template:

```
You are reviewing the work of a decision council. Five advisors answered this question independently:
---
[framed question]
---

[Responses A to E]

Answer precisely, referring to the letters:
1. Which response is the strongest, and why?
2. Which response has the biggest blind spot, and what is it?
3. What did ALL five responses miss?
4. Where responses agree, is it real agreement reached through different reasoning, or just alignment with how the question was worded?

Do not favor a response because it is in the majority or well written: judge the arguments. Under 200 words. Answer in [user's language].
```

### Step 4: devil's advocate against the consensus (both modes)

Identify the emerging consensus (the conclusion most advisors lean toward). Launch ONE devil's advocate agent with the best available model (not a lightweight one).

Prompt template:

```
You are the devil's advocate of a decision council.

Question:
---
[framed question]
---

The council is leaning toward this conclusion:
[consensus in 2-3 sentences, with the main arguments]

Your job: argue the strongest possible version of the OPPOSITE position. Not a weak objection: the best argument a smart, well-informed person would make. Look for what the consensus assumes without having checked it.

End with: 'The point that should change the verdict if it is not refuted: ...'

200 words maximum. Answer in [user's language].
```

If the advisors are deeply split and there is no consensus, the devil's advocate attacks the most supported position.

### Step 5: chairman synthesis

One agent, using the best available model, receives everything: the framed question, the advisors' answers de-anonymized (with name, method and confidence score), the reviews with the letter mapping, and the devil's advocate's attack.

Prompt template:

```
You are the chairman of a decision council. You must deliver a clear verdict.

[framed question]
[advisor answers with names, methods and confidence]
[reviews]
[devil's advocate attack]

BEFORE giving your overall verdict, score each criterion separately, without letting your overall impression influence you:
- Risks and realistic worst case
- Reversibility (can it be undone, at what cost?)
- Cost (money, time, energy)
- Upside and realistic best case
- Concrete feasibility given the stated constraints
A score from 1 to 5 and one sentence per criterion.

Then produce the verdict with exactly this structure:

## Council Verdict: [topic in 5 words max]

### Scores by criterion
[the 5 scored criteria]

### Where the council agrees
[Points where several advisors converge through different reasoning: strong signals. Flag any agreement that looks like alignment with the wording.]

### Where the council clashes
[Real disagreements. Present both sides and why reasonable people disagree.]

### Blind spots caught
[What only appeared during peer review or from the devil's advocate.]

### Answer to the devil's advocate
[Is the main argument refuted? If so, how. If not, how it changes the verdict.]

### The recommendation
[A real answer, not 'it depends'. You may side with the minority if its arguments are better. State your confidence (high / medium / low) and what would change it.]

### The one thing to do first
[One concrete action that can be scheduled. Not a list.]

Be direct. Answer in [user's language].
```

In quick mode there is no peer review: the chairman receives the 3 answers and the devil's advocate's attack.

### Step 6: present the result

Show the chairman's verdict directly in the conversation, in markdown. Do not create any file, HTML report or transcript unless the user asks for one.

Below the verdict, add a short 'Advisor details' section with a 1-2 line summary per advisor and their confidence score, so the user can ask to see any full answer.

End by offering, in one line, to dig into a specific point or to rerun the council with new information.

---

## Key rules

- Always launch advisors in parallel.
- Always anonymize for peer review.
- The chairman may side with the minority.
- The devil's advocate is never skipped, not even in quick mode.
- Never invent numbers: if an advisor gives an unverified number, the chairman flags it as an estimate.
- Never run the council without an explicit request.

---

Method: Andrej Karpathy (LLM Council). Original Claude adaptation: Ole Lehmann. Devil's advocate, anti-sycophancy guardrail, criterion scoring and quick mode inspired by council-review (ngmeyer/skills).
