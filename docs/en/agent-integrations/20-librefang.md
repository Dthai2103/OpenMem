# LibreFang MCP Memory Layer

LibreFang can use OpenViking as a durable workspace memory layer through MCP.
LibreFang remains the agent runtime; OpenViking stores and retrieves shared
workspace, project, user, agent, resource, and skill context.

## Connection

OpenViking exposes MCP over streamable HTTP:

```text
http://127.0.0.1:1933/mcp
```

For the current two-machine LibreFang setup, OpenViking runs on the OpenViking
machine and LibreFang runs on a different machine. LibreFang does not need the
OpenViking source tree. It only needs the remote MCP URL and API key.

Current OpenViking machine:

```text
Host:      192.168.88.204
Port:      1933
Health:    http://192.168.88.204:1933/health
MCP URL:   http://192.168.88.204:1933/mcp
Auth mode: api_key
```

OpenViking is started on that machine from the customized source tree:

```bash
cd /Users/dthai2103/Desktop/OpenViking
uv run openviking-server
```

The server config lives on the OpenViking machine at:

```text
/Users/dthai2103/.openviking/ov.conf
```

The relevant server config is:

```json
{
  "server": {
    "host": "0.0.0.0",
    "port": 1933,
    "auth_mode": "api_key",
    "root_api_key": "ov_root_vGOq1As9s1dfxveciPjq6eKBnQhCG4y5glX9r6heqTo"
  }
}
```

LibreFang runs on another machine. It should not start OpenViking and does not
need the OpenViking source code. It only connects to the MCP URL below.

LibreFang should use this key for MCP and data API calls:

```text
X-API-Key: bGlicmVmYW5n.bGlicmVmYW5nLWFkbWlu.ZDA2NWE0YTY4NWM4MmFlZmIwY2E4MjYzZTkyMzBkOTY1ZWI3NWNhNWE4MmE4YjM2Y2Y2Nzg4ZmQ1MWRkMDE0ZQ
```

The same key can also be sent as:

```text
Authorization: Bearer bGlicmVmYW5n.bGlicmVmYW5nLWFkbWlu.ZDA2NWE0YTY4NWM4MmFlZmIwY2E4MjYzZTkyMzBkOTY1ZWI3NWNhNWE4MmE4YjM2Y2Y2Nzg4ZmQ1MWRkMDE0ZQ
```

The current root key is only for OpenViking administration, such as creating
accounts/users. Do not configure LibreFang with the root key for normal memory
operations:

```text
Root admin key: ov_root_vGOq1As9s1dfxveciPjq6eKBnQhCG4y5glX9r6heqTo
```

From the LibreFang machine, first test network reachability:

```bash
curl http://192.168.88.204:1933/health
```

Then test authenticated OpenViking access:

```bash
curl 'http://192.168.88.204:1933/api/v1/fs/ls?uri=viking://' \
  -H 'X-API-Key: bGlicmVmYW5n.bGlicmVmYW5nLWFkbWlu.ZDA2NWE0YTY4NWM4MmFlZmIwY2E4MjYzZTkyMzBkOTY1ZWI3NWNhNWE4MmE4YjM2Y2Y2Nzg4ZmQ1MWRkMDE0ZQ'
```

If this curl fails from the LibreFang machine, fix network/firewall/VPN first.
MCP will not work until the LibreFang machine can reach the OpenViking server.

Example LibreFang MCP client configuration when remote streamable HTTP MCP is
supported:

```json
{
  "mcpServers": {
    "openviking": {
      "type": "streamable_http",
      "url": "http://192.168.88.204:1933/mcp",
      "headers": {
        "X-API-Key": "bGlicmVmYW5n.bGlicmVmYW5nLWFkbWlu.ZDA2NWE0YTY4NWM4MmFlZmIwY2E4MjYzZTkyMzBkOTY1ZWI3NWNhNWE4MmE4YjM2Y2Y2Nzg4ZmQ1MWRkMDE0ZQ"
      }
    }
  }
}
```

If the LibreFang deployment only supports stdio MCP servers, run the OpenViking
stdio proxy on the LibreFang machine. The proxy is only a bridge from stdio to
the remote OpenViking HTTP MCP endpoint; it is not the OpenViking server and does
not need to store memory locally:

```json
{
  "mcpServers": {
    "openviking": {
      "type": "stdio",
      "command": "node",
      "args": ["/path/to/OpenViking/agent-plugins/servers/mcp-proxy.mjs"]
    }
  }
}
```

The proxy resolves the OpenViking server URL and API key from the same local
configuration sources as the OpenViking CLI. For an explicit local connection,
set:

```bash
export OPENVIKING_URL="http://192.168.88.204:1933"
export OPENVIKING_API_KEY="bGlicmVmYW5n.bGlicmVmYW5nLWFkbWlu.ZDA2NWE0YTY4NWM4MmFlZmIwY2E4MjYzZTkyMzBkOTY1ZWI3NWNhNWE4MmE4YjM2Y2Y2Nzg4ZmQ1MWRkMDE0ZQ"
```

When OpenViking authentication is enabled, use a User or Admin API key. Root
keys are for administration and cannot read or write tenant memory through MCP.

## Workspace Namespace

The LibreFang-oriented MCP tools use a workspace/project namespace. In
OpenViking storage, the namespace is physically rooted under the authenticated
OpenViking user because API-key mode resolves data ownership from the key. For
LibreFang, treat that authenticated user as the **LibreFang service account**,
not as the end user.

Current deployment:

```text
OpenViking account:        librefang
OpenViking service user:   librefang-admin
LibreFang memory root:     viking://user/librefang-admin/
```

So the top node in LibreFang diagrams should be read as:

```text
LibreFang Memory Root
backed by viking://user/librefang-admin/
```

End-user memory belongs inside the workspace tree under
`memories/users/{user_id}`. Do not confuse the OpenViking service user
`librefang-admin` with LibreFang application users such as `dthai`.

The logical workspace/project namespace is:

```text
viking://~/workspaces/{workspace_id}/projects/{project_id}/
```

The expanded canonical form is:

```text
viking://user/{service_user_id}/workspaces/{workspace_id}/projects/{project_id}/
```

Workspace identifiers use ASCII letters, digits, `_`, and `-`, start with a
letter or digit, and are limited to 64 characters.

Recommended initial identifiers:

```text
workspace_id = "default"
project_id   = a stable LibreFang project slug
user_id      = LibreFang's stable user/account id
agent_id     = LibreFang's stable agent role id, such as planner, coder, reviewer
```

Memory scopes:

```text
viking://~/workspaces/{workspace_id}/projects/{project_id}/memories/project/
viking://~/workspaces/{workspace_id}/projects/{project_id}/memories/users/{user_id}/
viking://~/workspaces/{workspace_id}/projects/{project_id}/memories/agents/{agent_id}/
```

Optional project-local resource and skill document roots:

```text
viking://~/workspaces/{workspace_id}/projects/{project_id}/resources/
viking://~/workspaces/{workspace_id}/projects/{project_id}/skills/
```

Use `add_skill` and `viking://~/skills` or `viking://agent/skills` for
first-class OpenViking skill lifecycle management. The project-local `skills`
root above is for lightweight project documents that should be retrieved with
the workspace context tools.

## Native-First Mental Model

LibreFang can use OpenViking in two complementary ways:

```text
Native-first flow:
  LibreFang task/session finishes
    -> LibreFang sends conversation + task history to workspace_session_commit
    -> OpenViking stores the native session transcript
    -> OpenViking commit runs native extraction/compression
    -> OpenViking can produce user memories, peer project memories,
       trajectories, cases, and experiences according to native policy
    -> Future LibreFang tasks call workspace_context(include_native=true)

Curated memory flow:
  LibreFang extracts an explicit durable memory candidate
    -> LibreFang chooses scope and memory_type
    -> LibreFang calls workspace_memory_write
    -> OpenViking stores the approved memory in the custom workspace tree
```

Use the native-first flow for full OpenViking leverage: session lifecycle,
session capture, compression, native extraction, peer-based project memory,
and agent evolution experiences. Use the curated flow when LibreFang has
already decided on a precise durable memory to store.

Native project memory is represented as a peer:

```text
peer_id = workspace-{workspace_id}__project-{project_id}
```

`company_id` is intentionally not part of the normal flow because the current
deployment has only one company. The MCP tool still accepts it as an advanced
optional field for future multi-company expansion, but LibreFang should omit it
today.

The current deployment uses the `librefang-admin` OpenViking user as the
LibreFang service account. Native `self` memories therefore belong to that
OpenViking user unless LibreFang later provisions separate OpenViking users/API
keys for each application user. For now, pass LibreFang `user_id` and `agent_id`
so they are preserved as tags and task metadata.

## Curated Memory Model

LibreFang decides what should become an explicit curated memory. OpenViking
validates, stores, indexes, and retrieves it.

```text
LibreFang task/session finishes
  -> LibreFang extracts durable candidates
  -> LibreFang chooses scope and memory_type
  -> LibreFang calls workspace_memory_write once with the full memory body
  -> OpenViking stores the full body as L2
  -> OpenViking indexing derives L0/L1 retrieval views
  -> Future LibreFang tasks call workspace_context or workspace_memory_search
```

Do not write the same memory three times for L0, L1, and L2. Write once with the
full content. Choose the detail level only when reading.

## Memory Model

### Scope

`scope` answers who should reuse the memory:

| Scope | Owner field | Use for |
|-------|-------------|---------|
| `project` | none | Shared project facts, conventions, architecture decisions, task knowledge |
| `user` | `user_id` | Stable user preferences inside this workspace/project |
| `agent` | `agent_id` | Lessons or workflows useful to one LibreFang agent role |

### Memory Type

`memory_type` answers what kind of durable knowledge is being stored:

| Type | Use for |
|------|---------|
| `fact` | Stable factual knowledge |
| `decision` | Architecture/product/process decisions and their outcome |
| `preference` | User or team preference that should affect future behavior |
| `instruction` | A reusable rule, policy, or workflow instruction |
| `experience` | A self-improvement lesson learned from a task outcome |
| `reflection` | A higher-level observation that may guide future planning |

Recommended mapping:

```text
project architecture choice  -> scope=project, memory_type=decision
user likes concise answers   -> scope=user,    memory_type=preference
coder learned debug pattern  -> scope=agent,   memory_type=experience
shared project workflow      -> scope=project, memory_type=instruction
```

### L0/L1/L2 Detail Levels

The detail levels are read-time views of the same memory:

| Detail | Layer | Meaning | Use when |
|--------|-------|---------|----------|
| `abstract` | L0 | Smallest summary | Routing, quick ranking, low-token scans |
| `overview` | L1 | Practical summary | Default task-start context |
| `full` | L2 | Full markdown body | Debugging, exact evidence, deep inspection |

Write flow:

```text
workspace_memory_write(content="full durable memory...")
```

Read flow:

```text
workspace_context(detail="overview")       # default for task start
workspace_memory_search(detail="abstract") # cheap scan
workspace_memory_search(detail="full")     # inspect selected hits
```

## MCP Tools

LibreFang should call these as MCP tools against the `openviking` MCP server.
The examples below show the tool name and the JSON arguments LibreFang should
send.

### workspace_session_commit

Commits a completed LibreFang task/session into OpenViking's native session
lifecycle. This is the recommended default after a task finishes.

Tool name:

```text
workspace_session_commit
```

Arguments:

```json
{
  "workspace_id": "default",
  "project_id": "openviking-librefang",
  "session_id": "lf-session-2026-10-08-001",
  "user_id": "dthai",
  "agent_id": "coder",
  "task_id": "task-001",
  "messages": [
    {
      "role": "user",
      "content": "Implement OpenViking as LibreFang's memory layer.",
      "message_kind": "user_query"
    },
    {
      "role": "assistant",
      "content": "Implemented the MCP bridge and verified tests.",
      "message_kind": "assistant_step"
    }
  ],
  "task_history": {
    "status": "success",
    "summary": "Added native session commit and native context retrieval.",
    "files_changed": [
      "openviking/server/mcp_endpoint.py",
      "docs/en/agent-integrations/20-librefang.md"
    ]
  },
  "keep_recent_count": 0,
  "extract_self": true,
  "extract_peer": true
}
```

Response shape:

```json
{
  "status": "ok",
  "session_id": "lf-session-2026-10-08-001",
  "session_created": true,
  "workspace_id": "default",
  "project_id": "openviking-librefang",
  "user_id": "dthai",
  "agent_id": "coder",
  "task_id": "task-001",
  "peer_id": "workspace-default__project-openviking-librefang",
  "memory_policy": {
    "self": { "enabled": true },
    "peer": { "enabled": true }
  },
  "native_roots": {
    "session": "viking://user/librefang-admin/sessions/lf-session-2026-10-08-001",
    "project_peer_memories": "viking://user/librefang-admin/peers/workspace-default__project-openviking-librefang/memories",
    "user_memories": "viking://user/librefang-admin/memories",
    "experiences": "viking://user/librefang-admin/memories/experiences"
  },
  "event_tags": [
    "workspace_id=default",
    "project_id=openviking-librefang",
    "source=librefang",
    "user_id=dthai",
    "agent_id=coder",
    "task_id=task-001"
  ],
  "messages_added": 3,
  "commit": {
    "session_id": "lf-session-2026-10-08-001",
    "status": "queued",
    "task_id": "..."
  }
}
```

Rules:

- Call this once when a task/session is done.
- Send the useful conversation transcript and task history. Do not send raw
  secrets or credentials.
- `extract_peer=true` lets OpenViking extract native project memory under the
  project peer.
- `extract_self=true` lets OpenViking extract native user/self memory for the
  authenticated OpenViking user.
- Agent evolution memories such as trajectories, cases, and experiences are
  controlled by OpenViking's native Agent Evolution configuration and memory
  type registry.
- The commit response can return a background `task_id`; native extraction may
  complete after the MCP call returns.

### workspace_memory_write

Writes one durable L2 memory file with metadata frontmatter. OpenViking derives
L0/L1 during indexing.

Required arguments:

```json
{
  "workspace_id": "default",
  "project_id": "openviking-librefang",
  "scope": "project",
  "memory_type": "decision",
  "content": "LibreFang should call OpenViking through MCP for durable memory."
}
```

Optional arguments:

```json
{
  "title": "MCP memory decision",
  "tags": ["architecture", "mcp"],
  "user_id": "dthai",
  "agent_id": "librefang",
  "wait": false,
  "timeout": 30
}
```

Rules:

- `scope="project"` stores project facts, architecture decisions, conventions,
  and durable task knowledge.
- `scope="user"` requires `user_id` and stores stable user preferences for that
  workspace/project.
- `scope="agent"` requires `agent_id` and stores reusable agent workflow lessons.
- `memory_type` classifies the full L2 memory so multi-agent and self-improve
  flows can route it later. Allowed values are `fact`, `decision`, `preference`,
  `instruction`, `experience`, and `reflection`. Use `experience` for reusable
  lessons learned from task outcomes.
- Obvious secret-like content is rejected. Store credentials in a vault, not in
  OpenViking memory.

LibreFang writes one full memory body. OpenViking stores that as L2 and derives
L0/L1 retrieval views during indexing. LibreFang chooses `detail` when reading,
not when writing.

Example self-improvement memory:

Tool name:

```text
workspace_memory_write
```

Arguments:

```json
{
  "workspace_id": "default",
  "project_id": "openviking-librefang",
  "scope": "agent",
  "agent_id": "coder",
  "memory_type": "experience",
  "title": "Debug workspace memory search",
  "content": "When workspace_memory_search returns no items after a write, first check whether the files were indexed as resource context and verify generic find on the target URI. Restart the server after source changes before retesting MCP tools.",
  "tags": ["self-improve", "mcp", "debugging"],
  "wait": false
}
```

Example project decision:

Tool name:

```text
workspace_memory_write
```

Arguments:

```json
{
  "workspace_id": "default",
  "project_id": "openviking-librefang",
  "scope": "project",
  "memory_type": "decision",
  "title": "LibreFang uses OpenViking via MCP",
  "content": "LibreFang should call OpenViking through MCP for durable workspace/project memory. LibreFang remains responsible for deciding what to store; OpenViking validates, persists, indexes, and retrieves memory.",
  "tags": ["architecture", "memory-layer"]
}
```

### workspace_memory_search

Searches workspace/project memories.

Tool name:

```text
workspace_memory_search
```

Arguments:

```json
{
  "workspace_id": "default",
  "project_id": "openviking-librefang",
  "query": "How should LibreFang persist durable project memory?",
  "scope": "project",
  "limit": 10,
  "min_score": 0.35,
  "detail": "overview",
  "read_content": false
}
```

Omit `scope` to search all project, user, and agent memories for the project.
When searching `scope="user"` or `scope="agent"`, pass the matching `user_id` or
`agent_id` to narrow the owner.

`detail` controls how much of each hit is returned:

- `abstract`: L0, smallest summary for routing and quick ranking.
- `overview`: L1, default; enough context for normal task planning.
- `full`: L2, full readable file content for the few hits LibreFang needs to
  inspect deeply.

`read_content=true` is kept for compatibility and behaves like `detail="full"`.

### workspace_context

Retrieves task-start context buckets for LibreFang.

Tool name:

```text
workspace_context
```

Arguments:

```json
{
  "workspace_id": "default",
  "project_id": "openviking-librefang",
  "user_id": "dthai",
  "agent_id": "librefang",
  "query": "User wants to connect OpenViking as LibreFang's memory layer",
  "project_limit": 5,
  "user_limit": 3,
  "agent_limit": 3,
  "resource_limit": 3,
  "skill_limit": 2,
  "native_limit": 3,
  "include_native": true,
  "min_score": 0.35,
  "detail": "overview"
}
```

The response separates:

```json
{
  "project_memories": [],
  "user_memories": [],
  "agent_memories": [],
  "native_project_memories": [],
  "native_user_memories": [],
  "native_experiences": [],
  "resources": [],
  "skills": []
}
```

LibreFang should call this before planning a task and inject the relevant
results into its own context. It should call `workspace_memory_write` only for
durable facts, decisions, preferences, or reusable agent lessons.

Use `detail="overview"` by default. Use `detail="abstract"` when LibreFang only
needs to decide whether more context exists, and retry with `detail="full"` only
for selected memories/resources that must be read verbatim.

Typical task-start sequence:

```text
1. LibreFang receives a user task.
2. LibreFang resolves workspace_id, project_id, user_id, and agent_id.
3. LibreFang calls workspace_context(detail="overview", include_native=true).
4. LibreFang injects relevant project_memories, user_memories, agent_memories,
   native_* buckets, resources, and skills into its own planning context.
5. If a returned item needs exact text, LibreFang reads it with detail="full"
   or calls read(uri).
```

## Recommended LibreFang Policy

Use LibreFang native memory for short-term runtime state. Use OpenViking for
durable and cross-agent memory:

```text
Before each task:
1. Determine workspace_id, project_id, user_id, and agent_id.
2. Call workspace_context with the user's task as query.
3. Read or inject the returned context buckets.

During or after each task:
1. Call workspace_session_commit with the finished conversation and task history.
2. Optionally identify curated durable memory candidates.
3. Classify each curated candidate by scope and memory_type.
4. Call workspace_memory_write for approved curated candidates.
5. Use memory_type="experience" for explicit reusable lessons learned from outcomes.
6. Never store secrets, raw credentials, or transient logs.
```

## Multi-Agent Notes

For a LibreFang multi-agent run, give each role a stable `agent_id`:

```text
planner
coder
reviewer
researcher
```

Use shared `project` memories for facts and decisions that every agent should
reuse. Use `agent` memories for role-specific lessons.

Example:

```text
planner writes: scope=agent, agent_id=planner, memory_type=experience
coder writes:   scope=agent, agent_id=coder,   memory_type=experience
reviewer writes shared convention:
                scope=project, memory_type=instruction
```

## Operational Notes

- `wait=false` returns after the file is written and queues semantic/vector
  indexing. This is the recommended default for interactive agents.
- `wait=true` blocks until indexing catches up, but local Ollama indexing can be
  slow and may hit the timeout even though the file was written.
- To verify a write immediately, call `read` on the returned `uri`.
- To verify search, wait for indexing and then call `workspace_memory_search`.
- After changing OpenViking source code, restart `openviking-server`; the running
  process does not hot-reload source changes.
