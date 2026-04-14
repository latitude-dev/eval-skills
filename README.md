# eval-skills

A collection of free Claude skills for developers building with LLMs.

Each skill gives you a structured workflow for finding failure patterns, building evals, and validating that your evals actually work.

## Skills

| Skill | What it does | Status |
|---|---|---|
| [llm-issue-discovery](./llm-issue-discovery/) | Find and cluster failure patterns in LLM outputs | Available |
| llm-judge-creator | Build LLM-as-a-judge prompts from issues or annotations | Coming soon |
| llm-judge-alignment | Validate how well a judge aligns with human judgment | Coming soon |
| llm-annotation-guide | Structure a good annotation session for your team | Coming soon |
| llm-eval-type-selector | Decide whether to use a judge or a rule-based eval | Coming soon |
| llm-evals-checklist | Pre-build check: are you ready to build good evals? | Coming soon |
| llm-evals-audit | Post-build check: are your existing evals healthy and well-targeted? | Coming soon |
| llm-regression-runner | Run a golden dataset against your prompt, get a pass/fail report | Coming soon |
| ai-agent-pre-launch | Pre-ship checklist: tracing, eval baseline, test suite | Coming soon |

## Install

Clone the repo and copy any skill folder into your `~/.claude/skills/` directory:

```bash
git clone https://github.com/latitude-dev/eval-skills.git
cp -r eval-skills/llm-issue-discovery ~/.claude/skills/
```

The skill will be available in Claude Code immediately — no restart needed.

## About

These skills package the methodology behind [Latitude](https://latitude.so) - an AI observability platform that helps dev teams find, track, and fix what's breaking in their AI before users notice.