---
name: llm-evals-checklist
version: "1.0.0"
description: >
  Use this skill when a developer wants to check if they're ready to build evaluations for their AI system.
  Triggers on: "am I ready to build evals", "can I start building evaluations", "eval readiness check",
  "pre-eval checklist", "what do I need before building evals", "how do I know if I'm ready to evaluate my AI",
  "should I build evals now", "eval prerequisites", "do I have enough to start building evals".
  Runs a readiness check across four prerequisites — tracing, logs, annotations, issues — before a developer
  invests time in building evaluations. Checks the codebase and project files first; only asks the developer
  when it cannot determine the answer itself.
license: MIT
---

# LLM Evals Checklist

You help developers confirm they have the right foundation in place before building evaluations. Building evals without this foundation produces evaluations that are generic, misaligned, or impossible to validate — wasted effort.

There are four prerequisites. Each one depends on the previous:

1. **Tracing** — your AI feature's inputs and outputs are being captured somewhere you can access them
2. **Logs** — you have a meaningful volume of real production interactions to work from
3. **Annotations** — a human has reviewed a sample of those logs and marked what worked and what didn't
4. **Issues** — those annotations have been used to identify specific, named failure patterns

You're ready to build evals when all four are in place. If any are missing, the checklist tells you what to do first.

**Where you are:** Meta-skill — run before starting the workflow to confirm you have the right foundation (tracing, logs, annotations, issues). If anything is missing, it tells you what to fix first.

**This skill is running as a code agent.** Before asking the developer anything, check the codebase and project files. Look for evidence of each prerequisite. Only ask when you genuinely can't determine the answer from what's available.

---

## Step 1 — Check for tracing / observability setup

Search the codebase for evidence that the AI feature's inputs and outputs are being captured.

**Look for:**
- Imports or SDK calls for observability tools: `latitude`, `@latitude-data/sdk`, `opentelemetry`, `langsmith`, `braintrust`, `helicone`, `traceloop`, or any wrapper around LLM calls that logs inputs/outputs
- Environment variables referencing observability: `LATITUDE_API_KEY`, `LANGCHAIN_TRACING`, `OTEL_EXPORTER_*`, etc.
- Any logging of prompt inputs and model responses to a file, database, or external service
- A `CLAUDE.md` or context file that mentions tracing or observability

**If found:** note the tool or approach and mark this as ✅.

**If not found:** ask:
> "I couldn't find any tracing or observability setup in this codebase. Are your AI feature's inputs and outputs being captured anywhere — a platform like Latitude, LangSmith, or Braintrust, or even logged to a file or database?"

If the answer is no, stop and tell them:
> "Tracing is the foundation. Without it, you have no logs, so nothing to annotate, and nothing to build evals from. Set up observability first — the [Latitude quickstart](https://docs.latitude.so) takes about 10 minutes — then come back here."

---

## Step 2 — Check for production logs

Check whether there are real production interactions available to work from.

**Look for:**
- Log files, trace exports, or data files in the project (`.json`, `.csv`, `.jsonl`, `.log`) that contain LLM inputs and outputs
- References to a logging destination (a database table name, an S3 bucket, a platform project) in config files, `.env`, or docs
- A `CLAUDE.md` or context file that mentions how many logs or traces exist

**If found:** estimate the volume if possible. Mark as ✅ if there appear to be 20+ interactions, ⚠️ if fewer.

**If not found:** ask:
> "I couldn't find any exported logs or traces in the project. Do you have production interactions captured somewhere — a platform dashboard, a database, or an exported file? Roughly how many do you have?"

**Minimum to proceed:** 20 interactions. Fewer than that and patterns won't be reliable. Tell them:
> "20 interactions is the practical minimum for spotting patterns. If you're below that, keep the feature running and let more traffic accumulate before starting evals — or generate synthetic test cases to supplement."

---

## Step 3 — Check for annotations

Check whether a human has reviewed and rated a sample of the logs.

**Look for:**
- Files containing human ratings, labels, or notes on LLM outputs — look for fields like `annotation`, `rating`, `label`, `score`, `thumbs_up`, `good`, `bad`, `feedback`, or similar
- A dataset or spreadsheet in the project with a column that distinguishes good from bad outputs
- References to annotations in a `CLAUDE.md`, context file, or README
- Any mention of an annotation workflow or human review process

**If found:** check whether bad/failing outputs have notes explaining what went wrong — not just scores. Mark as ✅ if notes exist, ⚠️ if only scores with no reasoning.

**If not found:** ask:
> "I couldn't find any annotated data in the project. Have you reviewed a sample of your production logs and marked which outputs were good or bad — with notes on what went wrong in the bad cases?"

If the answer is no, stop and tell them:
> "Annotations are what turn raw logs into a grounded issue list. Without them, any evals you build will be based on what you imagine might go wrong — not what's actually going wrong. Run `llm-annotation-guide` to set up an annotation rubric, then come back here after you've reviewed at least 20–30 outputs."

**Minimum to proceed:** 20 annotated outputs, with notes on the failures.

---

## Step 4 — Check for a named issue list

Check whether the annotations have been turned into specific, named failure patterns.

**Look for:**
- An issue report file in the project — any document or structured file that names failure patterns with frequency and description
- Output from `llm-issue-discovery` — look for files with names like `issue-report`, `failure-patterns`, `eval-issues`, or similar
- A `CLAUDE.md` or context file that lists named issues
- Any structured list of failure modes, even in a README or notes file

**If found:** check the quality of the issues. Good issues are specific ("model recommends out-of-stock books when inventory check fails") not generic ("bad responses" or "hallucination"). Check that each issue has at least 5 examples tied to it. Mark as ✅ if both conditions are met, ⚠️ if issues are vague or under-supported.

**If not found:** ask:
> "I couldn't find a named issue list or failure pattern report in the project. Have you run `llm-issue-discovery` on your annotated logs to identify specific failure patterns?"

If the answer is no, tell them:
> "Before building evals, you need a specific issue list — not generic concerns, but named failure patterns you've actually observed. Run `llm-issue-discovery` on your annotated logs. Aim for 3–8 named issues, each with at least 5 examples, before building evals."

---

## Step 5 — Output the readiness report

Once you've worked through all four checks, produce a report in this format:

---

### Eval readiness report

| Prerequisite | Status | Notes |
|---|---|---|
| Tracing set up | ✅ / ⚠️ / ❌ | [What you found, or what's missing] |
| Production logs | ✅ / ⚠️ / ❌ | [Volume found or reported, minimum is 20] |
| Annotations | ✅ / ⚠️ / ❌ | [Annotated count, whether notes exist on failures] |
| Named issues | ✅ / ⚠️ / ❌ | [Issue count, whether each has 5+ examples] |

---

**[If all four are ✅:]**
> You're ready to build evals. Here's the recommended path:
> 1. For each issue, decide what type of eval to build → `llm-eval-type-selector`
> 2. Build the LLM-as-judge prompts → `llm-judge-creator`
> 3. Validate your judges against your annotated examples → `llm-judge-alignment`

**[If any are ⚠️:]**
> You can start building, but address these gaps soon — they'll affect eval quality:
> [List each ⚠️ item with a one-line fix]

**[If any are ❌:]**
> Stop here. You're missing a foundation piece. Fix this first:
> [The first ❌ item, with a concrete next action. Don't list all blockers — just the first one. Fix it, then re-run this checklist.]

---

## Principles

**Check before asking.** This skill runs as a code agent. Always search the codebase and project files before asking the developer a question. They shouldn't have to answer what you can find yourself.

**One blocker at a time.** If multiple prerequisites are missing, report only the first one as a blocker. The others are moot until the first is resolved. Don't overwhelm with a list of everything that's wrong.

**Specificity is a prerequisite.** An issue named "bad responses" is not an issue. If the issue list is vague, the checklist should flag it as ⚠️ and say why — because a vague issue produces a vague eval that tells you nothing useful.

**5 examples is the floor, not the target.** Five examples per issue is the minimum needed to auto-generate an eval with reasonable alignment. More is always better — 20–30 is where alignment becomes reliable. Flag anything under 5 as ❌, anything 5–15 as ⚠️.

**Annotations without reasoning are weak signal.** A thumbs-down with no note tells you something went wrong but not what. An eval built from scores alone will be less targeted than one built from annotated reasoning. Always check whether the notes field is being used.
