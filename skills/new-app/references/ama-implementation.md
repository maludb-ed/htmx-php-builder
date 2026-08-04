# Ask Me Anything — reference implementation

Every application ships an AMA screen: natural language in, answers out. Record questions ("which item sold the most last week?") are answered from PostgreSQL, activity questions ("when did Bob start planning for Oktoberfest?") from MaluDB — both **through the application's two MCP servers** (see the mcp-servers skill). The AMA agent never queries the databases directly; if AMA needs data the MCP servers can't provide, that's a gap in the MCP tool surface to fix first.

## Architecture (default: Claude Agent SDK service)

```
Browser (HTMX chat page — dedicated full page, never a modal/widget)
   │  POST /ama/ask  (hx-post → rendered message partial)
   ▼
PHP handler (thin proxy: session auth, CSRF, activity logging)
   │  HTTP to localhost
   ▼
AMA service — Python, Claude Agent SDK (systemd unit)
   │  agent loop, sessions, MCP client — all built in
   ▼
{app}_records_mcp          {app}_activity_mcp
   (PostgreSQL 17)             (MaluDB)
```

- The AMA service is a small Python web service (FastAPI or similar) wrapping the **Claude Agent SDK** (`pip install claude-agent-sdk`). The SDK supplies the agentic loop, multi-turn session handling, and MCP client handling; the service adds only: an endpoint per chat turn, per-user session mapping, and the system prompt.
- The service's agent options declare the two MCP servers (HTTP transport, service-account token) and restrict tools to those servers' tools — no filesystem or bash tools in a deployed AMA agent.
- The system prompt states who the current user is, what the application is, and the answer style (concise, cite which memory the answer came from). Build it per app from the Phase 0 planning output.
- Conversation state: the SDK manages session transcripts; the PHP side stores only `(user_id → session_id)` so a reload resumes the conversation.
- Model: Claude Opus (current `claude-opus-5`), configured in one place.
- **Verify SDK specifics against the Agent SDK docs when building** (`code.claude.com/docs/en/agent-sdk`) — options shape and MCP config keys evolve; do not code them from memory.

### PHP side

```php
// ama/ask.php — validate session + CSRF, log the question to the activity log,
// then proxy to the AMA service and render the answer as an HTMX partial.
$resp = http_post_json('http://127.0.0.1:8765/ask', [
    'user_id'    => $user['id'],
    'session_id' => $_SESSION['ama_session_id'] ?? null,
    'question'   => $question,
]);
$_SESSION['ama_session_id'] = $resp['session_id'];
render_partial('ama/message', ['question' => $question, 'answer' => $resp['answer']]);
```

The AMA service binds to localhost only; Apache does not expose it. The PHP handler is the sole caller and carries the user identity — the service trusts it because both live on the same host.

### Deployment

- systemd unit alongside the two MCP server units; Python venv under the app root.
- Requires `ANTHROPIC_API_KEY` in the service's environment (never in PHP or the browser).
- Every question, tool call, and answer is written to the activity log — AMA usage is itself activity memory.

## UI

- Dedicated full page in the sidebar navigation, built from the design-system chat pattern.
- `hx-post` appends to the message thread with an HTMX indicator while waiting; long answers may stream later, but v1 renders the completed answer partial.
- Works identically at 375px — the thread is a single column; no popovers or floating widgets.

## Fallback: PHP + MCP connector (no Python service)

For deployments where the extra service is unwanted, keep AMA in vanilla PHP calling the Messages API with the MCP connector — the same MCP servers do the work:

```php
$response = $client->beta->messages->create(
    model: AMA_MODEL,
    maxTokens: 4096,
    system: $amaSystemPrompt,
    mcpServers: [
        ['type' => 'url', 'name' => 'records',  'url' => $recordsMcpUrl,  'authorization_token' => $svcToken],
        ['type' => 'url', 'name' => 'activity', 'url' => $activityMcpUrl, 'authorization_token' => $svcToken],
    ],
    tools: [
        ['type' => 'mcp_toolset', 'mcp_server_name' => 'records'],
        ['type' => 'mcp_toolset', 'mcp_server_name' => 'activity'],
    ],
    betas: ['mcp-client-2025-11-20'],
    messages: $messages,
);
```

Requires the MCP servers to be reachable from Anthropic's API (i.e. the client-facing HTTPS endpoints, not localhost). Check `stopReason` before reading `content` (`refusal` renders a polite "can't answer that" partial); every `mcp_toolset` must reference a declared server by name. The PHP app stores conversation history itself in this variant.

Choose the Agent SDK service by default; use the fallback when a project explicitly wants zero Python beyond the MCP servers.
