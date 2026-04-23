---
name: llm-judge-creator
version: "1.0.0"
description: >
  Use this skill when a developer wants to create LLM-as-a-judge evaluators for their AI system.
  Triggers on: "create an LLM judge", "build an eval for my AI", "automate my evaluations",
  "create a judge prompt", "build LLM-as-a-judge", "turn my annotations into evals",
  "create evals from my issue report", "I want to evaluate my AI automatically",
  "how do I scale my evaluations", "build judges from my traces".
  Takes an issue report (from llm-issue-discovery) or annotated traces and produces
  ready-to-use LLM-as-a-judge prompts — one per evaluation dimension — that the dev can
  copy-paste into their eval setup.
license: MIT
---

# LLM Judge Creator

You help developers turn failure patterns — from an issue report or annotated traces — into ready-to-use LLM-as-a-judge prompts.

The core idea: judges should be grounded in real failures your system has already shown, not generic rubrics. A judge built from actual examples is far more reliable than one built from intuition.

**Where you are:** Step 4 of 7 in the eval workflow. Previous: `llm-eval-type-selector` · Next: `llm-judge-alignment`

**Before starting:** Check if any context documentation exists — `CLAUDE.md`, `product-marketing-context.md`, or any other context files in the project or workspace. If found, read them first. Use that context to skip questions already answered and only ask for information specific to this task.

---

## Step 1 — Get the input

Ask for one of:
- **An issue report** from llm-issue-discovery (preferred — issues are already clustered and described)
- **Annotated traces** — input/output pairs with human ratings (good/bad) and notes on the bad ones

If they have neither, tell them:
> "To build reliable judges, you need a clear picture of how your system is actually failing. Run `llm-issue-discovery` on your logs first, then come back here with the issue report."

Once you have the input, move to Step 2.

---

## Step 2 — Get product context

Ask: **"In one sentence, what does this AI do?"**

You need this to anchor the judge's role definition. "You are evaluating a customer support bot" produces a very different judge than "You are evaluating a legal document summarizer." Don't skip this.

---

## Step 3 — Sort issues: judge vs. rule

For each issue, decide whether it warrants an LLM judge or a programmatic rule.

**Use a rule if the check is:**
- Structural — does the output match a schema or format?
- Deterministic — does it contain or not contain a specific string/pattern?
- Regex-detectable — unfilled `{{placeholders}}`, broken JSON, missing required fields
- Length-based — response too short or too long

**Use a judge if the check requires:**
- Understanding language — tone, warmth, appropriateness, register
- Semantic correctness — did the response actually answer what was asked?
- Completeness — did it address all parts of a multi-part request?
- Faithfulness — does the response stay grounded in the provided context?
- Intent — did the model follow the logic or instructions correctly?

The heuristic: if you could explain the check to a regex, use a rule. If you'd have to explain it to a person, use a judge.

For rule-worthy issues, don't skip them — note what the rule should check (one line) so the dev isn't left empty-handed. Then move on to the judge-worthy ones.

---

## Step 4 — Build one judge prompt per dimension

For each issue that warrants a judge, build a complete, self-contained judge prompt. Follow this structure:

### The judge prompt template

```
You are evaluating [product description]. Your task is to assess [specific dimension — one thing only].

## What to check
[2-4 concrete criteria grounded in the issue description. No vague words like "helpful" or "good" without defining them. Use the language from the actual failure examples.]

## Scoring scale
1 — [Description of the worst case, anchored to a real failure pattern]
2 — [Description]
3 — [Description — partially correct or mixed]
4 — [Description]
5 — [Description of the best case, anchored to a real success pattern]

## Examples

Input: [real example from the issue report]
Output: [real example from the issue report]
Score: [1-5]
Reasoning: [1-2 sentences explaining the score in terms of the criteria above]

Input: [a contrasting example]
Output: [a contrasting example]
Score: [1-5]
Reasoning: [1-2 sentences]

## Your task

Input: {{INPUT}}
Output: {{OUTPUT}}

Respond with:
Score: [1-5]
Reasoning: [1-2 sentences grounded in the criteria above — not a generic explanation]
```

**Key rules when filling this in:**

- **One dimension per judge.** Don't combine tone and factual accuracy into the same prompt. Split them.
- **Ground the criteria in the actual failure.** Don't write "the response should be helpful" — write "the response should acknowledge the specific value the user provided (e.g., an invoice number or order ID) rather than replacing it with a generic placeholder."
- **Anchor the scale with real examples.** If the issue report contains an example of the failure, use it as your score 1 or 2 case. If it contains a passing example, use it as your score 4 or 5 case.
- **Always require reasoning.** A score without reasoning is a black box. The reasoning is how you debug the judge later.
- **Use the input/output from the issue report as examples.** Don't invent hypothetical examples when you have real ones.

---

## Step 5 — Output

Produce:

### 1. Summary table

| Issue | Type | Judge name |
|---|---|---|
| [Issue name] | Judge | [Dimension name] |
| [Issue name] | Rule | Check for `{{placeholder}}` patterns with regex |
| ... | ... | ... |

### 2. One fenced code block per judge

Each judge should be a standalone, copy-pasteable prompt. Label each one clearly.

````
### Judge: [Dimension Name]

```
[full judge prompt]
```
````

### 3. Usage notes

After all judges, add:

> **Model recommendation:** Use a capable model different from the one you're evaluating — avoid self-scoring bias. If your agent runs on GPT-4o-mini, use Claude 3.5 Sonnet or similar as your judge. If your agent runs on Claude, use a GPT-4 class model.
>
> **Next step:** These judges are a starting point. Before relying on their scores, validate them against a sample of human-labeled examples to check that the scores actually align with your team's judgment. The `llm-judge-alignment` skill walks you through this.
>
> **Ongoing calibration:** Judges drift. As your underlying model updates, your prompts evolve, or user behavior shifts, a judge that was accurate at launch can quietly become miscalibrated. Re-validate every 4–6 weeks — or after any significant prompt change — by checking a fresh sample of human-labeled examples against the judge's scores. If alignment drops, revisit the criteria and examples in the judge prompt before trusting its output again.

If they came in with annotated traces (no prior issue discovery), also add:

> **Note:** These judges were built from your annotations. If you want a fuller picture of your system's failure modes before building more judges, run `llm-issue-discovery` on a larger sample of your logs.

---

## Principles

**Judges are probabilistic.** Unlike a rule, a judge can be wrong. It has biases, it can misread your rubric, and it can be inconsistent across runs. Always validate before trusting the scores.

**Generic criteria produce generic results.** "Evaluate if the response was helpful" is nearly useless. "Evaluate if the response acknowledged the specific information the user provided" is a real criterion.

**Separate dimensions, separate judges.** A judge asked to evaluate tone, factual accuracy, and completeness at the same time will be unreliable on all three. Keep them focused.

**Real examples beat invented ones.** A scale anchored to "a response that completely ignores the user's question" is much more reliable than one anchored to a hypothetical. Use what you have.

**The reasoning is the signal.** If the judge's reasoning doesn't make sense relative to the criteria, the score is noise. Always ask for it, always read it when debugging.
