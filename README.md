# foundry-ai-mcp

Chat with your own AI account and let it run Foundry VTT as a Game Master would.

Two parts, one capability core:

- **A Foundry VTT module** that runs in the GM's browser and exposes the full GM
  surface — documents, canvas, combat, dice, chat, audio, permissions — behind a
  safe, audited, undoable operation layer.
- **An MCP server** that lets any Model Context Protocol client (Claude Desktop,
  Claude Code, Cursor, …) drive it. Your account, your credentials, no
  subscription, no third-party inference service, no telemetry.

Optionally, an in-Foundry chat sidebar so you never have to alt-tab mid-session.

> **Status: pre-implementation.** The design is written; the code is not.
> See [`docs/SPEC.md`](docs/SPEC.md) for scope, requirements, architecture,
> tool catalog, bridge protocol, safety model, roadmap and open decisions.

## Design principles

1. **Anything a GM can do, an agent can do** — including an opt-in scripting escape hatch.
2. **Your account, your data** — nothing routed through project infrastructure.
3. **No irreversible surprise** — every mutation is diffable, logged, and undoable.
4. **System-agnostic first** — runtime schema discovery, with adapters as ergonomics rather than prerequisites.

## Targets

Foundry VTT v13 minimum, v14.365 verified. Node 20+. MIT licensed.
