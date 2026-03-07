# a2ui-adk

A [Claude Code](https://claude.com/claude-code) skill for building Google ADK agents with [A2UI](https://github.com/google/A2UI) — the declarative JSON format that lets agents generate rich, interactive UIs.

## Install

```bash
npx skills add coolxeo/a2ui-adk
```

That's it. The skill is now available in your Claude Code sessions.

## What it does

When you're building an ADK agent that needs to render dynamic UI (cards, forms, lists, interactive elements), this skill gives Claude Code the full context on:

- **A2UI schema management** — `A2uiSchemaManager`, catalog configuration, prompt generation
- **Single-agent pattern** — LlmAgent setup, streaming with validation/retry, A2A server hosting
- **Orchestrator pattern** — multi-agent UI routing, surface-to-subagent mapping, part converters
- **A2A transport** — agent cards with A2UI extensions, `DataPart` format, user action handling
- **Response parsing** — `<a2ui-json>` tag extraction, JSON validation against catalog schemas

## Usage

Just ask Claude Code to build an A2UI agent:

```
> Build me an ADK agent that shows a product catalog with cards and a booking form
> Add A2UI support to my existing ADK agent
> Create an orchestrator that routes to multiple A2UI sub-agents
```

The skill triggers automatically when you're working with A2UI + ADK.

## Uninstall

Remove the skill directory:

```bash
rm -rf ~/.claude/skills/a2ui-adk
```
