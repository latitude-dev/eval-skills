---
name: llm-regression-runner
version: "1.0.0"
description: >
  Use this skill when a developer wants to test a prompt change against a golden dataset and see what broke.
  Triggers on: "run my evals", "test this prompt change", "check for regressions", "did I break anything",
  "run regression tests", "test against golden dataset", "compare prompt versions", "is it safe to deploy",
  "run offline evals", "what changed after my prompt update", "eval before deploying".
  Runs a golden dataset against the current prompt, scores each case with available judges,
  compares results against a saved baseline, and produces a pass/fail report with a clear deploy recommendation.
license: MIT
---

# LLM Regression Runner

You run a developer's golden dataset against their current prompt, score each case with their judges, compare results to the previous baseline, and tell them whether it's safe to deploy.

This is offline evaluation: the gate before a prompt change reaches production. Every time a developer changes a prompt, swaps a model, or adjusts system instructions, this skill answers the question — did quality improve, hold steady, or get worse?

**Where you are:** Step 7 of 7 in the eval workflow. Previous: `llm-golden-dataset-builder` · This is the final step — run before every prompt change.

**This skill runs as a code agent.** Search the codebase and project files for all required artifacts before asking the developer for anything.

---

## Step 1 — Find eval artifacts

Search the project for:

- **Golden dataset** — a file containing curated input/output pairs used for testing. Look for `.json`, `.jsonl`, or `.csv` files with names like `golden`, `dataset`, `test-cases`, `eval-data`, or similar. Each entry should have at minimum an input (or conversation) and an expected outcome or label.
- **Judge prompts or eval scripts** — files containing LLM-as-judge instructions, programmatic rule checks, or evaluation scripts. Look for prompts with scoring criteria or pass/fail definitions, and any code that runs evaluations.
- **Baseline results file** — a saved record of scores from the previous run. Look for files named `baseline`, `eval-results`, `last-run`, or similar (`.json`, `.jsonl`).
- **Prompt or system instruction being tested** — the current version of the prompt under evaluation. Look in config files, environment files, or prompt template files.
- **`CLAUDE.md` or context file** — may describe the eval setup, dataset location, or which judges are in use.

Once you've searched, report what you found before proceeding. If any critical artifact is missing, ask specifically for that one thing — don't ask for everything at once.

**If no golden dataset exists at all**, stop and tell the developer:
> "I couldn't find a golden dataset in this project. A golden dataset is the backbone of offline evaluation — without it, there's nothing to test against. Build one by promoting your highest-confidence annotated examples: pick 20–50 cases that represent real usage, include edge cases and known failure patterns, and define what a correct output looks like for each. Once it's in place, come back here."

**If no judges or eval scripts exist**, tell the developer:
> "I couldn't find any judge prompts or eval scripts. I can still run a basic evaluation using your annotation rubric as the scoring criteria, but the results will be less precise than running validated judges. Do you want to proceed that way, or set up judges first using `llm-judge-creator`?"

---

## Step 2 — Determine run mode

Check whether a baseline results file exists.

**If no baseline exists — this is a first run:**

Tell the developer:
> "No previous baseline found. I'll run the evaluation now and save the results as your baseline. This run won't detect regressions — it establishes the reference point. Run this skill again after your next prompt change to get a comparison."

Proceed to Step 3, then save results as the baseline after the run. Skip Steps 4 and 5 (no comparison to make). Go straight to saving the baseline and prompting the developer to run again after their next change.

**If a baseline exists — this is a comparison run:**

Note the baseline file location and proceed to Step 3. You'll compare new scores against it in Step 4.

---

## Step 3 — Run the evaluation

For each entry in the golden dataset:

1. **Run the current prompt** with the entry's input. If you have access to the AI system being evaluated, run it directly. If not, ask the developer to provide the outputs — either by running them manually or pointing you to a file of already-generated outputs.

2. **Score the output** using the available judges and/or rules:
   - For programmatic rules: run the check against the output. Record pass or fail.
   - For LLM-as-judge: run the judge prompt against the output. Record pass or fail and the reasoning.
   - For composite evals: run the rule first. If the rule fails, record fail and skip the judge. If the rule passes, run the judge.

3. **Record the result** for each case:
   ```json
   {
     "id": "[case identifier]",
     "input": "[the input or a short summary]",
     "score": "pass" | "fail",
     "judge": "[which judge or rule produced this score]",
     "reasoning": "[judge's reasoning if available]"
   }
   ```

4. Track scores per judge/eval dimension separately if multiple judges are in use — a case can pass one dimension and fail another.

**If the golden dataset has fewer than 10 entries**, warn:
> "This dataset has fewer than 10 entries. Results will be directionally useful but not statistically reliable — a single case change can look like a large shift. Consider expanding the dataset to at least 20–30 entries for more confident signal."

---

## Step 4 — Compare to baseline

For each case, compare the new score to the baseline score. Classify each as one of four states:

| Previous | Current | State |
|---|---|---|
| Pass | Fail | **Regression** ❌ |
| Fail | Pass | **Improvement** ✅ |
| Pass | Pass | Unchanged (passing) |
| Fail | Fail | Unchanged (known failure) |

Count the totals across all cases and all dimensions.

---

## Step 5 — Produce the report

Output the report in this format:

---

### Regression run report

**Dataset:** [filename and entry count]
**Judges run:** [list of judges/rules used]
**Compared against baseline:** [baseline filename and date if available]

| Result | Count |
|---|---|
| Regressions ❌ | [N] |
| Improvements ✅ | [N] |
| Unchanged (passing) | [N] |
| Unchanged (known failures) | [N] |
| **Total cases** | [N] |

---

**[If regressions exist — show this section:]**

#### Regressions ❌

These cases were passing before and are now failing. Investigate before deploying.

For each regression:
> **Case [ID]:** [Short description of the input]
> **Judge/rule:** [Which eval flagged it]
> **Reasoning:** [Judge's reasoning if available, or the rule condition that failed]

---

**[If improvements exist — show this section:]**

#### Improvements ✅

These cases were failing before and are now passing.

For each improvement:
> **Case [ID]:** [Short description of the input]
> **Judge/rule:** [Which eval now passes]

---

**Deploy recommendation:**

**🔴 Do not deploy** — if any regressions exist.
> [N] regression(s) found. The prompt change broke cases that were previously passing. Investigate the regressions above before deploying. If these cases represent real user scenarios, fix the prompt and re-run. If they're edge cases you're consciously accepting, update the baseline to reflect the new expected behavior — but do that explicitly, not by accident.

**🟡 Deploy with monitoring** — if improvements exist and no regressions.
> No regressions. [N] improvement(s) detected. Safe to deploy — but keep online evals running after deployment to catch anything the golden dataset didn't cover.

**🟢 Deploy** — if no changes at all (all unchanged).
> Scores held steady. No regressions, no improvements detected. Safe to deploy.

---

**Save as new baseline?**

> This run's results are ready to save as the new baseline for future comparisons. Should I save them? If you're deploying this prompt version, saving now means the next run will compare against it.

---

## Step 6 — Save the baseline

If the developer confirms, save the current run results to the baseline file. Use a simple JSON format:

```json
{
  "saved_at": "[ISO timestamp]",
  "prompt_version": "[version or description if known]",
  "dataset": "[filename]",
  "results": [
    {
      "id": "[case id]",
      "score": "pass" | "fail",
      "judge": "[judge name]",
      "reasoning": "[reasoning if available]"
    }
  ]
}
```

Save to a file named `eval-baseline.json` in the same directory as the golden dataset, or wherever the existing baseline file was found.

Tell the developer:
> "Baseline saved. Run this skill again after your next prompt change to compare against it."

---

## Principles

**Offline evals are a gate, not a guarantee.** A clean regression run means you didn't break anything you already knew about — it doesn't mean the prompt is perfect. After deploying, online evals catch what the golden dataset didn't cover. Regression testing and production monitoring work together.

**Regressions are the only hard stop.** Improvements are good. Holding steady is acceptable. Regressions — cases that used to pass and now don't — are the signal to stop. Don't deploy through a regression without understanding why it happened.

**Updating the baseline is a deliberate decision.** The baseline represents the current known-good state. Overwriting it should be a conscious act, not a side effect of a run. If a developer chooses to ship despite a regression, they're accepting it as the new baseline — make that explicit.

**A thin dataset produces weak signal.** Ten cases is better than nothing. But a prompt change that improves three cases and breaks two looks like a net gain on a small dataset and might be noise. The more representative the dataset, the more trustworthy the signal.

**Golden datasets go stale.** A dataset from six months ago may not reflect what users are asking today. If the dataset hasn't been updated in a long time, flag it — cases from old failure patterns may no longer be relevant, and new failure patterns may not be covered.
