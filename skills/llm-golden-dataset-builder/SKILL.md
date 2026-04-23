---
name: llm-golden-dataset-builder
version: "1.0.0"
description: >
  Use this skill when a developer wants to build or expand a golden dataset for regression testing.
  Triggers on: "build a golden dataset", "create a test dataset", "curate eval examples",
  "promote logs to dataset", "build regression test cases", "I need a golden dataset",
  "select good examples for testing", "create a baseline dataset", "curate passing traces".
  Takes production logs with eval results and helps the developer select, review, and curate
  representative examples into a structured golden dataset ready for regression testing.
license: Apache-2.0
---

# LLM Golden Dataset Builder

You help developers build a golden dataset from their production logs — a curated collection of input/output pairs that represents what good behavior looks like. This dataset becomes the backbone of regression testing: every time the prompt changes, it runs against this dataset to confirm nothing broke.

A golden dataset is not just a collection of passing outputs. It's a deliberate selection that covers the real range of what users ask and what good responses look like — common cases, edge cases, and fixed failure patterns. The goal is to have examples where, if the prompt ever stopped handling them correctly, you'd want to know immediately.

**This skill runs as a code agent.** Search the codebase and project files for logs, traces, and eval results before asking the developer for anything.

---

## Step 1 — Find available artifacts

Search the project for:

- **Production logs or traces** — files containing real user interactions with LLM inputs and outputs. Look for `.json`, `.jsonl`, `.csv`, or `.log` files with conversation data, prompt inputs, and model responses.
- **Eval results attached to traces** — pass/fail scores per trace from judges or rules. Look for fields like `score`, `pass`, `fail`, `eval_result`, `evaluation`, or a composite score column.
- **Existing golden dataset** — any previously curated dataset file. If one exists, this run is an expansion, not a fresh build.
- **Judge prompts or eval configs** — to understand what dimensions are being evaluated, which helps assess coverage.
- **`CLAUDE.md` or context file** — may describe the eval setup or where traces are stored.

Report what you found. If traces exist but have no eval results attached, tell the developer:
> "I found production logs but no eval scores attached to them. To build a good golden dataset, you need to know which traces passed your evals — that's how you identify candidates. Run your evals against the logs first, or use `llm-regression-runner` to score them. If you don't have evals set up yet, use `llm-judge-creator` to build them."

If no traces exist at all, tell the developer:
> "No production logs found. A golden dataset should be grounded in real user interactions — traces from actual users are more valuable than invented examples because they capture behavior you didn't anticipate. If your feature isn't in production yet, you can use synthetic inputs to bootstrap the dataset, but plan to replace them with real traces as soon as you have traffic."

---

## Step 2 — Filter for candidates

From the available traces, identify golden dataset candidates: traces that passed all (or most) of the configured evals.

**Filtering logic:**
- **Primary candidates:** traces that passed every eval — these fit the full definition of good behavior as your evals currently measure it.
- **Secondary candidates:** traces that passed all evals except one — worth reviewing manually; the failure may be a false positive from an imprecise judge rather than a genuine quality problem.
- **Exclude:** traces with multiple eval failures. These are examples of what the prompt does wrong, not what it does right. They belong in issue reports, not in the golden dataset.

Tell the developer how many candidates you found:
> "Found [N] traces that passed all evals and [M] traces that passed all but one. Starting with the [N] clean passes as primary candidates."

If fewer than 20 candidates exist, flag it:
> "Only [N] candidates found. A golden dataset needs at least 20 entries for regression testing to be reliable. This run will give you a starting point — keep adding examples as more traffic accumulates."

---

## Step 3 — Check coverage

A golden dataset that only contains easy, common cases will miss regressions on edge cases. Before selecting, check whether the candidates cover the real range of usage.

Work through these coverage dimensions using the available traces:

**Common queries** — does the candidate pool include a spread of the most frequent request types? Identify the 2–3 most common query patterns in the full trace set and check whether candidates represent them proportionally.

**Edge cases** — are there candidates that involve ambiguous inputs, multi-part requests, unusual phrasing, or requests that previously caused failures? These are the cases most likely to break after a prompt change.

**Known fixed failures** — for each named failure pattern from `llm-issue-discovery` or the issue report, is there at least one candidate that tests the fix? If a past failure was "model recommends out-of-stock books," there should be a candidate that demonstrates the model now handles that correctly.

**Rare but important scenarios** — are there low-frequency but high-stakes cases represented? These might not show up often in production but matter a lot when they do.

Produce a coverage summary:

```
Coverage check:
- Common queries: [covered / thin — only N of M pattern types represented]
- Edge cases: [covered / none found in candidates]
- Known fixed failures: [covered / gap — no candidates for: issue X, issue Y]
- Rare/high-stakes scenarios: [covered / none identified]
```

**If gaps exist**, tell the developer what's missing and how to fill it:
- For missing edge cases: look deeper in the secondary candidates (passed all but one eval) or trigger those scenarios manually and add them once they pass.
- For missing fixed failures: if the fix is working, find a trace that demonstrates it and add it manually — even if you need to construct the input yourself.
- For thin common query coverage: pull more candidates from the same pattern type before moving to rarer cases.

---

## Step 4 — Review and confirm selections

Present candidates for the developer to confirm. Don't add anything to the golden dataset without the developer explicitly approving it — evals can produce false passes, and the dataset needs to reflect genuine quality, not just eval scores.

For each candidate, show:

```
--- Candidate [N] ---
Input: [the user input or conversation]
Output: [the model's response — full or summarized if long]
Evals passed: [list of eval names and their scores]
Suggested label: [what use case this represents — e.g., "common genre query", "inventory check edge case", "multi-part request"]
Include? [yes / no / skip]
```

Work through candidates in batches of 5–10. After each batch, ask whether to continue or adjust the selection criteria.

If the developer rejects a candidate, ask why — that reasoning helps identify whether an eval is producing false passes, which is useful signal for `llm-evals-audit`.

---

## Step 5 — Save the dataset

Once the developer has confirmed their selections, save the golden dataset as a structured file.

**Format — JSONL (one entry per line, preferred for large datasets):**

```jsonl
{"id": "case-001", "input": "[user input]", "output": "[model output]", "evals_passed": ["eval-name-1", "eval-name-2"], "label": "[use case description]", "added": "[ISO date]"}
```

**Format — JSON array (preferred for small datasets under 50 entries):**

```json
[
  {
    "id": "case-001",
    "input": "[user input or full conversation]",
    "output": "[model output]",
    "evals_passed": ["eval-name-1", "eval-name-2"],
    "label": "[what use case this represents]",
    "added": "[ISO date]"
  }
]
```

**If an existing golden dataset was found in Step 1:** append new entries to it rather than overwriting. Preserve existing entries and their IDs. Log how many entries were added.

Save the file as `golden-dataset.jsonl` (or `golden-dataset.json`) in the same directory as the project's eval artifacts, or wherever the developer specifies.

Confirm what was saved:
> "Golden dataset saved: [N] entries total ([M] new, [K] existing). File: [path]"

---

## Step 6 — Maintenance guidance

After saving, tell the developer:

> **Keep this dataset alive.** A golden dataset from six months ago may not reflect what users are asking today. Here's when to add entries:
>
> - **After fixing a failure pattern** — add at least one example that tests the fix. If the prompt ever regresses on it, you'll catch it.
> - **After discovering a new edge case** — if something unexpected happened in production and you handled it well, promote that trace.
> - **After a major prompt rewrite** — review the dataset for entries that no longer represent expected behavior, and refresh them.
>
> Aim to grow the dataset gradually toward 50–100 entries that cover the full range of real usage. Run `llm-regression-runner` against it before every prompt change.

---

## Principles

**Grounded in real interactions, not invented ones.** Synthetic inputs are faster to create but miss the messy reality of how users actually behave. A golden dataset built from real traces catches regressions on real behavior. Avoid constructing inputs unless you're filling a specific coverage gap with no real examples available.

**Passing evals is necessary but not sufficient.** A trace that passed all evals might still be a mediocre response — evals can be imprecise. Every entry needs a human to confirm it's genuinely good before it goes in. The developer's judgment is the final gate.

**Coverage beats volume.** Fifty well-selected entries covering common cases, edge cases, and known failure patterns are more valuable than two hundred examples of the same query type. Before adding more entries, check whether you're adding new coverage or just reinforcing what you already have.

**The dataset is a contract.** Every entry represents a behavior the system must preserve. If a prompt change breaks an entry, that's not a test failure to dismiss — it's a signal that something you explicitly decided to protect has regressed. Treat regressions seriously.

**Remove stale entries deliberately.** If a use case is no longer relevant — the feature changed, a capability was intentionally removed — remove the entry explicitly rather than letting it accumulate. A golden dataset with entries that are no longer meaningful produces noise in regression runs.
