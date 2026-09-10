# Foundry AI MCP — Scope, Requirements & Specification

**Project:** `foundry-ai-mcp`
**Status:** Draft v0.1 (pre-implementation)
**Target:** Foundry Virtual Tabletop v13 (minimum) / v14.365 (verified)
**License:** MIT

---

## 1. Executive summary

A two-part open-source system that lets a Game Master drive Foundry VTT through
natural-language conversation with **their own AI account** — no vendor
subscription, no third-party inference service, no telemetry.

1. **The Module** (`foundry-ai-mcp`) — a Foundry VTT add-on module that runs in the
   GM's browser session and exposes the full GM capability surface (documents,
   canvas, combat, dice, chat, audio, permissions) behind a safe, audited,
   undoable operation layer. Optionally provides an in-Foundry chat sidebar.
2. **The MCP Server** (`@rashthepay/foundry-mcp`) — a local Node process speaking
   [Model Context Protocol](https://modelcontextprotocol.io) to any MCP client
   (Claude Desktop, Claude Code, claude.ai, Cursor, …), translating tool calls
   into module operations over a local WebSocket bridge.

### Positioning

| | Familiar | adambdooley/foundry-vtt-mcp | **foundry-ai-mcp** |
|---|---|---|---|
| Model access | Vendor subscription or BYO OpenAI-compatible key | BYO Claude subscription via MCP | **BYO account — MCP client _or_ in-Foundry panel** |
| Cost | Paid subscription | Free | **Free, MIT** |
| Chat surface | In-Foundry | External client only | **Both** |
| Write scope | Curated GM actions | ~41 curated tools | **Full document/canvas surface + escape hatch** |
| Safety | Vendor-defined | Read-only toggle | **Mode matrix, dry-run, confirm, transaction log, undo** |
| Systems | Several, curated | 6 systems | **Schema-driven generic + adapters (5e, PF2e)** |

**The differentiators worth building for:** (a) transactional undo, (b) runtime
schema discovery instead of hardcoded per-system tools, (c) two front doors onto
one capability core, (d) an explicit autonomy/permission model.

---

## 2. Goals and non-goals

### Goals

- **G1** — Anything a GM can do through the Foundry UI, an agent can do through a tool call.
- **G2** — The GM's own AI account and credentials; nothing routed through project infrastructure.
- **G3** — No irreversible surprise. Every mutation is logged, diffable, and undoable.
- **G4** — System-agnostic by default; system adapters improve ergonomics, they are not a prerequisite.
- **G5** — Works with hosted Foundry (Forge, Molten, self-hosted remote) as long as the GM's browser and the MCP server share a machine.
- **G6** — Installable by a non-developer in under ten minutes.

### Non-goals (v1)

- Player-facing assistants with their own permissions (post-1.0).
- Running without a GM browser session open (architecturally impossible — see §4.1).
- Shipping copyrighted game content (statblocks, adventures, art).
- Hosting an inference service, a relay, or any account system.
- Image/map generation as a core dependency (optional integration only).
- Voice I/O (post-1.0).

---

## 3. Personas & representative user stories

**P1 — Prep GM (async, between sessions).** Uses Claude Desktop/Code on a second monitor.
- "Read my Session 12 notes and build the three encounters I sketched, place them on the Ruined Chapel scene, and stat the cult leader as a CR 5 caster."
- "Every NPC in my Journal folder 'Waterdeep' that doesn't have an actor — create one and link it."
- "Summarise last session from the chat log into a recap journal page and show it to the players."

**P2 — Live GM (mid-session, hands on the table).** Uses the in-Foundry chat panel.
- "The party just set the tavern on fire. Give me a d20 table of consequences and roll on it."
- "Bandit 3 dashes to the balcony and shoots the wizard — resolve it."
- "Everyone rolls a DC 15 Dexterity save."

**P3 — World builder / tinkerer.** Bulk operations and data hygiene.
- "Find every item in the world with no icon and assign one from the compendium."
- "Convert all my v10-era journals to multi-page format."

**P4 — Module developer.** Registers a system adapter or extends the tool surface.

---

## 4. Architecture

### 4.1 The binding constraint

Foundry's meaningful API (`game.actors`, `canvas`, `Document` classes, `Roll`,
`Combat`) lives **client-side, in the browser**. There is no official server-side
world API and modules do not execute on the Foundry server. Therefore:

> Every write must be executed by a browser session authenticated as a GM user.

The module is that execution arm. A GM tab must be open and connected. This is a
hard constraint shared by every Foundry AI integration; it is stated up front
rather than discovered in week three.

Because a browser cannot accept inbound connections, the **module dials out** to
the MCP-side bridge. Connection direction: `module (WS client) → daemon (WS server)`.

### 4.2 Components

```
┌────────────────────┐        ┌──────────────────────────────────────┐
│  MCP client        │ stdio  │  foundry-mcp (server)                │
│  Claude Desktop /  │◄──────►│   ├─ tools / resources / prompts     │
│  Claude Code / …   │  MCP   │   ├─ zod schemas (shared package)    │
└────────────────────┘        │   └─ bridge client ──┐               │
                              └──────────────────────┼───────────────┘
                                                     │ local IPC
┌────────────────────┐                     ┌─────────▼──────────────┐
│  In-Foundry chat   │  Anthropic API      │  bridge daemon         │
│  panel (optional)  │  (BYO key)          │  ws://127.0.0.1:31415  │
│         │          │                     │  auth · multiplex      │
└─────────┼──────────┘                     └─────────▲──────────────┘
          │                                          │ WSS/WS + token
┌─────────▼──────────────────────────────────────────┴──────────────┐
│  foundry-ai-mcp module  (runs in GM's browser)                     │
│   bridge client · op router · safety layer · txn log · adapters    │
│                              ▼                                     │
│              Foundry VTT client API (game.*, canvas.*)             │
└────────────────────────────────────────────────────────────────────┘
```

**Why a separate daemon?** An MCP stdio server's lifetime is owned by its client —
it dies when Claude Desktop closes, taking the module's socket with it. A small
persistent daemon (auto-started, port-locked, shared) lets multiple MCP clients
and the in-Foundry panel share one Foundry connection and survives client restarts.
Single binary, two modes: `foundry-mcp serve --stdio` and `foundry-mcp daemon`.

### 4.3 Deployment topologies

| Topology | Foundry server | GM browser | MCP server | Supported |
|---|---|---|---|---|
| T1 Local all-in-one | localhost | same machine | same machine | v1 ✅ |
| T2 Hosted Foundry (Forge etc.) | remote | GM machine | GM machine | v1 ✅ |
| T3 Foundry on LAN box | LAN | GM machine | GM machine | v1 ✅ |
| T4 MCP on a different machine than browser | any | machine A | machine B | v1 ⚠️ requires LAN-bound daemon + token, documented but unsupported |
| T5 Mobile / claude.ai web client | any | GM machine | needs public relay | post-1.0 ❌ |

### 4.4 Two front doors, one core

The module's **capability layer** is a single registry of operations with zod-validated
inputs. Both the MCP server and the in-Foundry chat panel consume the *same* registry.
The MCP server generates MCP tool definitions from it; the panel generates Anthropic
tool definitions from it. Adding an operation exposes it to both surfaces at once.

---

## 5. Capability surface — what "anything a GM can do" means

This is the scope definition. Each row is in v1 unless marked.

### 5.1 World documents (CRUD)

Actor, Item, Scene, JournalEntry, RollTable, Playlist, Macro, Cards, Combat,
ChatMessage, Folder, User, Setting, plus embedded documents: ActiveEffect, Token,
Tile, Wall, AmbientLight, AmbientSound, Drawing, MeasuredTemplate, Note, Region,
Combatant, JournalEntryPage, TableResult, PlaylistSound, and Items owned by Actors.

Operations: create, read, update (deep-merge or replace), delete, duplicate, move
between folders, reorder (`sort`), set ownership, add/remove flags, batch variants
of all of the above.

### 5.2 Compendium packs

List packs (world/system/module), full-text and filtered search, index reads,
import documents into the world, export world documents into a pack, create and
delete world compendiums, lock/unlock, folder structure inside packs.

### 5.3 Scene & canvas

Create/clone/delete scenes; set background, foreground, dimensions, padding, grid
(type, size, units, distance), initial view; activate vs. view; scene navigation
order. Configure lighting: darkness level, global illumination, illumination colours,
fog exploration, vision. Weather effects. Token vision toggles.

Place and edit walls (with door type, door state, sense restrictions, thresholds,
directional flags), lights (dim/bright radius, angle, animation, colour), sounds
(radius, easing, walls), tiles (with occlusion, video), drawings (shapes, text),
notes (journal pins), regions (v12+ with behaviours), measured templates.

Runtime canvas: pan/zoom camera, ping a location, select/target tokens, toggle
token visibility and elevation, move tokens (animated or teleport), reset or
reveal fog of war, open/close/lock doors, cycle scene to players.

### 5.4 Combat

Create an encounter, add/remove combatants, roll initiative (per combatant,
group, or all; with formula override), reorder, set turn, advance turn/round,
step back, end combat, delete encounter. Read live combat state (order, HP,
conditions, resources, targeted tokens). Apply damage and healing with type and
resistance handling via the system adapter. Toggle status conditions. Award XP.

### 5.5 Dice & randomness

Evaluate any Foundry `Roll` formula with full data context; roll modes
(public / GM-only / blind / self); roll as a specific speaker; roll tables with
or without replacement; nested tables; recursive draws. **Request rolls from
players**: post an actionable chat card to named users, then await or poll the
result (see §7.5 events).

### 5.6 Chat

Post messages as GM, as an actor (IC), as OOC, as emote; whisper to selected
users; post rich HTML and enriched Foundry links (`@UUID[...]`); post roll
messages; read recent history with filters; delete messages.

### 5.7 Journals, tables, handouts

Create multi-page journal entries (text, image, PDF, video pages); append to and
edit pages; reorder pages; show a page or image to players; create and roll
RollTables; generate quest/recap/lore content; create Cards decks.

### 5.8 Audio

List playlists and tracks; play, stop, pause, loop, crossfade; set channel and
track volume; play a one-shot; control ambient sounds on scene.

### 5.9 Macros & scripts

List, create, edit, delete macros (script and chat types); execute a macro with
arguments; assign to hotbar.

### 5.10 Users & permissions

List users (role, online status, assigned character); create/update users
(post-1.0 for create); assign character; set document ownership levels; read and
write module and system settings; toggle module settings.

### 5.11 Escape hatch (expert mode, opt-in)

- `system.execute_script` — arbitrary JS in the GM client with serialised result and console capture.
- `system.call_api` — reflective invocation of a dotted API path with arguments.

These make "anything a GM can do" literally true and cover system-specific
functionality no curated tool anticipates. Off by default; require explicit
opt-in plus per-call confirmation; always logged verbatim.

### 5.12 Meta / session control

Transaction log read, undo, snapshot & restore, autonomy mode switching,
capability introspection, health check, kill switch.

---

## 6. Tool design

### 6.1 The tool-count problem

A naive one-tool-per-capability mapping of §5 yields 150–200 tools. That degrades
tool selection accuracy and burns context on every turn. The design response:

**Three layers.**

1. **Ergonomic tools (~40)** — the high-frequency GM path, hand-written with tight
   schemas and good descriptions. This is what the model reaches for 90% of the time.
2. **Generic document tools (6)** — `documents.query`, `documents.get`,
   `documents.create`, `documents.update`, `documents.delete`, `documents.describe_schema`.
   These reach anything the ergonomic tools don't, in any game system, because the
   model can discover the data model at runtime.
3. **Escape hatch (2)** — §5.11, opt-in.

**Tool profiles.** A module setting selects which sets are advertised:
`core` (~20) · `standard` (~40, default) · `full` (~55) · `expert` (+escape hatch).
The MCP server emits `notifications/tools/list_changed` when the profile changes.

**Runtime schema discovery** is the key to system-agnosticism.
`documents.describe_schema({ type: "Actor", subtype: "npc" })` returns a JSON Schema
derived from the live `DataModel.schema` plus the system's `template.json`, with
example values pulled from an existing document. The model then writes correct
`system.*` paths for *any* game system without the project shipping per-system code.

### 6.2 Tool catalog (v1)

Namespaced `verb_noun`, snake_case. `†` = mutating. `‡` = expert-only.

**world** — `get_world_info`, `get_capabilities`, `health_check`

**documents** — `query_documents`, `get_document`, `describe_schema`,
`create_documents`†, `update_documents`†, `delete_documents`†, `duplicate_document`†,
`manage_folders`†

**actors** — `list_actors`, `get_actor`, `create_actor_from_statblock`†,
`modify_actor_resource`†, `manage_actor_items`†, `apply_effect`†, `set_ownership`†

**compendium** — `list_packs`, `search_compendium`, `import_from_compendium`†,
`export_to_compendium`†

**scenes** — `list_scenes`, `get_scene`, `create_scene`†, `configure_scene`†,
`activate_scene`†

**canvas** — `place_objects`† (tokens/tiles/lights/sounds/notes/drawings/templates/regions),
`update_placeables`†, `delete_placeables`†, `edit_walls`†, `move_token`†,
`set_token_state`† (hidden/elevation/conditions/bar values), `control_camera`†
(pan/zoom/ping/select/target), `manage_fog`†, `set_door_state`†

**combat** — `get_combat_state`, `start_combat`†, `manage_combatants`†,
`roll_initiative`†, `advance_combat`†, `end_combat`†, `apply_damage`†, `toggle_condition`†

**dice** — `roll_dice`†, `roll_table`†, `request_player_roll`†, `await_event`

**chat** — `send_chat_message`†, `get_chat_history`, `delete_chat_messages`†

**journal** — `create_journal`†, `update_journal_page`†, `show_to_players`†,
`search_journals`

**audio** — `list_audio`, `control_audio`†

**macros** — `list_macros`, `create_macro`†, `execute_macro`†

**users** — `list_users`, `manage_settings`†

**system** ‡ — `execute_script`†, `call_api`†

**session** — `get_transaction_log`, `undo`†, `create_snapshot`†, `restore_snapshot`†,
`set_autonomy_mode`†

### 6.3 Universal tool conventions

Every mutating tool accepts:

| Field | Type | Meaning |
|---|---|---|
| `dry_run` | boolean | Compute and return the diff; apply nothing. |
| `reason` | string | One line shown in the confirmation dialog and audit log. |
| `txn_group` | string | Group several calls into one undoable transaction. |

Every read tool accepts `limit` (default 25), `cursor`, and `fields` (projection —
critical for keeping large Actor payloads out of context).

Every response returns:

```jsonc
{
  "ok": true,
  "data": { /* ... */ },
  "txn_id": "txn_01J...",        // present for mutations
  "affected": [{ "uuid": "...", "name": "...", "op": "update" }],
  "warnings": ["..."],
  "truncated": false             // with next_cursor when true
}
```

Errors are typed, never bare strings: `NOT_FOUND`, `PERMISSION_DENIED`,
`CONFIRMATION_TIMEOUT`, `CONFIRMATION_DENIED`, `VALIDATION_FAILED`,
`SYSTEM_UNSUPPORTED`, `BRIDGE_DISCONNECTED`, `RATE_LIMITED`, `READONLY_MODE`,
`PAYLOAD_TOO_LARGE`. Each carries `hint` — a remediation the model can act on.

### 6.4 MCP resources (cheap context, no tool call)

- `foundry://world/summary` — system, version, scene, users, module list
- `foundry://scene/current` — active scene with placeable counts and token roster
- `foundry://combat/current` — live initiative order with HP and conditions
- `foundry://actors/index` — id/name/type/CR index of all actors
- `foundry://journal/index`
- `foundry://uuid/{uuid}` — any document by UUID

### 6.5 MCP prompts (workflow starters)

`prep_session`, `run_combat_round`, `improvise_npc`, `session_recap`,
`statblock_from_text`, `build_encounter`, `dress_the_scene`, `audit_world`.

---

## 7. Bridge protocol

### 7.1 Envelope

```jsonc
{
  "v": 1,
  "id": "01J8XY...",           // ULID, correlates req/res
  "type": "req" | "res" | "event" | "hello" | "ping" | "pong",
  "op": "documents.update",     // req only
  "payload": { },
  "meta": { "dryRun": false, "txnGroup": null, "clientId": "claude-desktop", "ts": 0 }
}
```

### 7.2 Handshake & auth

1. Module reads `bridgeToken` from module settings (generated on first run, shown in the settings UI with a copy button).
2. Module connects to `ws://127.0.0.1:31415` and sends `hello` with `{ token, worldId, systemId, foundryVersion, moduleVersion, userId, userRole }`.
3. Daemon validates the token (constant-time compare), rejects non-GM roles, and replies with its own version and negotiated protocol version.
4. Version mismatch → structured error and a clear in-Foundry notification, not a silent failure.

Bind to `127.0.0.1` only by default. LAN binding (T4) is a separate setting that
forces TLS and a long token, with an explicit warning.

### 7.3 Liveness

Heartbeat every 15s; 45s without a pong closes the socket. Module reconnects with
exponential backoff (1s → 30s, jittered) and unlimited retries. Connection state
is visible in the Foundry UI (a coloured dot in the sidebar) — never a mystery.

### 7.4 Large payloads

Frames cap at 1 MiB. Larger results are stored server-side under a `result_ref`
and fetched in pages via `session.fetch_result`. Actor documents in some systems
exceed 200 KB; projection (`fields`) is the primary mitigation, refs the fallback.

### 7.5 Events (module → server, unsolicited)

`combat.turn_changed`, `combat.started`, `combat.ended`, `roll.completed`,
`chat.message`, `token.moved`, `scene.activated`, `document.changed`,
`confirmation.resolved`, `user.connected`.

MCP has no reliable server-initiated push to the model mid-turn, so events are
consumed two ways: a bounded ring buffer readable via `dice.await_event`
(long-poll, configurable timeout, returns immediately if a matching event is
already buffered), and — for the in-Foundry panel — direct callback, enabling true
agentic loops ("run the goblins' turn, then wait for the players").

---

## 8. Safety, permissions, audit, undo

This section is a requirement, not a disclaimer. An agent with GM rights can
delete a campaign.

### 8.1 Autonomy modes

| Mode | Reads | Reversible writes | Destructive writes | Scripts |
|---|---|---|---|---|
| `readonly` | ✅ | ❌ | ❌ | ❌ |
| `assist` **(default)** | ✅ | ✅ auto | ⚠️ confirm | ❌ |
| `autonomous` | ✅ | ✅ auto | ✅ auto (logged) | ⚠️ confirm |
| `expert` | ✅ | ✅ | ✅ | ⚠️ confirm |

"Destructive" = deletes, ownership/permission changes, mass updates over the
threshold (default 25 documents), scene activation, world setting writes.

### 8.2 Per-tool permission matrix

A settings UI grid: every tool × {allow, confirm, deny}. Mode presets seed it;
the GM can override any cell. Persisted per world.

### 8.3 Confirmation UX

A non-blocking Foundry dialog showing: tool name, the model's `reason`, a rendered
diff of the change (field-level, old → new), affected document count and names, and
Approve / Approve-and-remember-for-this-session / Deny. 60-second timeout defaults
to **deny**. Bulk operations show a paginated affected-document list.

### 8.4 Transaction log & undo

Every mutating op opens a transaction that records the **inverse operation** before
applying:

| Forward | Inverse |
|---|---|
| create | delete by `_id` |
| update | update with captured pre-state of the touched paths |
| delete | create with full captured source data, preserving `_id` |
| batch | inverses applied in reverse order |

Stored as a rolling buffer (default 50 transactions, configurable) in a world-scoped
setting, with overflow spilled to `Data/foundry-ai-mcp/txn/*.jsonl`. Exposed as
`session.undo({ txn_id? })` and as an Audit Log sidebar application with a per-entry
Undo button.

**Known undo limits — document these in the UI, not just here:** re-created
documents may lose references from *other* documents that pointed at them if those
references were also mutated; active effects re-applied out of order can produce
different derived values; canvas animation state is not restored. Undo is a strong
safety net, not a database transaction. For anything genuinely important,
`session.create_snapshot` (a full JSON export of the selected collections) is the
real answer, and the module nags for one before any operation touching >100 documents.

### 8.5 Rate limits & circuit breaker

Default 60 writes/minute and 500 documents/minute. Exceeding either pauses the
bridge and raises an in-Foundry prompt. A **kill switch** (sidebar button plus a
configurable hotkey, default `Ctrl+Shift+X`) drops the socket instantly and flips
to `readonly`.

### 8.6 Prompt injection

World content is untrusted input. Journal text, chat messages, item descriptions,
and player-authored content can contain instructions aimed at the model. Mitigations:

- All world-derived text is returned wrapped in an explicit untrusted-content
  envelope with a standing instruction that it is data, never instruction.
- Destructive tools always confirm in `assist` mode regardless of how convincing
  the justification is.
- The audit log records the `reason` string, making injected instructions visible after the fact.
- Documented in the README as a real, unsolved-in-general risk.

### 8.7 Privacy

No project-operated servers. No telemetry. The bridge is loopback-only by default.
The model provider sees whatever world content is sent as tool results — stated
plainly in the README, with the projection (`fields`) mechanism offered as the
mitigation and semantic indexing shipped **off by default**.

---

## 9. System adapters

```ts
interface SystemAdapter {
  readonly id: string;                 // "dnd5e" | "pf2e" | "generic"
  readonly systemVersions: string;     // semver range

  getVitals(actor): { hp, maxHp, tempHp, ac, level, cr } | null;
  applyDamage(actor, amount, opts): Promise<void>;
  applyHealing(actor, amount): Promise<void>;
  conditions(): Array<{ id, label, icon }>;
  toggleCondition(token, id, active): Promise<void>;
  rollCheck(actor, kind, key, opts): Promise<Roll>;   // ability/save/skill/attack
  initiativeFormula(combatant): string;
  parseStatblock(text): Promise<ActorData>;
  encounterBudget(partyLevels, difficulty): { xp, crRange };
  itemUse(actor, item, opts): Promise<void>;
  resources(actor): Array<{ key, label, value, max }>;
}
```

**`generic` adapter** (always present, always the fallback): derives everything it
can from `describe_schema` plus Foundry core concepts (token bars, `CONFIG.statusEffects`,
`Actor#getRollData`). Degrades to `SYSTEM_UNSUPPORTED` with a hint pointing at the
generic document tools rather than failing opaquely.

**Shipped v1:** `generic`, `dnd5e`. **v1.0:** `pf2e`.
**Third-party:** `game.modules.get("foundry-ai-mcp").api.registerAdapter(adapter)`
during the `foundry-ai-mcp.ready` hook.

---

## 10. Memory & retrieval (optional, off by default)

- Local index of journals, actor/item descriptions, chat history, and selected
  compendia, stored under `Data/foundry-ai-mcp/index/` (sqlite-vec or LanceDB in
  the MCP server process).
- Embeddings: local by default (`transformers.js`, `all-MiniLM-L6-v2`), with an
  option to use a provider embedding endpoint.
- Incremental reindex driven by `document.changed` bridge events, debounced.
- Exposed as `journal.search_journals({ mode: "semantic" | "keyword" | "hybrid" })`.
- **Campaign log**: an opt-in journal the module appends per-session summaries to,
  giving the agent long-term continuity without re-reading the whole world.

---

## 11. In-Foundry chat panel (M5)

A Foundry `ApplicationV2` sidebar tab.

**Requirements**
- BYO credentials: Anthropic API key, or an OpenAI-compatible base URL + key (so
  Ollama/LM Studio/OpenRouter work). Stored in a **client-scoped** setting (per
  user, not synced into the world database).
- Streaming responses with visible tool-call cards: name, arguments, diff preview,
  result, and an inline Undo button.
- Threads persisted per user; a session thread and named saved threads.
- Slash commands mirroring the MCP prompts (`/recap`, `/encounter`, …).
- Context injection: current scene, selected tokens, active combat, and targeted
  tokens are attached automatically as structured context.
- Respects the same permission matrix and confirmation flow as the MCP path.
- **Never** stores API keys in the world database or exposes them to non-GM users.

**Note on cost model.** The MCP path uses the GM's existing Claude subscription
(no per-token billing). The in-Foundry panel calls the API directly and *is*
per-token billed. Surface this clearly in the UI — it is the single most likely
source of user surprise in the whole project.

---

## 12. Non-functional requirements

| Area | Requirement |
|---|---|
| Foundry compatibility | minimum v13, verified v14.365; ApplicationV2 UI only; no jQuery-era APIs |
| Systems | any; `generic` adapter guarantees baseline function |
| Browsers | Chromium 120+, Foundry Electron app; Firefox best-effort |
| Node | 20 LTS+ for the MCP server; ESM; TypeScript strict |
| Latency | p95 < 400 ms for local reads; < 1 s for single-document writes |
| Payloads | 25-item default page size; 1 MiB frame cap; projection encouraged |
| Footprint | module bundle < 500 KB gzipped; no runtime CDN fetches |
| i18n | all user-facing strings in `lang/en.json`; no hardcoded strings |
| Accessibility | keyboard-navigable panel, ARIA live region for streaming, respects Foundry themes and `prefers-reduced-motion` |
| Offline | full function with a local model endpoint; no mandatory internet |
| Logging | structured, level-configurable, redacts tokens and API keys |

---

## 13. Repository layout & stack

```
foundry-ai-mcp/
├─ packages/
│  ├─ shared/                 # protocol types + zod schemas (source of truth)
│  │  └─ src/{protocol,ops,errors}.ts
│  ├─ module/                 # Foundry module — TS, bundled with Vite
│  │  ├─ module.json
│  │  ├─ src/
│  │  │  ├─ main.ts
│  │  │  ├─ bridge/           # ws client, handshake, reconnect, framing
│  │  │  ├─ ops/              # one handler file per namespace (§6.2)
│  │  │  ├─ safety/           # modes, matrix, confirm dialog, rate limit
│  │  │  ├─ txn/              # transaction log, inverse ops, undo, snapshots
│  │  │  ├─ adapters/         # generic, dnd5e, pf2e, registry
│  │  │  ├─ ui/               # chat panel, settings, audit log (AppV2)
│  │  │  └─ schema/           # describe_schema introspection
│  │  ├─ styles/  templates/  lang/
│  └─ server/                 # MCP server + bridge daemon
│     └─ src/
│        ├─ stdio.ts  daemon.ts  bridge.ts
│        ├─ tools/  resources/  prompts/
│        └─ memory/
├─ docs/                      # this spec, protocol ref, tool ref, guides
├─ tests/{unit,e2e,quench}/
└─ .github/workflows/
```

**Stack:** TypeScript everywhere · zod (single schema source → MCP JSON Schema,
Anthropic tool defs, and runtime validation) · `@modelcontextprotocol/sdk` · `ws` ·
Vite (module bundle) · tsup (server) · vitest · Playwright · Quench (in-Foundry tests) ·
changesets for versioning.

**`module.json` essentials:** `id`, `title`, `description`, `version`, `authors`,
`compatibility: { minimum: "13", verified: "14" }`, `esmodules`, `styles`,
`languages`, `socket: true`, `url`, `manifest`, `download`, `relationships`,
`flags`. `socket: true` is required for the module's own Foundry socket namespace
(used for GM↔player interactions such as roll requests).

---

## 14. Testing & CI

- **Unit (vitest)** — op handlers against a mocked `game`/`canvas`; inverse-op
  correctness (property test: `apply(inverse(apply(op))) == identity`); schema
  validation; protocol framing.
- **Integration (vitest)** — MCP server ↔ daemon ↔ fake module client; full tool
  round-trips; error taxonomy; rate limits; reconnection.
- **In-Foundry (Quench)** — real Document CRUD, undo fidelity, adapter behaviour,
  run against a live world.
- **E2E (Playwright)** — a Dockerised Foundry (`felddy/foundryvtt`) with dnd5e and
  pf2e installed; log in as GM, load the module, drive a scripted session, assert
  world state. Gated on a licence secret; skipped on forks.
- **Golden-path evals** — a fixture world plus ~30 natural-language tasks scored on
  correct tool selection and end state. This is the only way to catch tool-description
  regressions, and it should exist from M1.
- **CI** — lint, typecheck, unit + integration on every PR; E2E nightly; release
  builds `module.zip` + `module.json` and publishes the npm package.

---

## 15. Distribution

1. **Foundry module** — GitHub Release with `module.zip` and a stable
   `module.json` URL; submitted to the Foundry package registry once past v0.5.
2. **MCP server** — npm `@rashthepay/foundry-mcp`, runnable via `npx`; config
   snippet for `claude_desktop_config.json` and `.mcp.json` in the docs.
3. **One-click** — a `.mcpb` extension bundle for Claude Desktop (bundles the Node
   runtime and the server; user pastes only the bridge token).
4. **Docs site** — install guide per topology, tool reference generated from the
   zod schemas, safety guide, adapter authoring guide.

Installation success criterion: a GM who has never used a terminal is issuing their
first command within ten minutes, following the Claude Desktop path.

---

## 16. Roadmap

| Milestone | Scope | Exit criterion | Est. |
|---|---|---|---|
| **M0** Spike | module ↔ daemon ↔ stdio MCP; 3 tools (`get_world_info`, `query_documents`, `send_chat_message`) | Claude Desktop posts a chat message into a live world | 1 wk |
| **M1** Read core | generic query/get/`describe_schema`, resources, compendium search, scene + combat state, tool profiles, eval harness | An agent can answer any question about the world | 2–3 wk |
| **M2** Safe writes | document CRUD, dry-run, transaction log, undo, confirmation dialog, permission matrix, audit UI, kill switch | Nothing the agent does is unrecoverable | 3 wk |
| **M3** Table ops | tokens, canvas, walls/lights, combat control, dice, roll requests, chat, audio, journals; `dnd5e` adapter | A full combat encounter run end-to-end by voice-of-agent | 3 wk |
| **M4** Authoring | statblock import, encounter builder, quest/journal generation, RollTables, compendium import/export, macros | Session prep from notes to placed encounters | 3 wk |
| **M5** In-Foundry panel | AppV2 sidebar, BYO key, streaming, tool cards, threads, slash commands | No alt-tab needed to run a session | 3 wk |
| **M6** v1.0 | semantic memory, campaign log, `pf2e` adapter, i18n, `.mcpb` bundle, registry submission, docs site | Public release | 3 wk |
| **Post-1.0** | image/map generation, TTS/STT, hosted relay for mobile, player-facing restricted assistant, more adapters | — | — |

~18 weeks of focused part-time work to v1.0. M0–M2 (~7 weeks) is the point at which
it is genuinely useful and safe; consider that the real MVP.

---

## 17. Risks

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| R1 | Foundry v15 API churn breaks the module | High | Thin abstraction over Foundry APIs in `ops/`; compatibility matrix in CI; verified-version discipline |
| R2 | Game-system data divergence makes writes wrong | High | `describe_schema` first, adapters second, dry-run always; never guess `system.*` paths |
| R3 | Agent destroys world data | High | §8 in full — this is why M2 precedes M3 |
| R4 | Prompt injection via world content | Medium | §8.6; destructive ops always confirm |
| R5 | Tool-surface context bloat degrades accuracy | Medium | Profiles, projection, resources over tools, eval harness |
| R6 | GM tab must stay open | Medium | Documented up front; connection indicator; auto-reconnect |
| R7 | Undo fidelity gaps | Medium | Snapshots for large ops; explicit documented limits |
| R8 | Users confuse subscription (MCP) vs per-token (panel) cost | Medium | Explicit cost labelling in the panel UI |
| R9 | Licensing — shipping game content | Medium | Ship no content; import from the user's own compendia only |
| R10 | Concurrency: agent and human edit the same document | Low | Optimistic concurrency via `_stats.modifiedTime`; conflict → error with a diff |
| R11 | Token leakage in logs/screenshots | Low | Redaction in logging; token masked in the settings UI |

---

## 18. Notes & gotchas worth knowing before day one

- **Modules are client-side only.** There is no server-side hook to run this
  headlessly. Everything in §4.1 follows from that.
- **`socket: true` in `module.json`** is needed for the module's own
  GM↔player socket namespace (roll requests, showing handouts), which is separate
  from the external bridge WebSocket. Don't conflate them.
- **UUIDs, not IDs.** Always address documents by UUID (`Actor.xyz`,
  `Scene.abc.Token.def`, `Compendium.dnd5e.monsters.Actor.ghi`). It is the only
  identifier that survives being embedded, packed, or moved.
- **`Actor#getRollData()`** is the correct data context for evaluating formulas —
  don't hand-roll formula substitution.
- **Batch, don't loop.** `Actor.createDocuments([...])` and
  `scene.updateEmbeddedDocuments("Token", [...])` in one call, not N calls. Foundry
  broadcasts each call to every client; looping is the single biggest performance
  mistake in Foundry module code.
- **`ApplicationV2` + `HandlebarsApplicationMixin`** for all UI. AppV1 is deprecated in v13+.
- **Token vs Actor.** A Token on a scene may be linked (shares the Actor) or unlinked
  (has its own `delta`). Writing HP to the wrong one is the classic bug — always
  resolve through `token.actor`.
- **`CONFIG.statusEffects`** is the system-agnostic condition list; don't hardcode 5e conditions.
- **Roll modes** are set via `rollMode` in the ChatMessage data or
  `game.settings.get("core","rollMode")` — blind GM rolls need both the message
  flag and correct whisper targets.
- **Scene "activate" vs "view"** are different: activate pulls all players to the
  scene; view only moves the GM. Agents will get this wrong; make the tool
  parameter explicit and default to `view`.
- **Compendium packs are locked by default** — unlock, write, re-lock, and restore
  the prior lock state even on failure.
- **`_id` preservation on re-create** requires `keepId: true`; undo of a delete is
  wrong without it.
- **Foundry's "Data" path** is where snapshots and the index belong; reach it via
  `FilePicker`/`foundry.applications.apps.FilePicker` APIs, not `fs`.
- **Test against two systems from the start.** A world with only dnd5e installed
  will let generic-path bugs live until they're expensive to find.
- **Prior art is worth reading, not forking**: `adambdooley/foundry-vtt-mcp`
  (closest architecture — a WebSocket bridge module plus a stdio MCP server, 41
  tools), `TheStranjer/foundry-vtt-mcp` (direct-WebSocket, minimal),
  `laurigates/foundryvtt-mcp`. This spec's additions over all three are the
  transaction/undo layer, runtime schema discovery, the permission matrix, and the
  second front door.

---

## 19. Open decisions

These change the build materially and are worth settling before M1.

1. **Primary chat surface** — external MCP client only (cheaper, ships sooner), in-Foundry panel only, or both? *Recommendation: both, MCP first; the panel is M5 precisely because it can wait.*
2. **Systems for v1** — `generic` + dnd5e only, or pf2e from the start? *Recommendation: generic + dnd5e; pf2e at v1.0.*
3. **Autonomy default** — is `assist` (confirm destructive) the right out-of-box posture? *Recommendation: yes, with a first-run wizard that explains the modes.*
4. **Escape hatch** — ship `execute_script` at all? *Recommendation: yes, expert-only and opt-in; without it "anything a GM can do" is untrue.*
5. **Distribution** — free MIT forever, or open-core with paid convenience (installers, hosted relay, map generation)? Affects whether project infrastructure exists at all.
6. **Map/image generation** — in scope, or an integration point for ComfyUI/other? *Recommendation: integration point, post-1.0.*
