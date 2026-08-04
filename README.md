# htmx-php-builder

A Claude plugin for building memory-first HTMX + PHP + Bootstrap 5.3 applications with a consistent look and feel, a fixed build order, and an Ask Me Anything feature in every app.

## Installation

In Claude Code:

```
/plugin marketplace add maludb-ed/htmx-php-builder
/plugin install htmx-php-builder@maludb-ed
```

The repository is its own marketplace (see `.claude-plugin/marketplace.json`), with the plugin at the repository root.

## Stack

Ubuntu 24.04 · PostgreSQL 17 + MaluDB (activity memory) · two MCP servers per app (Python + FastMCP; Node.js as alternative) · Claude Agent SDK AMA service (Python) · Apache · vanilla PHP · Bootstrap 5.3 (nxl admin template) · HTMX. Mobile-first; no modal forms or display pages (`hx-confirm` guards destructive actions). Memories are served through MCP — the app's AMA agent and the client's own AI tools both read them via authenticated MCP endpoints (SaaS Plus+).

## Skills

| Skill | Purpose |
|---|---|
| `new-app` | The phased build workflow: memory-first planning → full database + memory design → auth + app shell → vertical feature slices → Ask Me Anything |
| `new-screen` | Generates each screen from the design system so consistency holds screen-to-screen |
| `mcp-servers` | The two MCP servers every app ships (record + activity memory): tool-surface design, FastMCP implementation, deployment, client-facing auth |
| `design-system` | The mandatory UI template distilled into layout skeletons, component patterns, and mobile rules |
| `php-patterns` | The mandatory PHP code structure (readable/maintainable for humans and agents) |
| `memory-first-planning` | Memory-first design, ask-me-anything, SaaS Plus+, the two-memory architecture |
| `php-session-auth` | Security rules for vanilla PHP session auth + CSRF, Google sign-in, TOTP 2FA |
| `malumail-send` | Transactional email via the MaluMail API — the default email channel for every app |

## Build order (enforced by new-app)

1. **Phase 0** — memory-first planning (memory model, question list, feature order)
2. **Phase 1** — database design for the ENTIRE application + activity-log/MaluDB design + the MCP tool surface (logging exists from day one)
3. **Phase 2** — authentication + application shell (after this phase the app looks exactly as intended, on desktop and at 375px mobile)
4. **Phase 3** — vertical feature slices, one at a time
5. **Phase 4** — the two MCP servers, the Ask Me Anything agent (Claude Agent SDK) consuming them, and the client-facing MCP endpoints

## Requirements for the memory/agent layer

Python 3 with `mcp` (FastMCP) and `claude-agent-sdk`, an `ANTHROPIC_API_KEY` in the AMA service environment, and read-only PostgreSQL/MaluDB roles for the MCP servers. The fallback PHP-only AMA variant needs `composer require "anthropic-ai/sdk"`.
