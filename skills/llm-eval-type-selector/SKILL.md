---
name: llm-eval-type-selector
version: "1.0.0"
description: >
  Use this skill when a developer wants to decide what type of evaluation to build for their AI system.
  Triggers on: "should I use a rule or a judge", "what type of eval should I build", "decide eval type",
  "judge vs programmatic rule", "LLM-as-judge vs rule-based eval", "which evaluation type should I use",
  "how do I evaluate [X]", "what eval should I use for this failure", "is this a rule or a judge",
  "how should I evaluate my AI automatically", "what kind of eval fits this issue".
  Takes one or more failure modes or quality dimensions and returns a concrete type recommendation —
  programmatic rule, LLM-as-judge, or composite — with rationale and a suggested implementation path.
license: Apache-2.0
---

# LLM Eval Type Selector

You help developers decide what type of automated evaluation to build for a given failure mode or quality dimension.

There are three types of automated eval. Each one is good at something the others aren't:

- **Programmatic rules** — deterministic code checks (regex, schema validation, keyword matching, length checks). Fast, cheap, perfectly consistent. Can only check form, not meaning.
- **LLM-as-judge** — a language model scores the output against criteria written in natural language. Understands nuance, tone, relevance. Costs more and can be wrong.
- **Composite** — a combination of both, weighted or run in sequence. Used when you need both structural and semantic coverage.

Picking the wrong type is expensive: a rule can't catch what a judge catches, and a judge is overkill (and slower) where a rule would do. This skill makes that decision explicit before you build anything.

**Where you are:** Step 3 of 7 in the eval workflow. Previous: `llm-issue-discovery` · Next: `llm-judge-creator`

**Before starting:** Check if any context documentation exists — `CLAUDE.md`, `product-marketing-context.md`, or any other context files in the project or workspace. If found, read them first. Use that context to skip questions already answered and only ask for information specific to this task.

---

## Step 1 — Get the failure modes or quality dimensions

Ask for what they want to evaluate. Accept any of:
- An issue report from `llm-issue-discovery`
- A list of failure modes or quality dimensions they've identified
- A single thing they want to start with ("I want to check if the model follows the right format")

If they don't have issues identified yet, tell them:
> "Before deciding on eval type, it helps to know exactly what you're measuring. The recommended path is: annotate your production logs first (`llm-annotation-guide`), then find patterns in those annotations (`llm-issue-discovery`). That gives you a grounded issue list to work from — one backed by human judgment, not just pattern scanning.
>
> If you have a specific failure or quality dimension already in mind, you can describe it here and we'll go from there."

Once you have at least one failure mode or dimension to evaluate, proceed.

---

## Step 2 — Apply the decision criteria

For each failure mode or quality dimension, work through these questions in order.

### Question 1: Can you write this check as a simple conditional?

If you could describe the check to a compiler — as a true/false, a match/no-match, or a count — it's a **programmatic rule**.

Common rule cases:
- **Format compliance** — is the output valid JSON? Does it match the required schema? Are all required fields present?
- **Length and structure** — is the response within a defined word or character count? Does it have the right number of sections?
- **Keyword and entity presence** — does the response include a required piece of information (a title, a price, an order number)?
- **Prohibited content** — does the response contain something it must never contain (an internal ID, a customer email, a forbidden phrase)? Regex-detectable.
- **Classification accuracy** — did the model output one of a fixed set of valid labels?
- **Unfilled placeholders** — are there template variables like `{{Customer Name}}` left in the output?

If yes to any of these → **programmatic rule**.

### Question 2: Does the check require understanding language to answer?

If you'd have to explain what you mean — using words like "appropriate," "relevant," "complete," "helpful," "faithful," or "empathetic" — it's an **LLM-as-judge** eval.

Common judge cases:
- **Semantic correctness** — did the response actually answer what was asked, not just contain the right keywords?
- **Tone and style** — is the response warm, professional, empathetic, or on-brand? Is it consistent with the persona defined in the system prompt?
- **Completeness** — did the response address all parts of a multi-part request, or did it skip something?
- **Faithfulness** — does the response stay grounded in the context or data it was given, or does it introduce information that wasn't there?
- **Instruction following** — did the model follow the logic or flow defined in the system prompt? Did it use the right tools in the right order?
- **Relevance** — did the response address the user's actual need, or did it answer a different question?

If yes to any of these → **LLM-as-judge**.

### Question 3: Do you need both?

Some failure modes have a structural component and a semantic component. Both matter independently.

Example: a bookseller chatbot responding to a recommendation request. You might want to check:
1. Does the response include a book title and price? (Rule — keyword presence)
2. Is the recommended book actually relevant to what the customer asked for? (Judge — semantic relevance)

These are separate checks. Run the rule first — if the output is structurally broken, there's no point running a judge on it. Then layer the judge on top.

If both apply → **composite eval** (rule first, then judge).

### The core heuristic

> If you can explain the check to a regex, use a rule.
> If you'd have to explain it to a person, use a judge.

---

## Step 3 — Output the recommendation

For each failure mode, output a self-contained block using the exact format below — one block per issue. The goal is that a developer reads it and immediately knows what to build and how to build it.

---

### [Issue name]

**Eval type:** [Programmatic rule / LLM-as-judge / Composite]

**What to build:** [One sentence naming the concrete artifact — e.g., "A regex check on the output", "An LLM judge prompt", "A regex check followed by an LLM judge prompt for flagged outputs"]

**[For programmatic rules — use this section:]**

**Check:** [Plain-language description of the condition — e.g., "Does the response contain an internal order ID (format: ORD-XXXXXX)? If yes → fail."]

**Implementation:** [Concrete code or pseudocode — e.g., `re.search(r'ORD-\d{6}', output)` → fail if match found]

**[For LLM-as-judge — use this section:]**

**Dimension:** [Short name for what the judge measures — e.g., "Guardrail compliance", "Recommendation relevance"]

**Judge brief:** [2–3 sentences that could be dropped directly into a judge prompt. What does pass look like? What does fail look like? Ground it in the actual failure, not generic language.]

**Scoring:** [Binary (pass/fail) — use when the guardrail either held or it didn't. 1–5 scale — use when there are meaningful degrees of quality.]

**Next step for this issue:** Take the judge brief above to `llm-judge-creator` to build the full judge prompt.

**[For composite — use both sections above, then add:]**

**Run order:** Rule first. If the rule finds no match, stop — output passes. If the rule flags the output, run the judge on it.

---

Repeat the block for every issue. After all blocks, add:

> **Overall next step:** For any issues marked LLM-as-judge or Composite, take the judge briefs to `llm-judge-creator` to build the full prompts. Once built, use `llm-judge-alignment` to validate them against human-labeled examples before trusting the scores in production.

---

## Principles

**Structural correctness comes before quality assessment.** There's no point asking a judge whether a response was helpful if the response is malformed JSON. Rules run first — they're the filter that lets expensive evaluation methods focus on the harder questions.

**Precision beats coverage.** One well-targeted eval that catches a real failure mode is worth more than five generic ones that pass everything. Resist the urge to evaluate every possible dimension.

**Don't use a judge where a rule will do.** Rules are faster, cheaper, and perfectly consistent. Every judge call costs money and introduces a potential error. If you can encode the check as code, do it.

**Rules can't measure meaning or behaviour.** A rule that checks for the word "empathetic" in a response is not measuring empathy. If you find yourself writing rules as proxies for quality, you probably need a judge.

**Separate dimensions, separate evals.** Don't build one judge that tries to measure tone, factual accuracy, and completeness simultaneously — it'll be unreliable on all three. One dimension per eval, whether rule or judge.

**Start with the highest-severity failures.** You don't need to evaluate everything at once. Pick the failure mode that causes the most user-facing harm and build that eval first.
