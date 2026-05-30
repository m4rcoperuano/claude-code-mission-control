# Phase 1: Core Session Logging

## Phase 1: Core Session Logging

This is the minimum viable Claude Code dashboard. By the end, every Claude Code session on your machine will be logged to a database with a web page to browse them. No transcript parsing, no summaries, no real-time updates -- just reliable event capture and a list view.

### Database Schema

Two tables. Keep them simple -- you can always add columns later.

**sessions**

```
CREATE TABLE sessions (
    id              SERIAL PRIMARY KEY,
    session_id      VARCHAR(255) NOT NULL UNIQUE,  -- Claude Code's identifier
    cwd             VARCHAR(1000),                  -- working directory
    transcript_path VARCHAR(1000),                  -- path to JSONL file on disk
    status          VARCHAR(20) DEFAULT 'active',   -- active | idle
    started_at      TIMESTAMP,
    ended_at        TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);
```

**session\_events**

```
CREATE TABLE session_events (
    id              SERIAL PRIMARY KEY,
    session_id      INTEGER NOT NULL REFERENCES sessions(id),
    event_type      VARCHAR(50) NOT NULL,           -- SessionStart, PostToolUse, etc.
    tool_name       VARCHAR(100),                   -- Read, Edit, Bash, etc.
    tool_input      JSONB,                          -- tool parameters
    payload         JSONB NOT NULL,                 -- full hook payload
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_session_events_session ON session_events(session_id);
CREATE INDEX idx_session_events_type ON session_events(event_type);
```

**Why JSONB for payload?** Claude Code's hook payloads vary by event type and may add fields over time. Storing the full payload as JSON means you never lose data, and you can query into it later if your database supports it.

**Why a separate** `tool_name` **column if it's already in the payload?** You'll filter and display by tool name constantly. Pulling it out into its own indexed column saves you from JSON path queries on every list view.

### Authentication Middleware

Every hook endpoint needs the same auth check. Write it once as middleware.

```
function verifyHookToken(request):
    expected = env("CLAUDE_HOOKS_TOKEN")

    if expected is empty:
        return 401  -- misconfigured server, reject everything

    token = extractBearerToken(request.headers["Authorization"])

    if token != expected:
        return 401 { "error": "Unauthorized" }

    return next(request)
```

**Pick a strong token.** This is the only thing protecting your endpoints. Generate something random:

```
openssl rand -hex 32
```

Store it in your app's environment config and in your Claude Code `settings.json` hook headers. They must match exactly.

### Endpoint 1: Session Start

```
POST /api/hooks/session-start
```

This is the most nuanced endpoint because of idempotency concerns.

```
function handleSessionStart(request):
    payload = request.body
    sessionId = payload.session_id

    if sessionId is missing:
        return 422 { "error": "Missing session_id" }

    -- Try to find existing session first
    session = db.sessions.findBy(session_id: sessionId)

    if session exists:
        -- Update metadata but respect existing status
        session.cwd = payload.cwd
        session.transcript_path = payload.transcript_path
        session.started_at = now()
        session.save()
    else:
        -- Create new session
        session = db.sessions.create(
            session_id:       sessionId,
            cwd:              payload.cwd,
            transcript_path:  payload.transcript_path,
            status:           "active",
            started_at:       now()
        )

    -- Record the event
    db.session_events.create(
        session_id:  session.id,
        event_type:  "SessionStart",
        payload:     payload,
        created_at:  now()
    )

    return 200 { "status": "ok" }
```

**Why not just upsert?** A naive upsert would overwrite `status` to `active` every time. If you later add Phase 4 features (sending messages to Claude from the UI), your app will set status to `processing` before the CLI subprocess starts. That subprocess fires `SessionStart`, and a blind upsert would clobber `processing` back to `active`. Separating "find or create" from "update fields" avoids this entirely.

**Why store** `started_at` **explicitly?** Don't rely on `created_at`. A session can be resumed -- `SessionStart` fires again for an existing session, and you want `started_at` to reflect the latest start, not when the row was first created.

### Endpoint 2: Tool Use

```
POST /api/hooks/tool-use
```

This single endpoint handles both `PostToolUse` and `UserPromptSubmit` events. The `hook_event_name` field in the payload tells you which.

```
function handleToolUse(request):
    payload = request.body
    session = findOrCreateSession(payload)

    db.session_events.create(
        session_id:  session.id,
        event_type:  payload.hook_event_name or "PostToolUse",
        tool_name:   payload.tool_name,      -- null for UserPromptSubmit
        tool_input:  payload.tool_input,      -- null for UserPromptSubmit
        payload:     payload,
        created_at:  now()
    )

    return 200 { "status": "ok" }
```

`findOrCreateSession` **is a helper you'll reuse.** Hooks can arrive out of order -- a `PostToolUse` may land before `SessionStart` if the start hook is slow or fails. This helper ensures every event has a session to attach to:

```
function findOrCreateSession(payload):
    sessionId = payload.session_id or "unknown"
    session = db.sessions.findBy(session_id: sessionId)

    if session is null:
        session = db.sessions.create(
            session_id:       sessionId,
            cwd:              payload.cwd,
            transcript_path:  payload.transcript_path,
            status:           "active",
            started_at:       now()
        )

    -- Backfill transcript_path if session existed without one
    if session.transcript_path is null and payload.transcript_path is not null:
        session.transcript_path = payload.transcript_path
        session.save()

    return session
```

**The transcript path backfill matters.** When a session is first created by a tool-use event (because `SessionStart` hasn't arrived yet), it might not have the transcript path. Later events carry it, so always backfill.

### Endpoint 3: Response Complete

```
POST /api/hooks/response-complete
```

Fires when Claude finishes generating a response (`Stop` event).

```
function handleResponseComplete(request):
    payload = request.body
    session = findOrCreateSession(payload)

    db.session_events.create(
        session_id:  session.id,
        event_type:  "Stop",
        payload:     payload,
        created_at:  now()
    )

    return 200 { "status": "ok" }
```

This is intentionally simple. The payload includes `last_assistant_message` which you're storing in the JSON payload column. In Phase 1 you don't do anything with it -- but it's there when you need it.

### Endpoint 4: Session End

```
POST /api/hooks/session-end
```

The only endpoint that requires the session to already exist. If it doesn't, something went wrong and a 404 is the right response.

```
function handleSessionEnd(request):
    payload = request.body
    sessionId = payload.session_id

    session = db.sessions.findBy(session_id: sessionId)

    if session is null:
        return 404 { "error": "Session not found" }

    session.status = "idle"
    session.ended_at = now()
    session.save()

    db.session_events.create(
        session_id:  session.id,
        event_type:  "SessionEnd",
        payload:     payload,
        created_at:  now()
    )

    return 200 { "status": "ok" }
```

**Why 404 instead of auto-creating?** A `SessionEnd` for an unknown session means something is broken -- the start and all mid-session hooks were lost. Creating a session just to immediately mark it idle produces a useless record. Better to surface the error.

### Session List Page

One page, one query, one table. This is your "did it work?" verification and your daily driver for browsing sessions.

**Query:**

```
SELECT
    s.id,
    s.session_id,
    s.cwd,
    s.status,
    s.started_at,
    s.ended_at,
    COUNT(e.id) AS event_count
FROM sessions s
LEFT JOIN session_events e ON e.session_id = s.id
GROUP BY s.id
ORDER BY s.updated_at DESC;
```

**Display columns:**

| Column | What to show |
| --- | --- |
| Status | Color-coded badge: green for `active`, gray for `idle` |
| Working directory | Trim to last 2-3 path segments (e.g., `SendJim/app`) |
| Events | Count of events in the session |
| Started | Relative time (`3 minutes ago`, `yesterday`) |
| Duration | Difference between `ended_at` and `started_at`, or "ongoing" if null |

**Link each row** to a placeholder detail page (even if it's just a raw JSON dump of events for now). You'll build the real timeline view in Phase 2.

### Testing It

Once your endpoints are up and your `~/.claude/settings.json` has the hooks configured (see the Hooks page), open a Claude Code session and do something simple:

```
claude "list the files in this directory"
```

Then check your database:

```
SELECT session_id, status, cwd FROM sessions ORDER BY created_at DESC LIMIT 5;
SELECT event_type, tool_name, created_at FROM session_events ORDER BY created_at DESC LIMIT 20;
```

You should see a `SessionStart`, one or more `PostToolUse` events (likely a `Bash` or `Read` tool call), a `Stop`, a `UserPromptSubmit`, and a `SessionEnd`. If you see the session but no events, your event recording is broken. If you see nothing, check your bearer token matches and your app is reachable at the URL in your hook config.

### Common Pitfalls

**Hooks silently fail.** Claude Code doesn't surface hook errors to the user. If your server is down or returns a 500, the hook just doesn't fire. Add logging on every request -- at minimum, log the endpoint hit and the `session_id`. When something seems missing, check your server logs first.

**Timestamps and timezones.** Store everything in UTC. Claude Code's transcript uses UTC timestamps. Your database server might default to a local timezone. Set your app and database timezone explicitly to avoid confusion when events appear out of order in the UI.

**Payload size.** `PostToolUse` payloads for tools like `Edit` or `Write` can include full file contents in `tool_input`. Make sure your JSON column and HTTP body parser can handle large payloads. Set your body size limit to at least 5 MB.

**Response time.** Claude Code waits for hook responses before continuing. Keep your handlers fast -- accept the payload, write to the database, return 200. Do any expensive processing asynchronously. If your handlers take more than a second or two, you'll feel it as latency in every Claude Code interaction.
