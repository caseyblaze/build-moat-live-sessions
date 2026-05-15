# ChatGPT Task Scheduler Prototype

## System Requirements

Build a job scheduler with an MCP (Model Context Protocol) interface:
- Users schedule tasks for future execution via MCP tool calls
- A background watcher scans for due jobs and pushes them to a queue
- Workers pull jobs from the queue and execute them
- Support task creation, listing, status checking, and cancellation
- Tool naming follows namespace + action verb pattern (e.g., `task_create`)

### Architecture

```
User → MCP Tool Call → Job Scheduler API → DB
                                            ↓
                              Watcher (scans DB) → Queue → Worker (executes)
```

## Design Questions

Answer these before you start coding:

1. **Watcher vs Cron:** Why separate the watcher from the worker? What problems does a single cron job that both scans and executes have?

2. **Queue Layer:** Why put a queue between the watcher and worker instead of having the watcher call the worker directly? What are the benefits?

3. **Time Bucket Partitioning:** Instead of `SELECT * WHERE scheduled_at <= now()`, why partition jobs by time bucket (e.g., hour)? What happens to query performance at 1M+ jobs without partitioning?

4. **Tool Naming:** Why `task_create` instead of `createTask`? How does naming convention affect LLM tool selection accuracy?

5. **Registry vs If-Else:** Why use a dictionary registry to route tool calls instead of if-else chains? What happens when you need to add the 20th tool?

## Verification

Your prototype is a real MCP server. Test it with the MCP inspector — no Claude needed.

### 1. Start the server (sanity check)

The `app` package lives in `scaffold/`, so run the server from there:

```bash
cd scaffold && python -m app.mcp_server
```

The process should hang waiting on stdin (it's a stdio MCP server — that's correct). Ctrl+C to stop. If you see an `ImportError` or other crash, fix that first.

### 2. Run the MCP inspector

Requires Node.js (uses `npx`).

```bash
cd scaffold && npx @modelcontextprotocol/inspector python -m app.mcp_server
```

This opens a browser GUI (usually `http://localhost:5173`).

Steps in the GUI:

1. Click **Connect** -> should show 4 tools: `task_create`, `task_list`, `task_status`, `task_cancel`
2. **task_create** -> fill `description="Summarize tech news"`, `scheduled_at="2025-01-01T00:00:00"` (past time so watcher picks it up immediately) -> **Run Tool** -> response should include `{"job_id": 1, "status": "pending", ...}`
3. Wait ~10 seconds, then **task_status** -> `job_id: 1` -> status should now be `"completed"`
4. **task_create** with future time `"2099-12-31T00:00:00"` -> get `job_id: 2`
5. **task_cancel** -> `job_id: 2` -> status `"cancelled"`
6. **task_list** -> see all your jobs

### 3. (Optional) Connect to Claude Desktop / Claude Code

Once the inspector tests pass, the server is ready. To talk to it through Claude:

Both clients accept the same config — edit the appropriate file:
- **Claude Desktop**: `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS)
- **Claude Code**: `~/.claude.json` (top-level `mcpServers` for user scope)

```json
{
  "mcpServers": {
    "task-scheduler": {
      "command": "/bin/bash",
      "args": [
        "-c",
        "cd /absolute/path/to/scaffold && exec /absolute/path/to/scaffold/.venv/bin/python -m app.mcp_server"
      ]
    }
  }
}
```

The `bash -c` wrapper sets the working directory inside the process — `python -m app.mcp_server` must run from `scaffold/` to find the `app` package and the relative SQLite path. Claude Desktop also honors a top-level `cwd` field as an alternative, but Claude Code ignores `cwd` for stdio servers (and `claude mcp add` / `add-json` don't persist it), so the wrapper is the most portable form. Restart the client fully after saving — the 🔨 icon should show 4 tools.

Then chat:
> "Schedule a task to review PR #123 tomorrow at 9am."
> -> Claude calls `task_create` -> returns job_id
> "What's the status of that task?"
> -> Claude calls `task_status`

## Suggested Tech Stack

Python + the official `mcp` SDK is recommended (already in `requirements.txt` for the Guided Track). Challenge Track may use any language with an MCP SDK.
