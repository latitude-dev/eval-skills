# Eval Skills for AI Agents

A collection of skills for developers building with LLMs.

Each skill gives you a structured workflow for finding failure patterns, building evals, and validating that your evals actually work.

## Recommended workflow

These skills are designed to be used in order. Each step builds on the previous one.

```
1. llm-annotation-guide      ← Start here. Build a rubric and annotate your production logs.
2. llm-issue-discovery        ← Find and cluster failure patterns from your annotations.
3. llm-eval-type-selector     ← For each issue, decide: rule or judge?
4. llm-judge-creator          ← Build judge prompts for the issues that need one.
5. llm-judge-alignment        ← Validate your judges against human-labeled examples.
```

You don't have to follow this order strictly — if you already have annotated logs or a clear issue list, jump in at the right step. But if you're starting from scratch, annotations come before issue discovery. Human judgment is what grounds everything else.

## Skills

| Skill | What it does | Status |
|---|---|---|
| [llm-annotation-guide](./skills/llm-annotation-guide/) | Build an annotation rubric or review existing annotations for quality | Available |
| [llm-issue-discovery](./skills/llm-issue-discovery/) | Find and cluster failure patterns in LLM outputs | Available |
| [llm-eval-type-selector](./skills/llm-eval-type-selector/) | Decide whether to use a judge or a rule-based eval | Available |
| [llm-judge-creator](./skills/llm-judge-creator/) | Build LLM-as-a-judge prompts from issues or annotations | Available |
| [llm-judge-alignment](./skills/llm-judge-alignment/) | Validate how well a judge aligns with human judgment | Available |
| [llm-evals-checklist](./skills/llm-evals-checklist/) | Pre-build check: are you ready to build good evals? | Available |
| [llm-evals-audit](./skills/llm-evals-audit/) | Post-build check: are your existing evals healthy and well-targeted? | Available |
| [llm-regression-runner](./skills/llm-regression-runner/) | Run a golden dataset against your prompt, get a pass/fail report | Available |
| ai-agent-pre-launch | Pre-ship checklist: tracing, eval baseline, test suite | Coming soon |

## Install

**Option 1 — Claude Code plugin (recommended):**

```bash
/plugin marketplace add latitude-dev/eval-skills
/plugin install eval-skills@latitude-dev-eval-skills
```

To upgrade:
```bash
/plugin update eval-skills@latitude-dev-eval-skills
```

**Option 2 — npx skills CLI:**

```bash
npx skills add https://github.com/latitude-dev/eval-skills
```

Install a single skill only:
```bash
npx skills add https://github.com/latitude-dev/eval-skills --skill llm-issue-discovery
```

Check for updates:
```bash
npx skills check
npx skills update
```

## About

These skills package the methodology behind [Latitude](https://latitude.so) - an AI observability platform that helps dev teams find, track, and fix what's breaking in their AI before users notice.
