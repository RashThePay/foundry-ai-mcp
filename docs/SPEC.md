# Foundry AI MCP — Scope, Requirements & Specification

**Project:** `foundry-ai-mcp`
**Status:** Draft v0.2 (pre-implementation)
**Target platform:** Foundry Virtual Tabletop v14 (LTS) — minimum and verified. v13 unsupported.
**Target game system:** D&D 5e only.
**Primary clients:** Claude Cowork and ChatGPT (Work/Business, Developer mode) — **cloud-hosted agents**.
**License:** MIT, permanently. No open-core, no paid tier, no project-operated infrastructure.

---

## 1. Executive summary

An open-source system that lets a GM run a D&D 5e campaign in Foundry VTT by
talking to their own AI agent in Claude Cowork or ChatGPT.

1. **The Module** (`foundry-ai-mcp`) — a Foundry v14 add-on that runs in a GM
   browser session and executes the full GM capability surface behind a safe,
   audited, undoable operation layer.
2. **The Server** (`@rashthepay/foundry-mcp`) — a self-hosted service exposing a
   **public HTTPS MCP endpoint** that Cowork and ChatGPT connect to, bridged over
   WebSocket to the module.

### The constraint that defines the architecture

Claude custom connectors — on claude.ai, Claude Desktop, **and Cowork** — connect
to your MCP server **from Anthropic's cloud infrastructure, not from your
machine**. Local stdio servers configured in `claude_desktop_config.json` are
[not available in Cowork or claude.ai](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp).
ChatGPT's Developer mode likewise takes a remote HTTPS MCP URL.

> **Therefore: a localhost stdio MCP server cannot serve this project's stated goal.**
> The server must be publicly reachable over HTTPS and authenticated.

This inverts the usual Foundry-MCP design (every existing project is
localhost/stdio) and drives §4, §7 and §8. It is the single most consequential
decision in this document.

### Positioning

| | Familiar | adambdooley/foundry-vtt-mcp | **foundry-ai-mcp** |
|---|---|---|---|
| Where the agent runs | In Foundry | Local Claude Desktop | **Cowork / ChatGPT, from anywhere** |
| Transport | Vendor / OpenAI-compatible API | stdio, localhost | **Public HTTPS MCP + OAuth** |
| Cost | Paid subscription | Free | **Free, MIT forever** |
| Write scope | Curated GM actions | ~41 curated tools | **Full document/canvas surface + escape hatch** |
| Safety | Vendor-defined | Read-only toggle | **Mode matrix, dry-run, snapshots, transaction log, undo** |
| Images | Generation built in | ComfyUI integration | **Upload from URL/HTTP/bytes; no generation** |
| Systems | Several | 6 systems | **D&D 5e** |

**Differentiators worth building for:** (a) it works from a cloud agent at all,
(b) transactional undo, (c) runtime schema discovery, (d) an explicit autonomy
model designed for *unattended* operation.

---

## 2. Goals and non-goals

### Goals

- **G1** — Anything a GM can do through the Foundry UI, the agent can do through a tool call.
- **G2** — Runs from Claude Cowork or ChatGPT with no local client software beyond a browser.
- **G3** — The GM's own AI account and self-hosted infrastructure. Nothing routed through project-operated services.
- **G4** — No irreversible surprise. Every mutation is diffable, logged, snapshot-protected and undoable.
- **G5** — The agent can put images into the world — sourced from the internet, generated in its own sandbox, or supplied by the GM.
- **G6** — Deployable by a technically comfortable hobbyist in under an hour, with a `docker compose up` path.
- **G7** — MIT forever.

### Non-goals

| Not doing | Why |
|---|---|
| **Image/map generation** | The agent generates images itself if it can; this project only ingests them. |
| **PF2e or other systems** | D&D 5e only. Keep a thin adapter seam, ship one implementation. |
| **In-Foundry chat panel** | Deferred. The chat app *is* the interface; a panel would duplicate it and add per-token billing confusion. See §11. |
| **Player-facing assistants** | Post-1.0. |
| **Running with no GM browser session** | Architecturally impossible — but see the optional Agent Seat (§4.5). |
| **Shipping game content** | Licensing. Import from the GM's own compendia only. |
| **Any hosted service, account system, or telemetry** | G3, G7. |
| **Voice I/O** | Post-1.0. |

---

## 3. Personas & user stories

**P1 — Prep GM (async, away from the table).** Cowork or ChatGPT on a laptop or phone.
- "Read my Session 12 notes and build the three encounters I sketched, place them on the Ruined Chapel scene, and stat the cult leader as a CR 5 caster."
- "Find a good portrait for Lady Velkaris, upload it, and set it as her actor art and token image."
- "Every NPC in my 'Waterdeep' journal folder without an actor — create one and link it."

**P2 — Live GM (mid-session).** Chat app on a second monitor or tablet, Foundry on the main screen.
- "The party set the tavern on fire. Give me a d20 consequence table and roll on it."
- "Bandit 3 dashes to the balcony and shoots the wizard — resolve it."
- "Everyone rolls a DC 15 Dex save."

**P3 — World builder.** Bulk data operations and hygiene passes.

---

## 4. Architecture

### 4.1 The two binding constraints

1. **Foundry's API is client-side.** `game.actors`, `canvas`, `Document`, `Roll` exist only in the browser; modules do not run on the Foundry server. **Every write must be executed by a browser session authenticated as a GM.**
2. **The agent is in someone else's cloud.** It cannot reach localhost. **The server must be publicly reachable over HTTPS.**

Together these mean the system has three moving parts that must all be up: the
cloud agent, a public HTTPS server, and a live GM browser session.

### 4.2 Components

```
┌──────────────────────────────┐
│  Claude Cowork / ChatGPT     │   agent runs in Anthropic's or
│  (cloud)                     │   OpenAI's cloud
└──────────────┬───────────────┘
               │  HTTPS · Streamable HTTP MCP · OAuth 2.1
               ▼
┌──────────────────────────────────────────────────────────┐
│  foundry-mcp  (self-hosted: VPS, home server, or tunnel) │
│   ├─ MCP endpoint      /mcp      tools/resources/prompts │
│   ├─ OAuth endpoints   /.well-known/*  /authorize /token │
│   ├─ Asset intake      /assets/upload   (agent curl)     │
│   ├─ Asset staging     /assets/stage/:token  (module GET)│
│   └─ Bridge hub        /bridge   (WSS ← module)          │
└──────────────┬───────────────────────────────────────────┘
               │  WSS + bridge token   (module dials out)
               ▼
┌──────────────────────────────────────────────────────────┐
│  foundry-ai-mcp module — in a GM browser session         │
│   bridge client · op router · safety · txn log · 5e      │
│                       ▼                                   │
│         Foundry VTT v14 client API (game.*, canvas.*)     │
└──────────────────────────────────────────────────────────┘
```

The browser dials out because browsers cannot accept inbound connections. The
server is therefore a WebSocket *host* and an HTTPS *host*, and needs no inbound
access to the GM's machine at all.

### 4.3 Where the server runs

It must be reachable inbound from the agent cloud and outbound-reachable from the
GM's browser. It does **not** need to be on the GM's machine.

| Option | Best for | Notes |
|---|---|---|
| **Alongside self-hosted Foundry** (same VPS, subdomain) | Self-hosters | Cleanest. Real TLS cert, always up, reuse existing reverse proxy. **Recommended.** |
| **Small VPS** ($5/mo) | Forge / Molten users | Foundry stays hosted; server runs separately. |
| **Home server + Cloudflare Tunnel / Tailscale Funnel** | No VPS | Free, no port forwarding, TLS terminated by the tunnel. |
| Localhost + ngrok | Development only | Ephemeral URL breaks the saved connector. |

Anthropic publishes [egress IP ranges](https://platform.claude.com/docs/en/api/ip-addresses) for firewall allowlisting — worth using, but not a substitute for auth.

### 4.4 Transports

- **Streamable HTTP** at `/mcp` — the primary transport, required by both target clients.
- **stdio** — retained as a secondary mode (`foundry-mcp --stdio`) purely for local development and Claude Code. Not the shipping path.

### 4.5 The Agent Seat (optional, and the thing that makes P1 actually work)

A GM browser session must be live for any operation to succeed. Requiring the GM
to leave a tab open on a machine that is awake defeats "prep from my phone."

The optional **Agent Seat** is a headless Chromium container (Playwright) that logs
into the world as a dedicated GM user (`AI Agent`) and loads the module, keeping
the bridge permanently up. Shipped as an opt-in service in the `docker compose`
file next to the server.

- Foundry licenses per server, not per user, so a dedicated GM user costs nothing.
- ~400 MB RAM. Auto-restarts, re-authenticates, and reports health via `world.health_check`.
- Actions appear in Foundry as that user, which makes the audit trail *better* — every agent change is attributable.
- **Caveats:** the world must be running; some hosted providers restrict automation in their terms — check before pointing it at Forge; a second GM session is a second vector if the server is compromised.

Without the Agent Seat the system still works — the GM just needs a Foundry tab open,
which is the normal case for P2 (live play) and fine for P1 at a desk.

### 4.6 One capability core

The module holds a single registry of operations with zod-validated inputs; the
server generates MCP tool definitions from it. The registry seam is kept so a
future in-Foundry panel (§11) or a second protocol can reuse it without a rewrite,
but only one consumer exists in v1.

---

## 5. Capability surface

Everything below is in scope for v1 unless marked.

### 5.1 World documents (CRUD)

Actor, Item, Scene, JournalEntry, RollTable, Playlist, Macro, Cards, Combat,
ChatMessage, Folder, User, Setting; embedded: ActiveEffect, Token, Tile, Wall,
AmbientLight, AmbientSound, Drawing, MeasuredTemplate, Note, Region (v14's
reworked regions and region behaviours), Combatant, JournalEntryPage, TableResult,
PlaylistSound, and Items owned by Actors.

Create, read, update (deep-merge or replace), delete, duplicate, move between
folders, reorder, set ownership, flags — with batch variants of all.

### 5.2 Compendium packs

List packs, full-text and filtered search, index reads, import into the world,
export to a pack, create/delete world compendiums, lock/unlock, pack folders.

### 5.3 Scene & canvas

Create/clone/delete scenes; background, foreground, dimensions, padding, grid,
initial view; activate vs. view; navigation order. Lighting: darkness, global
illumination, colours, fog exploration, vision. Weather. **v14 features in scope:**
multi-level scenes (level definitions, per-level placement, elevation binding) and
shared fog of war.

Place and edit walls (door type/state, sense restrictions, thresholds, direction),
lights, sounds, tiles, drawings, notes, regions with behaviours, measured templates.

Runtime: pan/zoom, ping, select/target tokens, toggle visibility and elevation,
move tokens (animated or teleport), reset/reveal fog, door states, pull players to a scene.

### 5.4 Combat

Create encounters; add/remove combatants; roll initiative (single, group, all,
formula override); reorder; set turn; advance/step back turn and round; end
combat. Read live state (order, HP, conditions, resources, targets). Apply damage
and healing with 5e type/resistance handling. Toggle conditions. Award XP.

### 5.5 Dice

Any `Roll` formula with proper `getRollData()` context; roll modes (public / GM /
blind / self); roll as a speaker; roll tables with and without replacement; nested
tables. **Request rolls from players** — post an actionable chat card to named
users, then poll for the result (§7.5).

### 5.6 Chat

Post as GM, as an actor (IC), OOC, emote; whisper; rich HTML and enriched
`@UUID[...]` links; roll messages; read history with filters; delete.

### 5.7 Journals, tables, handouts

Multi-page journals (text, image, PDF, video pages); append and edit; reorder;
show page or image to players; create and roll RollTables; Cards decks.

### 5.8 Audio

List playlists and tracks; play, stop, pause, loop, crossfade; channel and track
volume; one-shots; scene ambient sounds.

### 5.9 Macros

List, create, edit, delete (script and chat); execute with arguments; hotbar assignment.

### 5.10 Users & settings

List users (role, online, assigned character); assign character; document
ownership levels; read/write module and system settings.

### 5.11 Assets & images ⭐ new in v0.2

The agent cannot generate images — but it must be able to **get images into the
world**, from three sources:

| Source | Path | Notes |
|---|---|---|
| **Found on the internet** | `assets.upload_image({ source: "url", url })` — server fetches | Cheapest. **The primary path.** |
| **Generated by the agent** | Agent `POST`s bytes to `/assets/upload`, gets a handle, passes it to the tool | Cowork and ChatGPT sandboxes have shell/network; `curl` is the intended route. |
| **Supplied by the GM** | GM drops files in a watched directory, or uploads via Foundry normally | `assets.list_files` makes them addressable. |
| *(fallback)* Inline bytes | `source: "base64"`, hard-capped at 256 KB | See the token-cost warning in §18 — this is for icons, not maps. |

Operations: upload, list directory, create directory, move, delete, and **bind** an
uploaded asset to its use — actor portrait, prototype token art, token art on a
specific token, scene background/foreground, tile texture, journal image page,
item icon, note icon.

**Transfer mechanism.** Bridge frames cap at 1 MiB, so images are not sent over the
WebSocket. Instead: the server acquires the bytes, stages them under a single-use
token, and tells the module to `fetch()` the staging URL directly over HTTPS. The
module turns the response into a `File` and calls Foundry's `FilePicker.upload()`.
No chunking, and the browser does the transfer it is good at.

**Hard requirements:**
- **SSRF defence** — the model chooses the URL. Block private, loopback, link-local and cloud-metadata ranges (`169.254.169.254` especially); resolve DNS and re-check before connecting; follow at most 3 redirects and re-check each.
- **Path confinement** — uploads confined to a configurable root (default `foundry-ai-mcp/` under Foundry's Data dir). Reject `..`, absolute paths, and anything resolving outside the root.
- **Content validation** — verify magic bytes, not just the extension or `Content-Type`. Allowlist `png/jpg/webp/gif/webm/mp4/ogg`. **SVG denied by default** (it is script-capable and Foundry renders it inline).
- **Size cap** — default 20 MB, configurable.
- **Mixed content** — Foundry is served over HTTPS in every realistic deployment, so the staging URL must be HTTPS too, or the browser blocks the fetch.

### 5.12 Escape hatch (expert mode, opt-in)

`system.execute_script` (arbitrary JS in the GM client, serialised result and
console capture) and `system.call_api` (reflective invocation of a dotted path).
These make "anything a GM can do" literally true and cover 5e module interop no
curated tool anticipates. Off by default, logged verbatim.

### 5.13 Meta

Transaction log, undo, snapshot/restore, autonomy mode, capability introspection,
health check (including whether a GM session is actually connected), kill switch.

---

## 6. Tool design

### 6.1 Three layers

A one-tool-per-capability mapping of §5 is 150–200 tools, which wrecks selection
accuracy and burns context every turn. Instead:

1. **Ergonomic tools (~42)** — the high-frequency GM path, tight schemas, good descriptions.
2. **Generic document tools (6)** — `query_documents`, `get_document`, `describe_schema`, `create_documents`, `update_documents`, `delete_documents`. These reach anything the ergonomic tools don't.
3. **Escape hatch (2)** — §5.12, opt-in.

**Tool profiles** — a setting selects what is advertised: `core` (~20) ·
`standard` (~42, default) · `full` (~58) · `expert` (+escape hatch), with
`notifications/tools/list_changed` on change.

**Runtime schema discovery.** `describe_schema({ type: "Actor", subtype: "npc" })`
returns JSON Schema derived from the live `DataModel.schema` plus the system
template, with real example values. Even single-system this earns its place: dnd5e's
data model shifts across its own major versions, and this keeps the agent correct
without the project chasing every release.

### 6.2 Tool catalog

`†` mutating · `‡` expert-only.

**world** — `get_world_info`, `get_capabilities`, `health_check`

**documents** — `query_documents`, `get_document`, `describe_schema`,
`create_documents`†, `update_documents`†, `delete_documents`†, `duplicate_document`†, `manage_folders`†

**actors** — `list_actors`, `get_actor`, `create_actor_from_statblock`†,
`modify_actor_resource`†, `manage_actor_items`†, `apply_effect`†, `set_ownership`†

**compendium** — `list_packs`, `search_compendium`, `import_from_compendium`†, `export_to_compendium`†

**scenes** — `list_scenes`, `get_scene`, `create_scene`†, `configure_scene`†, `activate_scene`†

**canvas** — `place_objects`†, `update_placeables`†, `delete_placeables`†,
`edit_walls`†, `move_token`†, `set_token_state`†, `control_camera`†, `manage_fog`†, `set_door_state`†

**combat** — `get_combat_state`, `start_combat`†, `manage_combatants`†,
`roll_initiative`†, `advance_combat`†, `end_combat`†, `apply_damage`†, `toggle_condition`†

**dice** — `roll_dice`†, `roll_table`†, `request_player_roll`†, `await_event`

**chat** — `send_chat_message`†, `get_chat_history`, `delete_chat_messages`†

**journal** — `create_journal`†, `update_journal_page`†, `show_to_players`†, `search_journals`

**assets** ⭐ — `upload_image`†, `list_files`, `manage_files`†, `set_artwork`†

**audio** — `list_audio`, `control_audio`†

**macros** — `list_macros`, `create_macro`†, `execute_macro`†

**users** — `list_users`, `manage_settings`†

**system** ‡ — `execute_script`†, `call_api`†

**session** — `get_transaction_log`, `undo`†, `create_snapshot`†, `restore_snapshot`†, `set_autonomy_mode`†

### 6.3 Conventions

Mutating tools take `dry_run` (compute and return the diff, apply nothing),
`reason` (one line, shown in confirmations and the audit log) and `txn_group`
(bundle calls into one undoable transaction). Read tools take `limit` (default 25),
`cursor` and `fields` (projection — essential; a 5e Actor can exceed 200 KB).

```jsonc
{
  "ok": true,
  "data": { },
  "txn_id": "txn_01J...",
  "affected": [{ "uuid": "...", "name": "...", "op": "update" }],
  "warnings": [],
  "truncated": false
}
```

Typed errors, each with an actionable `hint`: `NOT_FOUND`, `PERMISSION_DENIED`,
`NO_GM_SESSION`, `CONFIRMATION_TIMEOUT`, `CONFIRMATION_DENIED`, `VALIDATION_FAILED`,
`RATE_LIMITED`, `READONLY_MODE`, `PAYLOAD_TOO_LARGE`, `ASSET_REJECTED`, `UNSAFE_URL`.

`NO_GM_SESSION` matters more than it looks — with a cloud agent it is the most
common failure, and its hint must say plainly that a Foundry GM tab (or the Agent
Seat) needs to be running.

### 6.4 MCP resources

`foundry://world/summary` · `foundry://scene/current` · `foundry://combat/current` ·
`foundry://actors/index` · `foundry://journal/index` · `foundry://uuid/{uuid}`

### 6.5 MCP prompts

`prep_session`, `run_combat_round`, `improvise_npc`, `session_recap`,
`statblock_from_text`, `build_encounter`, `dress_the_scene`, `audit_world`.

---

## 7. Protocols

### 7.1 Agent ⇄ server (MCP over HTTPS)

- **Transport:** Streamable HTTP at `POST /mcp`. HTTP+SSE fallback for older clients.
- **Auth:** OAuth 2.1 with PKCE and dynamic client registration per the MCP authorization spec, advertised at `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource`. Claude custom connectors expect OAuth; ChatGPT Developer mode also accepts unauthenticated servers — **do not offer that**, the endpoint has GM rights over a live world.
  - Single-operator design: one admin user, credentials set at deploy time, no user database.
  - A static bearer token is supported as a documented escape hatch for clients that cannot do OAuth.
- **Hardening:** TLS required; `Origin` validation and DNS-rebinding protection; per-token rate limits; structured request logging with token redaction; optional allowlist of Anthropic/OpenAI egress ranges.

**Implementation note:** OAuth 2.1 + DCR is a meaningful chunk of work — budget it
explicitly in M1 rather than discovering it late. It is the price of the goal.

### 7.2 Module ⇄ server (bridge over WSS)

```jsonc
{
  "v": 1,
  "id": "01J8XY...",
  "type": "req" | "res" | "event" | "hello" | "ping" | "pong",
  "op": "documents.update",
  "payload": { },
  "meta": { "dryRun": false, "txnGroup": null, "ts": 0 }
}
```

Handshake: module reads a bridge token from module settings (generated on first
run, shown with a copy button), connects to `wss://<host>/bridge`, sends `hello`
with `{ token, worldId, systemId, foundryVersion, moduleVersion, userId, userRole }`.
Server does a constant-time compare, rejects non-GM roles, and negotiates protocol
version — mismatches produce a clear in-Foundry notification, not a silent failure.

Heartbeat every 15s, 45s timeout, reconnect with jittered exponential backoff
(1s → 30s, unlimited). Connection state is a coloured dot in the Foundry sidebar.

Frames cap at 1 MiB; oversized results are stored under a `result_ref` and paged
via `session.fetch_result`. Images bypass this entirely (§5.11).

### 7.3 Multiple GM sessions

If both a human GM and the Agent Seat are connected, operations route to a single
designated executor (Agent Seat preferred when present) to avoid double-applying.
Others receive events read-only.

### 7.4 Events

`combat.turn_changed`, `combat.started`, `combat.ended`, `roll.completed`,
`chat.message`, `token.moved`, `scene.activated`, `document.changed`,
`confirmation.resolved`, `user.connected`.

Buffered in a bounded ring and read via `dice.await_event` with a cursor.

**Constraint worth stating plainly:** over HTTPS through a cloud agent, long-polling
is unreliable — proxies and gateways cut idle connections around 30–60s, and the
agent only runs when it has the turn. Cap waits at 25s and return a cursor for
resumption. Tight agentic loops ("run the goblins, then wait for the players to
roll") work far better in a local setup than a cloud one. Async prep — the P1
use case — is unaffected. Set expectations for P2 accordingly.

---

## 8. Safety, permissions, audit, undo

An agent with GM rights can delete a campaign, and here it is reachable from the
public internet. This section is a requirement, not a disclaimer.

### 8.1 Autonomy modes

| Mode | Reads | Reversible writes | Destructive writes | Scripts |
|---|---|---|---|---|
| `readonly` | ✅ | ❌ | ❌ | ❌ |
| `assist` **(default)** | ✅ | ✅ auto | ⚠️ confirm in Foundry | ❌ |
| `unattended` | ✅ | ✅ auto | ✅ auto + **forced snapshot first** | ❌ |
| `expert` | ✅ | ✅ | ✅ | ⚠️ confirm |

"Destructive" = deletes, ownership changes, mass updates past the threshold
(default 25 documents), scene activation, world setting writes.

**`unattended` exists because of the cloud-agent model.** In `assist`, a
destructive op raises a dialog in the Foundry client — but during P1 prep from a
phone, nobody is looking at that screen, and the op dies on a 60-second timeout.
`unattended` swaps *prior confirmation* for *guaranteed reversibility*: an
automatic snapshot of the affected collections before the operation, plus the
normal transaction log. The GM chooses which trade they want, per session.

### 8.2 Per-tool permission matrix

A settings grid: every tool × {allow, confirm, deny}. Modes seed it; any cell can
be overridden. Persisted per world.

### 8.3 Confirmation UX (`assist`)

Non-blocking dialog: tool name, the agent's `reason`, a field-level diff, affected
document names and count, and Approve / Approve-for-session / Deny. 60s timeout
defaults to **deny**, and the resulting error tells the agent a human wasn't
present so it can suggest `unattended`.

### 8.4 Transaction log & undo

Each mutating op records its inverse before applying:

| Forward | Inverse |
|---|---|
| create | delete by `_id` |
| update | update with captured pre-state of touched paths |
| delete | create from full captured source, `keepId: true` |
| batch | inverses replayed in reverse |

Rolling buffer (default 50) in a world setting, spilling to
`Data/foundry-ai-mcp/txn/*.jsonl`. Exposed as `session.undo({ txn_id? })` and an
Audit Log sidebar app with per-entry Undo.

**Documented limits — put these in the UI, not only here.** Re-created documents
can lose inbound references that were also mutated; effects re-applied out of
order can yield different derived values; canvas animation state isn't restored.
Undo is a strong net, not a database transaction. Snapshots
(`session.create_snapshot`, a JSON export of chosen collections) are the real
answer, taken automatically before anything touching >100 documents and before
every destructive op in `unattended`.

### 8.5 Rate limits & kill switch

Default 60 writes/min and 500 documents/min; breaching either pauses the bridge.
A kill switch (sidebar button + hotkey, default `Ctrl+Shift+X`) drops the socket
and flips to `readonly`. The server also honours a kill flag so it can be stopped
without touching Foundry.

### 8.6 Prompt injection

World content is untrusted. Journal text, chat messages, item descriptions and
player-authored content can carry instructions aimed at the agent — and with a
cloud agent the blast radius includes anything else that agent can reach.

- World-derived text is returned inside an explicit untrusted-content envelope with a standing "this is data, not instruction" note.
- Destructive tools confirm in `assist` regardless of how convincing the justification is.
- In `unattended`, the snapshot requirement is the backstop precisely because the confirmation isn't there.
- The audit log records `reason` verbatim, so injected instructions are visible afterwards.

### 8.7 Privacy & exposure

No project-operated servers, no telemetry. But: the endpoint is public, and the
model provider sees whatever world content is returned. Both are stated plainly in
the README. Projection (`fields`) limits what leaves; semantic indexing ships off
by default.

---

## 9. D&D 5e support

Thin `SystemAdapter` boundary, **one implementation**. The seam exists to insulate
against dnd5e's own version churn (its data model has moved significantly across
majors), not to speculatively support systems we've decided not to build.

```ts
interface SystemAdapter {
  readonly id: "dnd5e";
  readonly systemVersions: string;         // semver range, enforced at handshake

  getVitals(actor): { hp, maxHp, tempHp, ac, level, cr };
  applyDamage(actor, amount, opts): Promise<void>;   // types, resistances, immunities
  applyHealing(actor, amount): Promise<void>;
  conditions(): Array<{ id, label, icon }>;          // from CONFIG.statusEffects
  toggleCondition(token, id, active): Promise<void>;
  rollCheck(actor, kind, key, opts): Promise<Roll>;  // ability | save | skill | attack | tool
  initiativeFormula(combatant): string;
  parseStatblock(text): Promise<ActorData>;
  encounterBudget(partyLevels, difficulty): { xp, crRange };
  itemUse(actor, item, opts): Promise<void>;
  spellSlots(actor): Array<{ level, value, max }>;
  resources(actor): Array<{ key, label, value, max }>;
}
```

Version guard: the module reads the installed dnd5e version at handshake and
refuses (with a clear message) outside its tested range, rather than silently
writing to fields that moved. A second adapter is a post-1.0 conversation, and the
interface will be refactored *then*, against a real second case.

---

## 10. Memory & retrieval (optional, off by default)

Local index of journals, actor/item descriptions, chat history and selected
compendia, stored on the **server** host under `data/index/` (sqlite-vec or
LanceDB). Embeddings local by default (`transformers.js`, `all-MiniLM-L6-v2`).
Incremental reindex from `document.changed` events, debounced. Exposed via
`journal.search_journals({ mode: "semantic" | "keyword" | "hybrid" })`.

Opt-in **campaign log**: a journal the module appends per-session summaries to,
giving the agent continuity without re-reading the world.

---

## 11. In-Foundry chat panel — deferred

**Not in v1.** Rationale, recorded so it isn't relitigated: the chat app is
already the interface, a panel would duplicate it, and it would introduce
per-token API billing alongside a subscription-billed path — the most likely
source of user confusion in the whole design. The capability registry (§4.6)
keeps the seam open if this is ever wanted.

---

## 12. Non-functional requirements

| Area | Requirement |
|---|---|
| Foundry | v14 LTS only — `minimum: "14"`, `verified: "14"` (stable 14.367). ApplicationV2 UI only |
| System | dnd5e, version range enforced at handshake |
| Browsers | Chromium 120+, Foundry Electron app; headless Chromium for the Agent Seat |
| Node | 22 LTS+ for the server (its own process; Foundry v14's own runtime is Node 24) |
| Transport | Streamable HTTP MCP; TLS required; OAuth 2.1 + PKCE + DCR |
| Latency | p95 < 800 ms for reads end-to-end from the agent (cloud round-trip included) |
| Payloads | 25-item default page; 1 MiB frames; 20 MB asset cap |
| Availability | Server and Agent Seat auto-restart; module reconnects indefinitely |
| Footprint | Module bundle < 500 KB gzipped; no runtime CDN fetches; server < 200 MB RSS without the index |
| i18n | All strings in `lang/en.json` |
| Accessibility | Keyboard-navigable dialogs, Foundry theme support, `prefers-reduced-motion` |
| Logging | Structured, level-configurable, redacts tokens and OAuth secrets |

---

## 13. Repository layout & stack

```
foundry-ai-mcp/
├─ packages/
│  ├─ shared/            # protocol types + zod schemas (single source of truth)
│  ├─ module/            # Foundry v14 module — TS, bundled with Vite
│  │  ├─ module.json
│  │  └─ src/
│  │     ├─ main.ts
│  │     ├─ bridge/      # wss client, handshake, reconnect, framing
│  │     ├─ ops/         # one handler per namespace (§6.2)
│  │     ├─ assets/      # staged fetch → File → FilePicker.upload
│  │     ├─ safety/      # modes, matrix, confirm dialog, rate limits
│  │     ├─ txn/         # inverse ops, undo, snapshots
│  │     ├─ dnd5e/       # the one adapter
│  │     ├─ schema/      # describe_schema introspection
│  │     └─ ui/          # settings, audit log, connection indicator (AppV2)
│  ├─ server/            # MCP over HTTPS + bridge hub
│  │  └─ src/
│  │     ├─ http.ts  mcp.ts  oauth.ts  bridge.ts  stdio.ts
│  │     ├─ assets/    # intake, SSRF guard, validation, staging
│  │     ├─ tools/ resources/ prompts/
│  │     └─ memory/
│  └─ agent-seat/        # optional headless Chromium GM session
├─ deploy/               # docker-compose, Caddy/Traefik, tunnel guides
├─ docs/  tests/  .github/workflows/
```

**Stack:** TypeScript · zod (one schema source → MCP JSON Schema + runtime
validation) · `@modelcontextprotocol/sdk` · `ws` · Vite (module) · tsup (server) ·
Playwright (Agent Seat + E2E) · vitest · Quench (in-Foundry tests) · changesets.

`module.json`: `id`, `title`, `description`, `version`, `authors`,
`compatibility: { minimum: "14", verified: "14" }`, `esmodules`, `styles`,
`languages`, `socket: true`, `url`, `manifest`, `download`, `relationships`
(dnd5e), `flags`. `socket: true` is for the module's *own* Foundry socket namespace
(roll requests, showing handouts) — a different thing from the bridge WebSocket.

---

## 14. Testing & CI

- **Unit (vitest)** — op handlers against a mocked `game`/`canvas`; inverse-op correctness as a property test (`apply(inverse(apply(op))) == identity`); SSRF guard against a table of hostile URLs; asset magic-byte validation; protocol framing.
- **Integration** — server ↔ fake module client; full tool round-trips; OAuth flow; error taxonomy; rate limits; reconnection.
- **In-Foundry (Quench)** — real Document CRUD, undo fidelity, `FilePicker.upload`, dnd5e adapter behaviour.
- **E2E (Playwright)** — Dockerised Foundry v14 + dnd5e; the Agent Seat is itself Playwright, so E2E reuses it. Drive a scripted session, assert world state. Gated on a licence secret, skipped on forks.
- **Security** — an SSRF/path-traversal/upload suite as a first-class gate, not a nice-to-have. This project accepts model-chosen URLs and writes files to disk from a public endpoint.
- **Golden-path evals** — a fixture world plus ~30 natural-language tasks scored on tool selection and end state. Build it in M1; it is the only defence against tool-description regressions.
- **CI** — lint, typecheck, unit, integration, security suite per PR; E2E nightly; releases build `module.zip` + `module.json` and publish the npm package and a container image.

---

## 15. Distribution

1. **Module** — GitHub Release with `module.zip` and a stable `module.json` URL; Foundry package registry submission once past v0.5.
2. **Server** — npm `@rashthepay/foundry-mcp` and a published container image.
3. **`deploy/docker-compose.yml`** — server + optional Agent Seat + Caddy for automatic TLS. The one-command path, and the one most users should take.
4. **Docs** — a guide per deployment option (§4.3), connector setup for Cowork and for ChatGPT Developer mode, a security checklist, and a tool reference generated from the zod schemas.

Success criterion: a GM comfortable with Docker is issuing their first command
within an hour, including TLS and OAuth setup.

MIT, permanently. No paid tier, no hosted option, no CLA.

---

## 16. Roadmap

| Milestone | Scope | Exit criterion | Est. |
|---|---|---|---|
| **M0** Spike | module ⇄ bridge ⇄ HTTPS MCP over a tunnel; 3 tools; bearer auth only | Cowork posts a chat message into a live world | 1–2 wk |
| **M1** Transport & reads | OAuth 2.1 + PKCE + DCR, hardening, generic query/get/`describe_schema`, resources, compendium search, scene + combat state, tool profiles, eval harness | The agent can answer any question about the world, over a real connector | 3–4 wk |
| **M2** Safe writes | document CRUD, dry-run, transaction log, undo, snapshots, confirmation dialog, permission matrix, `unattended` mode, audit UI, kill switch | Nothing the agent does is unrecoverable | 3 wk |
| **M3** Table ops | tokens, canvas, walls/lights, multi-level scenes, combat, dice, roll requests, chat, audio, journals; dnd5e adapter | A full encounter run end-to-end from the chat app | 3 wk |
| **M4** Assets | URL fetch, `/assets/upload`, staging + `FilePicker.upload`, SSRF and validation suite, artwork binding | "Find a portrait for this NPC and set it" works | 2 wk |
| **M5** Agent Seat | headless Chromium GM session, compose file, health and auto-restart | Prep from a phone with no tab open | 1–2 wk |
| **M6** v1.0 | semantic memory, campaign log, statblock import, encounter builder, macros, i18n, docs site, registry submission | Public release | 3 wk |
| Post-1.0 | player-facing restricted assistant, voice, second system adapter, in-Foundry panel | — | — |

~17–20 weeks part-time. **M0–M2 (~8 weeks) is the real MVP** — useful and safe.
M4 and M5 are what make the P1 phone-prep story actually land, and both are small.

---

## 17. Risks

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| R1 | Public endpoint with GM rights is compromised | **High** | OAuth 2.1, TLS, rate limits, egress allowlist, `readonly` default on fresh installs, security checklist in docs, kill switch |
| R2 | SSRF / malicious upload via model-chosen URLs | **High** | §5.11 guards; dedicated security test suite in CI |
| R3 | Agent destroys world data with nobody watching | **High** | §8 in full; forced snapshots in `unattended`; M2 precedes M3 |
| R4 | Prompt injection from world content, amplified by a cloud agent's other reach | **High** | §8.6; untrusted-content envelopes; destructive ops gated |
| R5 | Connector platform changes (transport, auth, tool limits) break the integration | Medium | Both target clients supported; stdio retained; transport isolated behind one module |
| R6 | No GM session connected — the most common runtime failure | Medium | Agent Seat; `NO_GM_SESSION` with an actionable hint; connection indicator |
| R7 | dnd5e data model churn | Medium | `describe_schema` first; version range enforced at handshake |
| R8 | Foundry v15 API churn | Medium | Thin abstraction in `ops/`; v14 LTS baseline buys the longest runway |
| R9 | Cloud round-trip latency makes live play sluggish | Medium | Projection, resources over tools, batching; set expectations for P2 (§7.4) |
| R10 | Undo fidelity gaps | Medium | Snapshots for large ops; limits documented in the UI |
| R11 | Base64 image ingestion is prohibitively expensive in tokens | Low | URL and HTTP-upload paths are primary; base64 capped at 256 KB with a warning |
| R12 | Hosted Foundry providers restrict automation (Agent Seat) | Low | Opt-in; check terms; system works without it |
| R13 | Agent and human edit the same document concurrently | Low | Optimistic concurrency on `_stats.modifiedTime`; conflict returns a diff |

---

## 18. Notes & gotchas

- **Modules are client-side only.** No server-side hook, no headless mode except a real browser. Everything in §4.1 follows.
- **Cowork cannot use local MCP servers.** `claude_desktop_config.json` servers are unavailable in Cowork and claude.ai; connectors are brokered from Anthropic's cloud even when the client app runs on your machine. Any design starting from "stdio on localhost" fails the goal on day one.
- **v14 only; carry no v13 shims.** v13 and v14 need mutually exclusive Node versions (v13 won't run on Node 24, v14 requires it), so nobody straddles both. This is a constraint on the *Foundry server's* runtime — the MCP server picks its own Node.
- **v14 reworked scenes, regions, effects and fog.** Multi-level scenes, the new regions system, shared fog of war and revised effects handling are v14-era. Verify class and field names against the 14.367 API docs; several moved namespaces, and there is a lot of v13-era Foundry code on the internet that a model will happily imitate.
- **Base64 images through a tool call are a trap.** A 1024×1024 PNG is ~1.5 MB, ~2 MB base64, on the order of 500k tokens. It will not fit and it would be absurdly expensive if it did. `curl` to `/assets/upload`, or a URL. The 256 KB cap on the inline path exists to make the failure early and obvious rather than expensive.
- **The browser can't fetch arbitrary internet images** — CORS blocks most hosts. That's why the server fetches and stages, and the module only ever fetches from the server's own origin (§5.11).
- **`FilePicker` upload** needs the `FILES_UPLOAD` permission (GMs have it) and lives at `foundry.applications.apps.FilePicker` in the v13+ namespacing — confirm against v14 before writing it.
- **SVG uploads are an XSS vector** in Foundry, which renders them inline. Denied by default.
- **UUIDs, not IDs.** `Actor.xyz`, `Scene.abc.Token.def`, `Compendium.dnd5e.monsters.Actor.ghi` — the only identifier that survives embedding, packing or moving.
- **`Actor#getRollData()`** is the right context for formula evaluation. Don't hand-roll substitution.
- **Batch, don't loop.** `Actor.createDocuments([...])`, `scene.updateEmbeddedDocuments("Token", [...])` — one call, not N. Foundry broadcasts every call to every client; looping is the classic Foundry performance bug.
- **Token vs Actor.** A token is linked (shares the Actor) or unlinked (has its own delta). Writing HP to the wrong one is *the* classic bug — always resolve via `token.actor`.
- **`CONFIG.statusEffects`** is the condition source of truth even in a 5e-only build; dnd5e populates it and modules extend it.
- **Scene activate ≠ view.** Activate pulls every player to the scene; view moves only the current user. Agents will get this wrong — make the parameter explicit and default to `view`.
- **Compendium packs are locked by default.** Unlock, write, re-lock — and restore the prior lock state on failure too.
- **`keepId: true`** is required when re-creating a deleted document, or undo is silently wrong.
- **Two GM sessions need an elected executor** (§7.3), or every write applies twice.
- **Prior art worth reading, not forking:** `adambdooley/foundry-vtt-mcp` (closest architecture, but localhost/stdio — the part this project can't reuse), `TheStranjer/foundry-vtt-mcp`, `laurigates/foundryvtt-mcp`.

---

## 19. Decisions settled

| # | Decision |
|---|---|
| 1 | **Clients:** Claude Cowork and ChatGPT. Cloud agents ⇒ public HTTPS MCP + OAuth. stdio kept for local dev only. |
| 2 | **System:** D&D 5e only. Thin adapter seam, one implementation, no speculative registry. |
| 3 | **Licence:** MIT forever. No open-core, no hosted service, no project infrastructure. |
| 4 | **In-Foundry panel:** deferred (§11). |
| 5 | **Images:** ingest only — URL, HTTP upload, or capped inline bytes. No generation tool. |
| 6 | **Autonomy default:** `assist`, with `unattended` for away-from-desk prep. Fresh installs start `readonly`. |
| 7 | **Escape hatch:** ship `execute_script`, expert-only and opt-in. |

### Still open

- **Agent Seat in v1 or post-1.0?** Speced as M5 because it's small and it's what makes phone-prep real, but it is the one component that could slip without breaking anything else.
- **Multi-world support** — one server instance per world, or one server brokering several? One-per-world is simpler and assumed throughout; revisit if you run more than one campaign.
