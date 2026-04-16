# Eval Skills for AI Agents

A collection of skills for developers building with LLMs.

Each skill gives you a structured workflow for finding failure patterns, building evals, and validating that your evals actually work.

## Skills

| Skill | What it does | Status |
|---|---|---|
| [llm-issue-discovery](./llm-issue-discovery/) | Find and cluster failure patterns in LLM outputs | Available |
| [llm-annotation-guide](./llm-annotation-guide/) | Build an annotation rubric or review existing annotations for quality | Available |
| [llm-judge-creator](./llm-judge-creator/) | Build LLM-as-a-judge prompts from issues or annotations | Available |
| [llm-judge-alignment](./llm-judge-alignment/) | Validate how well a judge aligns with human judgment | Available |
| llm-eval-type-selector | Decide whether to use a judge or a rule-based eval | Coming soon |
| llm-evals-checklist | Pre-build check: are you ready to build good evals? | Coming soon |
| llm-evals-audit | Post-build check: are your existing evals healthy and well-targeted? | Coming soon |
| llm-regression-runner | Run a golden dataset against your prompt, get a pass/fail report | Coming soon |
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