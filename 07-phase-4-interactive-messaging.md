# Phase 4: Interactive Messaging and Session Management in Claude Code

---

## Phase 4: Sending Messages, Session Management, and Settings Sync

Phases 1-3 built a read-only dashboard. Phase 4 makes it interactive -- you can send messages to Claude from the browser, manage session lifecycle, and edit Claude Code settings for each project without touching the filesystem.

### Sending Messages to Claude

This is the feature that turns your dashboard from a log viewer into a control panel. Instead of switching to a terminal to resume a session, you type a message in the browser and Claude picks up where it left off.

#### How It Works

Claude Code has a non-interactive mode (`claude -p`) that accepts a prompt, runs to completion, and exits. Combined with `--resume`, it continues an existing session. Your app dispatches a background job that shells out to the CLI:

```
claude -p --resume <session_id> --permission-mode <mode> "your message here"
```

The hooks you set up in Phase 1 fire during this subprocess just like they do during a normal interactive session. Your dashboard sees the events in real time.

#### The Flow

```
Browser                   App Server                Background Worker         Claude CLI
   |                          |                          |                        |
   |-- POST /send-message --> |                          |                        |
   |                          |-- set status=processing  |                        |
   |                          |-- dispatch job ---------> |                        |
   |<-- 200 { status: ok } -- |                          |                        |
   |                          |                          |-- claude -p --resume -> |
   |                          |                          |                        |
   |                          | <-- hooks fire (session-start, tool-use, stop) -- |
   |                          |                          |                        |
   |                          |                          | <-- CLI exits --------- |
   |                          |                          |-- set status=idle       |
   |                          |                          |-- dispatch summary job  |
```

**The app server returns immediately.** The actual Claude work happens in a background job that may take seconds or minutes. The browser doesn't wait -- it sees events arrive via hooks (and optionally via WebSocket/polling).

#### Schema Changes

You don't need new tables. Add a status value to track the in-between state:

```
sessions.status: "active" | "idle" | "processing"
```

-   **active** -- Claude Code is running interactively in a terminal (set by SessionStart hook)
    
-   **idle** -- no Claude process is running (set by SessionEnd hook or job completion)
    
-   **processing** -- your app dispatched a job that hasn't started the CLI yet, or the CLI is running as a subprocess
    

The `processing` state exists because there's a gap between "user clicked send" and "Claude CLI starts and fires SessionStart." Without it, the session appears idle during that gap, and the user might send another message.

#### The Send Message Endpoint

```
POST /sessions/{id}/message

Body: {
    "message": "Fix the failing test in UserControllerTest",
    "permission_mode": "bypassPermissions"  -- optional, defaults to bypassPermissions
}
```

```
function handleSendMessage(request, sessionId):
    session = db.sessions.find(sessionId)

    if session.status == "active":
        return 409 { "error": "Session is active in a terminal. Cannot send messages." }

    if session.status == "processing":
        return 409 { "error": "A message is already being processed." }

    session.status = "processing"
    session.save()

    autoName = session.name is null

    dispatchBackground(processClaudeMessage, {
        session: session,
        message: request.message,
        permissionMode: request.permission_mode or "bypassPermissions",
        autoName: autoName
    })

    return 200 { "status": "ok" }
```

**Why reject active sessions?** If someone is using Claude interactively in a terminal, sending a message from the dashboard would collide. The CLI doesn't support concurrent input.

#### The Background Job

```
function processClaudeMessage(session, message, permissionMode, autoName):
    try:
        -- Auto-name before running Claude (so the list view updates immediately)
        if autoName:
            name = generateQuickName(message)
            if name:
                session.name = name
                session.save()

        cwd = session.cwd or appBasePath

        -- Try --resume first (works for any session that has run at least once)
        result = exec(
            command: [claudePath, "-p", "--resume", session.session_id,
                      "--permission-mode", permissionMode, message],
            workingDirectory: cwd,
            timeout: 1800  -- 30 minutes
        )

        if result failed:
            -- Session doesn't exist in Claude's state yet (brand new manual session)
            -- Fall back to --session-id which starts a fresh session with our ID
            exec(
                command: [claudePath, "-p", "--session-id", session.session_id,
                          "--permission-mode", permissionMode, message],
                workingDirectory: cwd,
                timeout: 1800
            )

    finally:
        session.status = "idle"
        session.save()
        dispatchBackground(generateSessionSummary, session)
```

#### Permission Modes

Claude Code supports permission modes that control what the CLI can do without human approval. The two useful modes for a dashboard:

| Mode | Behavior |
| --- | --- |
| `bypassPermissions` | Claude can use any tool without asking. Fast, no human in the loop. Use when you trust the prompt. |
| `plan` | Claude can only read files and think. No writes, no shell commands. Use for analysis or when you want to review before acting. |

Expose this as a dropdown or toggle in the send-message UI. Default to `bypassPermissions` for convenience, but offer `plan` as a safe option.

#### Safety Prefix

If Claude is running with `bypassPermissions`, it can do anything -- including destructive database operations. Prepend safety rules to every message:

```
safetyPrefix = "CRITICAL SAFETY RULES: You MUST NOT run any destructive database commands. ..."

fullMessage = safetyPrefix + "\n\n" + userMessage
```

This isn't bulletproof, but it's a strong guardrail. Claude follows system-level instructions reliably, and the prefix is indistinguishable from a system prompt in non-interactive mode.

**Strip the prefix before auto-naming.** Otherwise your session names will be about safety rules, not the actual task.

#### Rate Limiting

If your Claude CLI authenticates via an OAuth subscription (the default), you're sharing a rate limit with your interactive terminal sessions. Running multiple concurrent CLI subprocesses will hit that limit fast.

**Cap your background worker concurrency to 1.** One message processing at a time. If you use a queue system, configure it with a single worker process for the Claude message queue. This means messages are serialized -- if you send a message to session A while session B is still processing, session A's message waits in the queue.

This is a real constraint. If you switch to API key authentication instead of OAuth subscription, you can raise concurrency, but that's a different billing model.

#### The Resume vs Session-ID Fallback

Two ways to start a Claude CLI subprocess against a session:

-   `--resume <session_id>` -- Continues an existing Claude session. Works for any session that Claude has seen before (including hook-created sessions where Claude was the one that started the original interactive session).
    
-   `--session-id <session_id>` -- Starts a new session with a specific ID. Only works if Claude hasn't seen this ID before.
    

Try `--resume` first. If it fails (non-zero exit code), the session was created manually through your dashboard and Claude has never seen it. Fall back to `--session-id` to bootstrap it.

### Session Lifecycle Management

With the send-message feature, sessions need more lifecycle controls.

#### Manual Session Creation

Let users create a session from the dashboard without starting Claude Code in a terminal:

```
POST /sessions

Body: {
    "project_id": 1,
    "cwd": "/Users/you/Projects/SendJim"
}
```

```
function handleCreateSession(request):
    project = db.projects.find(request.project_id)

    session = db.sessions.create(
        session_id: generateUUID(),
        project_id: project.id,
        cwd: request.cwd,
        status: "idle",
        started_at: now()
    )

    redirect to session detail page
```

The user then sends a message from the detail page, which triggers the Claude CLI subprocess.

#### Take Control (End Active Session)

If a session is stuck in `active` status (the terminal was closed without a clean exit, or the hook failed), the user needs a way to force it to idle:

```
POST /sessions/{id}/take-control
```

```
function handleTakeControl(sessionId):
    session = db.sessions.find(sessionId)

    if session.status != "active":
        return 409 { "error": "Session is not active" }

    session.status = "idle"
    session.ended_at = now()
    session.save()

    return 200 { "status": "ok" }
```

This doesn't actually kill any process -- it just updates the database status so the dashboard allows sending messages again. Use it when the session state is stale.

#### Rename

Auto-naming is good but not always right. Let users rename sessions:

```
PATCH /sessions/{id}/rename

Body: { "name": "PROJ-891 Payment Gateway Migration" }
```

#### Soft Delete

Sessions accumulate fast. Let users delete sessions without losing data:

```
ALTER TABLE sessions ADD COLUMN deleted_at TIMESTAMP;
```

Filter all queries with `WHERE deleted_at IS NULL`. The data is still there for auditing or recovery.

#### Archive

Deletion is binary. Archiving is a softer "get this out of my active list":

```
ALTER TABLE sessions ADD COLUMN archived_at TIMESTAMP;
```

Your session list page shows two views: active and archived, toggled by a tab or filter. The default view excludes archived sessions.

#### Flag / Bookmark

Some sessions are worth revisiting -- a particularly good debugging session, a session you want to reference in a retrospective, a session that exposed a bug in your tooling:

```
ALTER TABLE sessions ADD COLUMN is_flagged BOOLEAN DEFAULT FALSE;
```

A simple toggle in the UI. Filter by flagged sessions when you need to find something you bookmarked.

#### Notes

Free-text notes attached to a session. The summary is auto-generated, but notes are human-written context:

```
ALTER TABLE sessions ADD COLUMN notes TEXT;
```

```
PATCH /sessions/{id}/notes

Body: { "notes": "This session found the race condition in the payment webhook handler. See PROJ-891 for follow-up." }
```

#### Pinned Links

Sessions reference external resources -- but the auto-extracted GitHub/Jira links from Phase 3 only capture what appeared in the conversation text. Sometimes you want to manually pin a link that's relevant but wasn't mentioned:

```
CREATE TABLE pinned_links (
    id              SERIAL PRIMARY KEY,
    session_id      INTEGER NOT NULL REFERENCES sessions(id),
    title           VARCHAR(255) NOT NULL,
    url             VARCHAR(2000) NOT NULL,
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);
```

CRUD endpoints:

```
POST   /sessions/{id}/pinned-links          -- add a link
PATCH  /pinned-links/{id}                   -- edit title/url
DELETE /pinned-links/{id}                   -- remove
```

Display pinned links alongside auto-extracted links on the session detail page, but visually distinguish them (e.g., a pin icon) so users know which were manually added.

### Settings Editor

Each project has a `.claude/settings.local.json` file that controls Claude Code's behavior in that project -- API tokens, environment variables, permissions, allowed tools. Editing these files means SSHing or opening a terminal and navigating to the right path. A web editor is faster.

#### How It Works

Your projects table already has `settings_json` (a cached copy of the file). The editor loads from the database, saves to the database AND writes back to disk:

```
GET  /projects/{id}/settings      -- render editor with current JSON
PUT  /projects/{id}/settings      -- save to DB + write to disk
POST /projects/{id}/settings/sync -- read from disk into DB
```

**Save flow:**

```
function handleSettingsUpdate(request, projectId):
    project = db.projects.find(projectId)

    json = request.settings_json

    -- Validate it's actual JSON
    try:
        parsed = parseJson(json)
    catch:
        return 422 { "error": "Invalid JSON" }

    -- Security: validate path is within allowed base
    if not project.project_path.startsWith(allowedBasePath):
        return 403 { "error": "Path outside allowed directory" }

    -- Save to database
    project.settings_json = parsed
    project.save()

    -- Write to disk
    filePath = project.project_path + "/.claude/settings.local.json"
    ensureDirectoryExists(dirname(filePath))
    writeFile(filePath, prettyPrintJson(parsed))

    redirect back with success message
```

**Sync from disk flow:**

```
function handleSettingsSync(projectId):
    project = db.projects.find(projectId)

    filePath = project.project_path + "/.claude/settings.local.json"

    if not fileExists(filePath):
        redirect back with error "File not found on disk"

    content = readFile(filePath)
    parsed = parseJson(content)

    project.settings_json = parsed
    project.save()

    redirect back with success message
```

**Why bidirectional sync?** Sometimes you edit settings from the dashboard. Sometimes you (or Claude itself) edit the file on disk. Sync-from-disk lets you pull those changes into the dashboard without manual copy-paste.

#### Path Validation

**This is a security boundary.** Your settings editor writes files to disk at paths derived from database content. Without validation, a tampered `project_path` could write to arbitrary locations.

Validate that every disk write targets a path under a known safe base:

```
allowedBasePath = "/Users/you/Projects"

if not resolvedPath.startsWith(allowedBasePath):
    reject
```

Check the resolved/canonical path, not just the string, to prevent `../../` traversal.

#### The Editor UI

A JSON editor doesn't need to be fancy. A `<textarea>` with monospace font works. Useful additions:

-   **Format button** -- pretty-print the JSON on demand
    
-   **Validation feedback** -- parse the JSON on save and show a clear error if it's invalid, with the line number if possible
    
-   **Sync from disk button** -- one click to pull the current file contents
    
-   **Diff indicator** -- show whether the database copy matches the disk copy (stale detection)
    

### Real-Time Updates (Optional)

Up to now, the session detail page is static -- you load it and see the events that existed at page load. If Claude is actively working (processing a message or running interactively), new events appear in the database but the browser doesn't know.

Three approaches, in order of complexity:

**Polling** -- Simplest. Hit an API endpoint every 5 seconds and refresh the event list. Works fine for low-traffic single-user dashboards. Wasteful if nothing is changing.

**Server-Sent Events (SSE)** -- The server pushes updates to the browser over a long-lived HTTP connection. Good middle ground. One-directional (server to browser), no special infrastructure needed.

**WebSockets** -- Full bidirectional communication. Best real-time experience but requires a WebSocket server (e.g., Laravel Reverb, [http://Socket.IO](http://Socket.IO) , SignalR). Overkill for Phase 4 unless your framework makes it easy.

Whichever approach you choose, the broadcast trigger is the same: **every time you record an event or update session status, notify connected clients.**

```
-- At the end of recordEvent() in your hook controller:
broadcast("session.{session.id}", {
    session_id: session.id,
    status: session.status,
    change_type: "event"  -- or "status" for status-only changes
})
```

The frontend listens on the session's channel and either:

-   Refetches the full event list (simple, slightly wasteful)
    
-   Appends the new event to the existing list (efficient, more complex)
    

Start with a full refetch. Optimize later if it matters.

---
