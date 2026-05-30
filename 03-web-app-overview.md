# Web App Overview

## Building a Claude Code Dashboard

The previous pages covered hooks and folder structure. This page covers how to put them together into a web app that logs and displays your Claude Code sessions.

### What You're Building

At its core, a Claude Code dashboard is a webhook receiver with a UI on top. The moving parts are:

1.  **An HTTP API** that receives hook events from Claude Code
    
2.  **A database** that stores sessions and their events
    
3.  **A web frontend** that displays the session timeline
    
4.  **A background worker** (optional) for post-processing like summaries, link extraction, or sending notifications
    

The language and framework don't matter. What matters is that you can receive HTTP POSTs, store JSON, and render a page.

### Data Model

You need two core tables and one optional table:

**Sessions** -- one row per Claude Code session

| Column | Type | Purpose |
| --- | --- | --- |
| session\_id | string (unique) | Claude Code's session identifier, used to correlate hook events |
| company / project | foreign key | Which project this session belongs to (matched by `cwd`) |
| cwd | string | Working directory where Claude was running |
| transcript\_path | string, nullable | Path to the JSONL transcript file on disk |
| status | enum | `active`, `idle`, `processing` |
| name | string, nullable | Auto-generated or user-edited session title |
| started\_at | timestamp | When the session began |
| ended\_at | timestamp, nullable | When the session ended |

**Session Events** -- one row per hook event or parsed transcript entry

| Column | Type | Purpose |
| --- | --- | --- |
| session\_id | foreign key | Which session this event belongs to |
| event\_type | string | `SessionStart`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SessionEnd`, `AssistantMessage` |
| tool\_name | string, nullable | For tool events: `Read`, `Edit`, `Bash`, etc. |
| tool\_input | json, nullable | The tool's input parameters |
| payload | json | The full hook payload -- store everything |
| created\_at | timestamp | When the event occurred |

**Projects / Companies** (optional) -- one row per client or codebase

| Column | Type | Purpose |
| --- | --- | --- |
| name | string | Display name |
| project\_path | string | Base path used to match sessions by `cwd` prefix |
| settings\_json | json, nullable | Cached copy of the project's `.claude/settings.local.json` |

### API Endpoints

You need 4 endpoints, all accepting JSON POST with bearer token auth:

```
POST /api/hooks/session-start
POST /api/hooks/tool-use          (handles PostToolUse + UserPromptSubmit)
POST /api/hooks/response-complete (handles Stop)
POST /api/hooks/session-end
```

The handler logic for each is straightforward:

**session-start:**

```
find or create session by session_id
set status = "active" (unless already "processing")
match project by cwd prefix
store the event
```

**tool-use:**

```
find or create session by session_id
read hook_event_name from payload to get actual event type
store the event with tool_name and tool_input
```

**response-complete:**

```
find or create session by session_id
store the event (payload includes last_assistant_message)
```

**session-end:**

```
find session by session_id (404 if missing)
set status = "idle", set ended_at
store the event
trigger any post-processing (summaries, notifications)
```

**In every handler**, create the session if it doesn't exist yet. Hooks can arrive out of order -- a `PostToolUse` might land before `SessionStart`.

### Matching Sessions to Projects

When a hook arrives, it includes `cwd` (e.g., `/Users/you/Projects/ClientX/src`). To associate it with a project, do a prefix match against your registered project paths:

```
for each project:
    if cwd starts with project.project_path:
        return project
```

This is why folder structure matters -- clean, non-overlapping project paths make matching trivial.

### The Transcript File (Optional but Valuable)

Hooks give you _events_ (tool calls, status changes) but not the full assistant text. For that, you need to parse the transcript file.

The `transcript_path` field in hook payloads points to a JSONL file on disk that Claude Code writes to in real time. Each line is a JSON object representing an API interaction, including the assistant's full responses and token usage.

**Parsing strategy:**

```
track a byte offset per session (start at 0)
on each hook event:
    open transcript file, seek to offset
    for each line:
        try to parse as JSON
        if parse fails: stop (line is still being written)
        if it's an assistant text block: create an AssistantMessage event
        advance offset past this line
    save new offset to session
```

This gives you incremental, non-blocking reads that work while Claude is still streaming.

**If you skip transcript parsing**, you still get a useful dashboard -- you'll see every tool call, user message, and session boundary. You just won't have the assistant's full text responses inline.

### Post-Processing on Session End

`SessionEnd` is the natural trigger for background work:

-   **Auto-naming**: Send the first user prompt to a cheap/fast LLM (e.g., Claude Haiku) and ask for a 3-4 word title
    
-   **Summarization**: Send assistant message text to an LLM for a 2-3 sentence summary of what was accomplished
    
-   **Link extraction**: Scan event payloads for GitHub PR/issue URLs, Jira ticket links, etc.
    
-   **Notifications**: Post to Slack, send an email, update a project tracker
    

These should run asynchronously (background job, queue worker, async task) so the hook response isn't delayed.

### Frontend

The UI can be as simple or as rich as you want. The minimum useful views are:

**Session list** -- table of sessions with status, project, name, timestamp, event count. Filter by project and active/archived state.

**Session detail** -- chronological event timeline. Group events by assistant turn (each `AssistantMessage` starts a group, tool calls within that turn are children). Collapsible groups keep the view manageable for long sessions.

### Tech Stack Options

This can be built in whatever you're comfortable with. The requirements are minimal:

| Requirement | Options |
| --- | --- |
| HTTP API | Express/Fastify (Node), [http://ASP.NET](http://ASP.NET) Core (C#), Laravel/Lumen (PHP), FastAPI/Flask (Python), Gin (Go) |
| Database | PostgreSQL, MySQL, SQLite (for single-user), MongoDB |
| Background jobs | BullMQ (Node), Hangfire (C#), Laravel Horizon (PHP), Celery (Python) |
| Frontend | React, Vue, Svelte, Blazor, server-rendered templates -- anything |
| Real-time updates | WebSockets, SSE, or just polling on an interval |

For a single-user personal dashboard, SQLite + a single-process server is perfectly fine. You don't need Redis or a queue worker unless you want async post-processing.

### What You Can Skip

Not everything is day-one necessary. A pragmatic build order:

| Phase | What to build |
| --- | --- |
| 1. **Core** | 4 hook endpoints, session + event tables, bearer token auth, basic session list page |
| 2. **Useful** | Project matching by cwd, session detail timeline, event grouping |
| 3. **Nice to have** | Transcript parsing, auto-naming/summaries, link extraction, real-time updates |
| 4. **Power features** | Send messages back to Claude, settings editor with disk sync, skill management |

Phase 1 is an afternoon of work in any framework. You get session logging immediately and can iterate from there.
