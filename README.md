# foundry-ai-mcp

Run a D&D 5e campaign in Foundry VTT by talking to your own AI agent in
**Claude Cowork** or **ChatGPT**.

- **A Foundry v14 module** that runs in a GM browser session and executes the full
  GM surface — documents, canvas, combat, dice, chat, audio, images, permissions —
  behind a safe, audited, undoable operation layer.
- **A self-hosted MCP server** exposing a public HTTPS endpoint your agent connects
  to as a custom connector. Your account, your infrastructure, no subscription,
  no third-party inference service, no telemetry.

> **Status: pre-implementation.** The design is written; the code is not.
> See [`docs/SPEC.md`](docs/SPEC.md) for scope, requirements, architecture, tool
> catalog, protocols, safety model, roadmap and open questions.

## Why it's built this way

Claude connectors — including in Cowork — connect from Anthropic's cloud, not from
your machine, and local stdio MCP servers aren't available there at all. ChatGPT's
Developer mode likewise wants a remote HTTPS URL. So unlike every other Foundry MCP
project, this one is **not** a localhost stdio server: it's a small self-hosted
service you run next to Foundry, on a cheap VPS, or behind a Cloudflare Tunnel.

## Design principles

1. **Anything a GM can do, the agent can do** — including an opt-in scripting escape hatch.
2. **Your account, your data, your server** — nothing routed through project-operated services.
3. **No irreversible surprise** — every mutation is diffable, logged, snapshot-protected and undoable.
4. **Ingest images, don't generate them** — from a URL, an upload, or your own files.

## Scope

Foundry VTT **v14 (LTS)** only · **D&D 5e** only · Node 22+ for the server ·
**MIT, permanently**.
