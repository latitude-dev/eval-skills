# eval-skills

A collection of free Claude skills for developers building with LLMs.

Each skill gives you a structured workflow for finding failure patterns, building evals, and validating that your evals actually work.

## Skills

| Skill | What it does | Status |
|---|---|---|
| [llm-issue-discovery](./llm-issue-discovery/) | Analyze LLM outputs, find failure patterns, get a prioritized issue report | Available |
| llm-eval-checklist | Check if your evals are actually good | Coming soon |
| llm-judge-creator | Create an LLM-as-judge based on annotated traces | Coming soon |
| llm-judge-alignment | Measure how well your LLM judge aligns with human annotations | Coming soon |
| llm-eval-type-selector | Decide whether to use an LLM judge or a rule-based eval | Coming soon |
| llm-eval-creator | Build targeted evals from your failure patterns | Coming soon |
| llm-eval-audit | Audit your existing evals for gaps and quality issues | Coming soon |

## Install

Clone the repo and copy any skill folder into your `~/.claude/skills/` directory:

```bash
git clone https://github.com/latitude-dev/eval-skills.git
cp -r eval-skills/llm-issue-discovery ~/.claude/skills/
```

The skill will be available in Claude Code immediately — no restart needed.

## About

These skills package the methodology behind [Latitude](https://latitude.so) - an AI observability platform that helps dev teams find, track, and fix what's breaking in their AI before users notice.