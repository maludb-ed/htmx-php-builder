---
name: chat-actions
description: The voice-first chat command bar that ships on every screen of every application — natural language in (typically dictated via a tool like Wispr Flow), application actions and navigation out, powered by an LLM router activating MCP tools. Use when building or extending the assistant command bar, the actions MCP server, the action manifest, chat-driven navigation, or voice-to-data capture.
---

# Chat Actions — voice-to-data and voice-to-navigation

Every application ships a persistent chat command bar. The user says (or types) what they want — "record three sets of bench press at 45 pounds", "navigate to the screen to create a new exercise" — and the assistant does it, using **both the utterance and the context of the screen they're on**. Input is chat-shaped but voice-first: users dictate via tools like Wispr Flow, so utterances are conversational, unpunctuated, and reference the visible screen ("add ten pounds to that").

## Architecture: LLM router → MCP tools → the app's own endpoints

```
Every screen: fixed bottom command bar (#assistant-bar)
   │ hx-post /assistant/message.php  (+ screen context)
   ▼
PHP handler (session auth, CSRF, activity log; mints a signed action token)
   ▼
Unified assistant service — Python, Claude Agent SDK (the SAME agent as AMA)
   │ LLM processes the utterance and routes it by activating an MCP tool
   ▼
{app}_actions_mcp  (Python + FastMCP, localhost-only)      + the two read servers
   ├─ navigate(screen, …)      → resolves via the screen registry, returns a directive
   ├─ {entity}_{action}(…)     → POSTs to the app's own PHP endpoints (action token)
   └─ undo_last(…)             → inverse action from the manifest + activity log
```

- **One unified assistant.** The AMA page and the command bar are two surfaces of the same Agent SDK service: read tools (record/activity MCP servers), action tools, and navigation. One conversation memory per user; questions, actions, and navigation mix freely in a single utterance.
- **The router is the LLM; the tools are MCP.** The agent never navigates or writes by itself — it activates a tool on the actions MCP server. Tool descriptions carry the routing knowledge (which screen/action serves which intent), so routing quality is a Phase 1 design artifact, not prompt luck.
- **Actions go through the app's own controllers.** The actions server's tools call the same `public/{feature}/save.php`-style endpoints a human uses — same validation, same authorization, same activity logging. The **read MCP servers stay strictly read-only**; nothing ever writes around the PHP layer.
- **The actions server is localhost-only** — unlike the client-facing read servers, it acts *as a user* and is reachable only by the assistant service. Per-request authority comes from a short-lived HMAC-signed action token minted by the PHP handler (user id + expiry); internal endpoints accept it in place of the session and enforce the same authorization.

## The action manifest (designed in Phase 1)

The sibling of the MCP tool surface: a designed registry, not an afterthought.

- **Screen registry:** every screen's stable id, canonical URL, title, and a one-line "when the user wants…" description. This is what `navigate` resolves against — the navigation router's routing table.
- **Action registry:** every performable action — name (`exercise_record_set`), target endpoint, parameter schema, natural-language trigger description, **undo definition** (the inverse: delete the created row / restore prior values), and whether it may execute immediately or must confirm.
- Both derive from the Phase 0 question/feature list. A screen or action missing from the manifest is unreachable by voice — treat that as unfinished design, exactly like a question with no MCP tool.

## Confirmation policy (locked): act + undo

- **Creates and updates execute immediately.** The reply states what was done in the interpreted terms ("Logged 3 × bench press @ 45 lbs") with an **Undo** control; "undo that" by voice resolves the last action from the conversation and activity log.
- **Destructive actions always confirm first** (`hx-confirm` on the rendered confirm control, or a spoken "yes" turn).
- **Ambiguity asks, briefly.** If the utterance can't be resolved against the manifest + screen context ("which exercise?"), the assistant asks one short question rather than guessing.

## How results reach the browser (pure HTMX — no new machinery)

| Assistant outcome | Response mechanism |
|---|---|
| Navigation | `HX-Location: {"path": "<canonical URL>", "target": "#page-content"}` — the shell swaps and pushes the explicit URL (never `hx-push-url="true"`) |
| Data action | Reply bubble + `HX-Trigger: {entity}Changed` — screens listening per Pattern D refresh their own regions; the assistant knows nothing about their markup |
| Answer (AMA-style) | Reply bubble in the command bar thread (full conversations live on the AMA page) |

## The command bar UI (locked design)

- **Fixed bottom bar** on every screen, rendered by the shell: `#assistant-bar` with input `#assistant-input`, send `#assistant-send-btn`. Slim, always visible, above the footer; full-width and thumb-reachable on mobile; never overlaps `#page-content` scroll (body gets bottom padding).
- The latest exchange renders in `#assistant-reply` expanding upward from the bar; full history in the offcanvas transcript `#assistant-transcript` (full-width on mobile). No modals.
- `Ctrl/Cmd+K` (and `/` outside inputs) focuses the input — dictation tools type into the focused field, so focus-fast matters more than any mic button. Enter submits; the input clears and refocuses for the next utterance.
- **Screen context travels with every message:** each screen partial stamps `#page-content` with `data-screen`, `data-entity`, `data-record-id`; the bar submits them via `hx-vals` (js) alongside the text. This is how "that" resolves.

## Voice-first behaviors (bake into the assistant's system prompt)

- Parse dictation tolerantly: spelled-out numbers ("forty five"), units ("pounds", "lbs"), missing punctuation, filler ("uh, log, um, three sets").
- Always echo the interpretation in the reply — the confirmation *is* the readback.
- Support correction verbs as first-class: "undo that", "make it 50", "no, dumbbell press".
- Keep replies to one short sentence; this surface is a command bar, not a chat page.
- Latency matters: run this route at low effort, and prefer the project's fastest acceptable model for router-only turns (configurable in one place, like AMA's model).

## Build order integration

- **Phase 1:** design the action manifest (screen + action registries, undo definitions, confirm flags) alongside the schema and MCP tool surface; checkpoint covers all three.
- **Phase 2:** the shell ships the command bar (ids above) wired to a stub handler, so the surface exists from the first screen.
- **Phase 3:** each vertical slice registers its screens and actions in the manifest as part of "done".
- **Phase 4:** build the actions MCP server + unified assistant service; verify with a spoken-style test script: every manifest action and every screen reachable by a one-sentence utterance.
