# ai-agent-stack skill

A Claude Code skill that adds production safety to any AI agent codebase in minutes.

## What it does

Installs and integrates three zero-dependency Python libraries:

- **ai-cost-guard** — hard budget cap before LLM calls (`pip install ai-cost-guard`)
- **ai-injection-guard** — prompt injection scanner (`pip install ai-injection-guard`)
- **ai-decision-tracer** — local agent decision tracer (`pip install ai-decision-tracer`)

## Install in Claude Code

```
/plugin marketplace add LuciferForge/lucifer-skills
```

Or install directly:
```
/plugin install ai-agent-stack@manja316
```

## Usage

Just describe what you want:
- "Add cost protection to my agent"
- "Instrument my LLM calls with tracing"
- "Protect my bot from prompt injection"
- "Make my agent production-ready"
- "Add all three safety layers"

Claude will analyze your codebase, find the LLM call points, and wrap them with the right guards.

## The libraries

| Library | GitHub | PyPI |
|---|---|---|
| ai-cost-guard | github.com/LuciferForge/ai-cost-guard | `pip install ai-cost-guard` |
| ai-injection-guard | github.com/LuciferForge/prompt-shield | `pip install ai-injection-guard` |
| ai-decision-tracer | github.com/LuciferForge/ai-trace | `pip install ai-decision-tracer` |

All three: MIT licensed, zero dependencies, pure Python stdlib, no network calls.

## License

Apache-2.0
