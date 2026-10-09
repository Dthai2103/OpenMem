# OpenMem

OpenMem is a customized OpenViking source checkout used as the MCP memory layer
for LibreFang.

This README only covers how to start the OpenViking MCP server from this repo
and how a remote LibreFang machine should connect to it.

## Requirements

- Python 3.10+
- `uv`
- A configured model provider for OpenViking
- Optional local setup: Ollama running locally

For the current local setup, Ollama is used for embedding, VLM, and query
planning.

## Install From Source

```bash
git clone https://github.com/Dthai2103/OpenMem.git
cd OpenMem
uv sync
```

## Configure The Server

Run the setup wizard:

```bash
uv run openviking-server init
```

Recommended choices for local development:

```text
Embedding:     ollama
VLM:           litellm + ollama model
Query planner: litellm + ollama model
Server host:   127.0.0.1 for local only, or 0.0.0.0 for another machine
Port:          1933
Auth:          api_key for remote access
```

The wizard writes server config to:

```text
~/.openviking/ov.conf
```

If using Ollama, make sure it is running:

```bash
ollama serve
```

Then validate the setup:

```bash
uv run openviking-server doctor
```

## Start The MCP Server

Start OpenViking from this source checkout:

```bash
uv run openviking-server
```

Keep this terminal open. The server does not hot-reload source changes, so after
editing code restart the process:

```bash
# stop the running server with Ctrl+C, then start again
uv run openviking-server
```

## Verify Locally

Health check:

```bash
curl http://127.0.0.1:1933/health
```

Expected response shape:

```json
{
  "status": "ok",
  "healthy": true,
  "version": "...",
  "auth_mode": "api_key"
}
```

The MCP endpoint is:

```text
http://127.0.0.1:1933/mcp
```

If auth is enabled, send a user/admin API key, not the root key:

```bash
export OPENVIKING_API_KEY="<USER_OR_ADMIN_API_KEY>"
```

List MCP tools:

```bash
curl -sS -X POST http://127.0.0.1:1933/mcp \
  -H "X-API-Key: $OPENVIKING_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

Important LibreFang tools should include:

```text
workspace_context
workspace_session_commit
workspace_memory_search
workspace_memory_write
```

## Connect From Another Machine

When LibreFang runs on another machine:

1. Start OpenViking with server host `0.0.0.0`.
2. Make sure port `1933` is reachable from the LibreFang machine.
3. Use this MCP URL on the LibreFang machine:

```text
http://<OPENVIKING_MACHINE_IP>:1933/mcp
```

Example LibreFang MCP config:

```json
{
  "mcpServers": {
    "openviking": {
      "type": "streamable_http",
      "url": "http://<OPENVIKING_MACHINE_IP>:1933/mcp",
      "headers": {
        "X-API-Key": "<USER_OR_ADMIN_API_KEY>"
      }
    }
  }
}
```

Do not use the root API key for LibreFang memory operations. Root keys are for
administration; LibreFang should use a user/admin API key.

## LibreFang Memory Flow

Before a task, LibreFang asks OpenViking for context:

```json
{
  "tool": "workspace_context",
  "arguments": {
    "workspace_id": "default",
    "project_id": "openviking-librefang",
    "user_id": "dthai",
    "agent_id": "coder",
    "query": "task hiện tại user muốn làm gì",
    "include_native": true,
    "detail": "overview"
  }
}
```

After a task, LibreFang sends the session/task history to OpenViking:

```json
{
  "tool": "workspace_session_commit",
  "arguments": {
    "workspace_id": "default",
    "project_id": "openviking-librefang",
    "session_id": "lf-task-001",
    "user_id": "dthai",
    "agent_id": "coder",
    "task_id": "task-001",
    "messages": [
      {
        "role": "user",
        "content": "Nội dung user yêu cầu",
        "message_kind": "user_query"
      },
      {
        "role": "assistant",
        "content": "Agent đã làm gì / trả lời gì",
        "message_kind": "assistant_step"
      }
    ],
    "task_history": {
      "status": "success",
      "summary": "Task đã hoàn thành"
    }
  }
}
```

Main rule:

```text
Start task -> workspace_context
End task   -> workspace_session_commit
```

`workspace_memory_write` is only for explicit/manual memory writes, such as when
a user or admin says something must be remembered exactly.
