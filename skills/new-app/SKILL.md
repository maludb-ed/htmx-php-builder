---
name: new-app
description: Start building a new HTMX + PHP + Bootstrap 5.3 application. Use when the user says "new app", "start a new application", "build an app", "begin the project", or invokes /new-app. Enforces the fixed build order — full database + memory design first, then authentication + app shell, then vertical feature slices — on the standard stack (Ubuntu 24.04, PostgreSQL 17 + MaluDB, Apache, PHP, HTMX).
---

# New Application Workflow

Build every application in the same fixed order. Do not skip phases, reorder them, or start writing feature screens before Phase 2 is approved by the user. Each phase ends with a user checkpoint.

Before starting, load context from the sibling skills in this plugin:

- **memory-first-planning** — memory-first design, ask-me-anything, SaaS Plus+, the two-memory architecture served through MCP. Consult during Phase 0 and 1.
- **mcp-servers** — the two MCP servers every app ships (record + activity memory). Consult during Phase 1 (tool-surface design) and Phase 4.
- **design-system** — the mandatory UI template and rules. Consult during Phase 2 and every screen thereafter.
- **php-patterns** — the mandatory PHP code structure. Consult before writing any PHP.
- **php-session-auth** — security rules for the auth build in Phase 2.
- **malumail-send** — the default email channel (MaluMail API) for reset/verification emails and notifications. Consult in Phase 2 and wherever a feature sends mail.

## The stack (fixed — do not reopen unless the user does)

| Layer | Choice |
|---|---|
| OS | Ubuntu 24.04 |
| System of record | PostgreSQL 17 |
| Activity memory | MaluDB (PostgreSQL extensions), fed by application logs |
| Memory interface | Two MCP servers per app (Python + FastMCP), per the mcp-servers skill |
| AMA agent | Claude Agent SDK service (Python), consuming the MCP servers |
| Web server | Apache (also reverse-proxies the MCP servers and AMA service) |
| Application | PHP (vanilla, per php-patterns skill) |
| UI | Bootstrap 5.3 per the design-system skill |
| Interactivity | HTMX partial updates |

Node.js is the documented alternative for the MCP/agent layers; default to Python, one runtime per project.

## Phase 0 — Planning (memory-first)

Plan the application with the user using the memory-first-planning skill: decide what the application *remembers* before what it *does*. Deliverables:

1. The memory model — entities, facts, relationships.
2. The question list — the record questions (answered from PostgreSQL) and activity questions (answered from MaluDB logs) the application exists to answer. Features are pre-packaged answers to these questions.
3. The feature list, ordered for Phase 3 vertical slices.

**Checkpoint:** user approves the memory model and feature order.

## Phase 1 — Database and memory design (the ENTIRE application)

Design the complete database up front — every table for every planned feature, not just the first slice.

1. Write one SQL file per concern under `db/` (e.g. `db/001_auth.sql`, `db/002_core.sql`, …) targeting PostgreSQL 17. The auth schema always includes the Google-identity and TOTP-2FA structures from the php-session-auth skill's references (`auth_identities`, nullable `password_hash`, `totp_*` columns, recovery codes) — they ship in every app even if a project enables the features later.
2. Design the **activity log** from day one: a log table (and/or file stream) capturing every screen entered, record created, and action taken by every actor, in the shape MaluDB ingests. Activity memory cannot be backfilled — logging exists before the first feature ships.
3. Include the MaluDB extension setup and ingestion wiring.
4. Every table gets created_at/updated_at and, where rows are user-owned, the owning user id. Use `bigint generated always as identity` primary keys unless the user specifies otherwise.
5. **Design the MCP tool surface** (per the mcp-servers skill): map every question from the Phase 0 question list to a named tool on the record or activity server, plus one guarded read-only search tool per server. Define the read-only database roles the servers will use. The question list is the contract — a question with no tool is unfinished design.

**Checkpoint:** user approves the full schema *and* the MCP tool surface before any PHP is written.

## Phase 2 — Authentication and application shell

Goal: at the end of this phase the application *looks and feels exactly like the finished product*, with working login, even though it has no features yet.

1. Build session auth (register, login, logout, reset) in vanilla PHP following the **php-session-auth** skill — all of its non-negotiables apply. The login page offers email/password **and** "Sign in with Google" (server-side OIDC per `references/google-signin.md`); authenticator-app 2FA (per `references/totp-2fa.md`) ships with its enrollment settings page and login challenge page. CSRF is wired for HTMX from the first shell render (meta tag + `htmx:configRequest` listener). Reset and verification emails go through the **malumail-send** skill's `malumail_send()` helper. Auth pages use the design-system minimal auth layout; login/logout/2FA are full page navigations, never swaps.
2. Build the application shell from the **design-system** skill: the sidebar/header/footer layout, the HTMX content-swap target, an empty dashboard, and the navigation stubs for every planned feature.
3. Wire activity logging into the shell from the first request: page entry, login/logout events.
4. Verify on a mobile viewport (375px): navigation collapses correctly, no horizontal scroll, no modals anywhere.

**Checkpoint:** user reviews the running shell (desktop and mobile) and confirms the look and feel before Phase 3.

## Phase 3 — Vertical feature slices

Build one feature at a time, end-to-end, in the order fixed in Phase 0. Each slice includes:

1. The screens (list / detail / create / edit as needed) generated per the **new-screen** skill — dedicated full pages, never modals.
2. The PHP handlers per the **php-patterns** skill, against the Phase 1 schema (schema changes at this point are exceptional and need user sign-off).
3. Activity logging for every action in the slice.
4. A mobile-viewport check before calling the slice done.

**Checkpoint per slice:** demo the feature, then move to the next.

## Phase 4 — MCP servers and Ask Me Anything

1. **Build the two MCP servers** designed in Phase 1 (record memory + activity memory) per the mcp-servers skill: Python + FastMCP, read-only roles, systemd units, Apache reverse proxy, per-client bearer tokens. Verify every Phase 0 question is answerable through the tools using MCP Inspector before moving on.
2. **Build the AMA feature** per [references/ama-implementation.md](references/ama-implementation.md): a Claude Agent SDK service (Python) consuming the two MCP servers, fronted by a PHP/HTMX chat page. The agent reads memories only through MCP.
3. **Publish the client-facing endpoints**: a settings screen listing the two MCP URLs and managing access tokens, so clients can connect their own AI tools to their memories (SaaS Plus+).

## Hard rules that apply in every phase

- **No modal forms or display pages.** No Bootstrap `.modal` for anything that shows or collects data. Create/edit/detail are dedicated full pages loaded via HTMX; quick edits may expand inline; offcanvas drawers only for filters/quick views and full-width on mobile. Confirmations are exempt: `hx-confirm` guards destructive actions.
- **Mobile first.** Every screen must work at 375px width with no horizontal scroll before it is done.
- **Log everything.** Any handler that changes state or renders a screen writes an activity log row.
- **Consistency over creativity.** Never invent new UI patterns; copy the canonical markup from the design-system skill.
