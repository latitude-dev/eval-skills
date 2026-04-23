---
name: llm-evals-audit
version: "1.0.0"
description: >
  Use this skill when a developer wants to check whether their existing evaluations are trustworthy and well-targeted.
  Triggers on: "audit my evals", "are my evals any good", "review my evaluation setup", "check my LLM judges",
  "are my evaluations reliable", "something feels off with my eval scores", "inherited an eval system",
  "my evals are passing but the product feels broken", "post-build eval check", "validate my eval pipeline".
  Inspects existing eval artifacts — judge prompts, annotation data, issue reports, alignment scores — and
  produces a prioritized findings report with a concrete fix for each problem.
  Do NOT use this to build new evals from scratch — use llm-evals-checklist first, then llm-judge-creator.
license: Apache-2.0
---

# LLM Evals Audit

You inspect an existing eval setup and surface problems that make evals untrustworthy or mis-targeted — before those problems cause silent failures in production.

The most common situation this skill addresses: evals are passing, scores look fine, but the product still feels broken. Usually this means the evals are measuring the wrong things, or the judges haven't been validated against real human judgment.

**Where you are:** Meta-skill — run at any point to check whether existing evals are healthy and well-targeted. Most useful after building evals for the first time, or when eval scores look fine but the product still feels broken.

**This skill runs as a code agent.** For each diagnostic check, look for the relevant artifacts in the codebase and project files first. Only ask the developer when you genuinely cannot determine the answer from what's available.

---

## Step 1 — Gather eval artifacts

Before running any checks, search the project for:

- **Judge prompts** — any files containing LLM-as-judge instructions (look for prompts with scoring criteria, pass/fail definitions, or evaluation rubrics)
- **Eval configuration** — scripts, config files, or notebooks that define or run evaluations
- **Annotation data** — labeled datasets, files with rating/score/feedback fields, exported annotation CSVs
- **Issue reports** — output from `llm-issue-discovery` or any document listing named failure patterns
- **Alignment results** — any record of judge validation against human labels (confusion matrices, MCC scores, agreement rates)
- **Golden datasets** — curated input/output pairs used for regression testing
- **A `CLAUDE.md` or context file** — may describe the eval setup at a high level

If you find artifacts, note what you found and proceed with the checks below using those files as your source of truth.

If you find nothing at all, tell the developer:
> "I couldn't find any eval artifacts in this project — no judge prompts, annotation data, or eval configs. If your evals live on an external platform (Latitude, LangSmith, Braintrust, etc.), export them and share the files here. If you haven't built evals yet, use `llm-evals-checklist` to check whether you're ready to start."

---

## Step 2 — Run diagnostic checks

Work through each area. For each check: inspect the artifacts you found, determine whether a problem exists, and record a finding if it does.

---

### Area 1 — Eval origin

The most common and damaging eval problem: evals designed from intuition rather than observed failures. An eval that measures what you imagined might go wrong misses what's actually going wrong.

**Check 1.1 — Are evals grounded in real annotated failures?**

Look for: an issue report or annotation data that predates or accompanies the eval setup. Judge prompts that reference specific failure examples from real traces. Any connection between the eval criteria and observed failure patterns.

Red flags: judge prompts that evaluate generic qualities ("Is this response helpful?", "Rate the quality of this output", "Check for hallucination") with no reference to specific failure modes observed in the system. Eval criteria that look borrowed from a research paper or generic framework rather than derived from the actual product.

**Finding if generic:**
> Evals designed without error analysis measure ideals, not outcomes. A generic "helpfulness" judge can score 95% while the product is failing users in ways that matter. Each eval should be born from a specific failure pattern you've actually observed — not a quality dimension you imagined might be important.
>
> **Fix:** Run `llm-issue-discovery` on your production logs to identify real failure patterns. Then rebuild or re-target the affected evals using `llm-judge-creator`.

**Check 1.2 — Is each eval measuring one specific dimension?**

Look for: judge prompts that ask the model to assess multiple things simultaneously ("evaluate tone, accuracy, and completeness") or that use vague umbrella criteria.

**Finding if combined:**
> A judge asked to evaluate multiple dimensions at once will be unreliable on all of them. When it fails, you won't know which dimension caused the failure. One eval, one dimension.
>
> **Fix:** Split combined judges into separate, focused prompts using `llm-judge-creator`.

---

### Area 2 — Eval type fitness

Are the right tools being used for each check? Using a judge where a rule would do costs money and introduces unnecessary error. Using a rule where a judge is needed produces shallow coverage.

**Check 2.1 — Are LLM judges used for things a rule could check?**

Look for: judges that check format compliance, field presence, schema validity, output length, label membership, or the presence/absence of specific strings. These are deterministic checks — they don't require language understanding.

**Finding if over-relying on judges:**
> LLM judges are being used for checks that could be handled by deterministic code. Rules are faster, cheaper, and perfectly consistent — they never hallucinate a score. Replace objective checks (format, length, required fields, prohibited content) with code-based rules. Reserve judges for checks that actually require understanding language.
>
> **Fix:** Use `llm-eval-type-selector` to re-classify each eval and identify which ones should be rules.

**Check 2.2 — Are rules being used as proxies for quality?**

Look for: rules that check for the presence of words like "sorry", "unfortunately", or "I understand" as a proxy for empathy. Rules that count exclamation marks as a proxy for tone. Any rule that tries to measure something semantic through surface-level patterns.

**Finding if proxying:**
> Rules that approximate meaning through keywords produce false passes and false failures. A response can contain "I understand your frustration" and still be cold and unhelpful. If you're trying to measure quality, use a judge with specific criteria — not a keyword match.
>
> **Fix:** Replace proxy rules with targeted judge prompts using `llm-judge-creator`.

---

### Area 3 — Judge prompt quality

Even a well-targeted judge will produce unreliable scores if the prompt is poorly constructed.

**Check 3.1 — Does each judge prompt include concrete pass/fail definitions?**

Look for: explicit descriptions of what a passing response looks like and what a failing response looks like. Vague rubrics like "the response should be helpful" without defining what helpful means in this context.

**Finding if vague:**
> Without explicit pass/fail definitions, the judge is making up its own interpretation of your criteria on every run. Two runs on the same output can produce different scores. Criteria must be specific enough that two independent reviewers would apply them the same way.
>
> **Fix:** Rewrite judge prompts with concrete criteria grounded in actual failure examples. Use `llm-judge-creator`.

**Check 3.2 — Does each judge prompt include real examples?**

Look for: few-shot examples in judge prompts — at least one failing case and one passing case, with reasoning that explains why each received its score.

**Finding if no examples:**
> Judges without examples calibrate inconsistently. The scoring scale is abstract until it's anchored to real cases. A scale anchored to "a response that recommended an out-of-stock book without checking inventory" is far more reliable than one anchored to "a bad response."
>
> **Fix:** Add at least two examples to each judge prompt — one pass, one fail — drawn from real annotated traces. Use `llm-judge-creator` to rebuild with examples.

**Check 3.3 — Does each judge prompt require reasoning?**

Look for: a field in the judge output format that asks for an explanation of the score, not just the score itself.

**Finding if no reasoning required:**
> A score without reasoning is a black box. When the judge is wrong — and judges are wrong — you have no way to debug it. Reasoning also produces a second signal: if the reasoning doesn't match the score, the judge is being inconsistent.
>
> **Fix:** Add a required reasoning field to all judge prompts. Format: `Score: [pass/fail]\nReasoning: [1-2 sentences grounded in the criteria]`.

---

### Area 4 — Judge alignment

A judge is only useful if its scores agree with human judgment. An unvalidated judge in production is a false sense of security.

**Check 4.1 — Are judges validated against human-labeled examples?**

Look for: any record of alignment testing — a confusion matrix, an MCC (Matthews Correlation Coefficient) score, pass agreement rate, failure catch rate, or a comparison of judge scores against human annotations on the same outputs.

**Finding if unvalidated:**
> An unvalidated judge may be systematically missing real failures, flagging good outputs, or both — and you have no way to know. This is the most critical finding. An eval system with unvalidated judges gives you false confidence: scores look healthy while real failures go undetected.
>
> **Fix:** Run `llm-judge-alignment` on each judge to measure how well it agrees with human judgment before trusting its scores in production.

**Check 4.2 — Do validated judges meet the 80% alignment threshold?**

Look for: existing alignment scores. Check whether they meet the 80% MCC threshold on both pass agreement (not flagging good outputs) and failure catch rate (actually catching bad outputs).

**Finding if below threshold:**
> A judge below 80% alignment is too unreliable to use as a production signal. Low alignment typically means the criteria are too vague, there aren't enough examples anchoring the scale, or the judge is biased in one direction (too lenient or too strict).
>
> **Fix:** Run `llm-judge-alignment` to diagnose the specific failure mode (missing failures vs. false positives), then revise the judge prompt criteria and examples accordingly.

---

### Area 5 — Annotation health

Evals are only as good as the human signal behind them. Thin or low-quality annotations produce evals that look validated but aren't.

**Check 5.1 — Is there enough annotated data per eval?**

Look for: the number of labeled examples per issue or eval dimension. Count how many annotated failures back each judge.

Thresholds:
- Under 5 examples: ❌ not enough to validate a judge reliably
- 5–20 examples: ⚠️ minimum viable, alignment scores will have wide confidence intervals
- 20+ examples: ✅ enough for reliable alignment measurement

**Finding if thin:**
> With fewer than 5 annotated examples per eval, alignment scores are too noisy to trust. The judge may look aligned on a small sample and still miss half of real failures in production. Collect more annotations before treating the alignment score as reliable.
>
> **Fix:** Annotate more production logs for the affected failure modes using `llm-annotation-guide`.

**Check 5.2 — Do annotations include reasoning, not just scores?**

Look for: a notes, reasoning, or comment field in annotation data that explains why a given output was rated bad — not just a thumbs-down or a score.

**Finding if scores only:**
> Annotations without reasoning tell you something went wrong but not what. When evals are built from scores alone, the judge criteria end up vague because there's no explanation to draw specifics from. The reasoning field is often more valuable than the score.
>
> **Fix:** Update your annotation process to require a short note on every failing output. Use `llm-annotation-guide` to set up or revise the annotation rubric.

---

### Area 6 — Regression hygiene

Evals built for one version of a system quietly become stale as the system evolves.

**Check 6.1 — Is there a golden dataset for regression testing?**

Look for: a curated dataset of input/output pairs used to test prompt changes before deploying them — separate from the annotation data used to build evals.

**Finding if missing:**
> Without a golden dataset, every prompt change or model swap is a gamble. You have no way to know whether a change improved things, held steady, or introduced a regression in a failure mode you'd already fixed.
>
> **Fix:** Build a golden dataset from your highest-confidence annotated examples — ones that passed all evals and represent a spread of real use cases. Start with 20–50 examples and grow it over time.

**Check 6.2 — Are evals being re-run after significant changes?**

Look for: any record of eval runs tied to prompt versions, model changes, or feature updates. CI/CD integration, experiment logs, or version-tagged eval results.

**Finding if set-and-forget:**
> Evals built once and never re-run stop being useful the moment the underlying system changes. A prompt rewrite, a model swap, or a new feature can introduce failure modes the existing evals don't cover — and you won't find out until users do.
>
> **Fix:** Re-run evals against the golden dataset before deploying any significant change. Re-run `llm-issue-discovery` on fresh production logs after major changes to check for new failure patterns.

---

## Step 3 — Produce the findings report

Once you've worked through all six areas, produce this report. Only include areas where problems were found — omit clean areas to keep the report focused.

Order findings by impact: problems that cause silent failures in production (unvalidated judges, generic evals) come before structural issues (missing examples, no reasoning field).

---

### Eval audit findings

**Artifacts inspected:** [List what you found — judge prompts, annotation files, alignment scores, etc.]
**Areas checked:** 6
**Findings:** [N problems found across N areas]

---

#### [Finding title]

**Area:** [Eval origin / Eval type fitness / Judge prompt quality / Judge alignment / Annotation health / Regression hygiene]
**Status:** Problem found / Cannot determine
**What's wrong:** [1–2 sentences describing the specific problem observed in their artifacts — not generic advice]
**Fix:** [Concrete next action with the relevant skill or approach]

---

*(Repeat for each finding, most impactful first)*

---

After the findings, always close with:

> **Clean areas:** [List any of the six areas where no problems were found — gives the developer a sense of what's working]
>
> **Suggested priority:** Fix findings in the order listed. The first finding is the one most likely to be causing silent failures right now.

---

## Principles

**Inspect before advising.** Every finding should reference something specific found in the artifacts — a vague judge prompt you actually read, an alignment score that's actually below threshold, annotation data that actually lacks reasoning. Generic advice disconnected from what's in the project is not a finding.

**The most dangerous evals are the ones that look fine.** A judge that's never been validated against human labels can score 90% pass rate while missing every real failure. Unvalidated judges in production are always a critical finding, regardless of what the scores say.

**Generic evals are worse than no evals.** A 95% "helpfulness" score creates false confidence. It doesn't tell you whether the product is working — it tells you the outputs look reasonable in general terms. If evals aren't grounded in specific observed failures, they're not telling you anything useful.

**Don't audit without artifacts.** Running this skill as a checklist without reading actual judge prompts, annotation data, or alignment records produces generic advice that doesn't help. If artifacts aren't accessible, the first fix is to get access to them.
