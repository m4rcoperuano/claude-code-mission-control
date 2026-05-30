# Phase 2: Project Matching, Transcript Parsing, and Event Grouping

## Phase 2: Project Matching, Transcript Parsing, and Event Grouping

Phase 1 gave you a log of every hook event. Phase 2 turns that log into something you actually want to look at -- sessions tied to projects, full assistant responses in the timeline, and events grouped into coherent turns instead of a flat list.

### Project Matching

In Phase 1, sessions have a raw `cwd` string. That's useful for debugging but not for browsing. You want to see "this was a SendJim session" at a glance.

**Add a projects table:**

```
CREATE TABLE projects (
    id           SERIAL PRIMARY KEY,
    name         VARCHAR(255) NOT NULL,
    project_path VARCHAR(1000) NOT NULL UNIQUE,
    color        VARCHAR(7),                     -- hex color for UI badges
    created_at   TIMESTAMP DEFAULT NOW(),
    updated_at   TIMESTAMP DEFAULT NOW()
);
```

**Add a foreign key to sessions:**

```
ALTER TABLE sessions ADD COLUMN project_id INTEGER REFERENCES projects(id);
```

**Matching logic -- prefix match against registered paths:**

```
function matchProjectByCwd(cwd):
    for each project in db.projects.all():
        if cwd starts with project.project_path:
            return project
    return null
```

Call this in your `session-start` and `findOrCreateSession` handlers:

```
project = matchProjectByCwd(payload.cwd)
session.project_id = project.id if project exists
```

**Why prefix matching and not exact matching?** Claude Code's `cwd` is the actual directory it's running in, which is often a subdirectory of the project root. If your project path is `/Users/you/Projects/SendJim` and Claude is invoked from `/Users/you/Projects/SendJim/app/Services`, prefix matching still connects it to SendJim. Exact matching would miss it.

**Watch for overlapping paths.** If you have `/Users/you/Projects` registered as a project AND `/Users/you/Projects/SendJim`, a session in SendJim would match both. Two ways to handle this:

-   Sort projects by path length descending and return the first match (longest/most specific prefix wins)
    
-   Keep project paths non-overlapping by design (register leaf projects, not parent directories)
    

The second approach is simpler and what the sunny-bots codebase does.

### Transcript Parsing

This is the biggest addition in Phase 2 and the thing that transforms your dashboard from "list of tool calls" to "readable conversation."

#### Why You Need It

Hooks tell you _what Claude did_ (tool calls, start/stop signals) but not _what Claude said_. The assistant's actual text responses -- explanations, questions, reasoning -- only exist in the transcript file. Without parsing it, your timeline is a sequence of `PostToolUse: Edit`, `PostToolUse: Bash`, `Stop` with no context for why those tools were called.

#### The Transcript File Format

Claude Code writes a JSONL file (one JSON object per line) at the path given by `transcript_path`. It's written incrementally while the session is active. Each line represents an API interaction and contains the full request/response including streaming chunks.

The lines you care about for Phase 2 are **assistant message entries** -- objects that contain the assistant's text response. The key identifiers:

-   Look for entries with `type: "assistant"` and content blocks containing `type: "text"`
    
-   Each entry has a unique identifier you can use for deduplication
    
-   The file also contains streaming chunks (partial responses) -- you want the final complete messages, not every intermediate chunk
    

#### Incremental Parsing

The transcript file grows throughout the session. You don't want to re-read the entire file on every hook event. Track a byte offset per session and only read new content.

**Add an offset column:**

```
ALTER TABLE sessions ADD COLUMN transcript_offset BIGINT DEFAULT 0;
```

**Parser pseudocode:**

```
function parseNewEntries(session):
    if session.transcript_path is null:
        return
    if file does not exist at session.transcript_path:
        return

    file = open(session.transcript_path)
    seek to session.transcript_offset
    currentOffset = session.transcript_offset

    for each line in file:
        lineStart = currentOffset

        try:
            entry = parseJson(line)
        catch:
            -- Partial line (file still being written). Stop here.
            -- Do NOT advance offset past this line.
            break

        currentOffset = currentOffset + byteLength(line)

        if isAssistantTextMessage(entry):
            uuid = extractUniqueId(entry)

            -- Deduplicate: skip if we already recorded this message
            if db.session_events.exists(transcript_uuid: uuid):
                continue

            db.session_events.create(
                session_id:      session.id,
                event_type:      "AssistantMessage",
                transcript_uuid: uuid,
                payload:         { text: extractText(entry) },
                created_at:      parseTimestamp(entry) or now()
            )

    session.transcript_offset = currentOffset
    session.save()
```

**Add** `transcript_uuid` **to your events table** for deduplication:

```
ALTER TABLE session_events ADD COLUMN transcript_uuid VARCHAR(255);
CREATE UNIQUE INDEX idx_events_transcript_uuid
    ON session_events(transcript_uuid) WHERE transcript_uuid IS NOT NULL;
```

#### When to Parse

Call `parseNewEntries` in two places:

1.  **In every hook handler, before recording the hook event.** This ensures transcript-sourced `AssistantMessage` events appear in the timeline _before_ the tool calls they triggered. Without this, your timeline would show `PostToolUse: Edit` with no preceding assistant message explaining what's being edited.
    
2.  **On session detail page load.** The final assistant message after the last `Stop` may not have been parsed yet (no subsequent hook triggered parsing). Parsing on page load catches these.
    

#### Partial Line Handling

This is the subtlety that will bite you if you miss it. Claude Code writes to the transcript file while streaming a response. The last line in the file may be incomplete JSON -- the write hasn't finished yet.

Your parser must:

1.  Attempt to parse each line as JSON
    
2.  If parsing fails, **stop immediately**
    
3.  Set the offset to the position **before** the failed line, not after it
    
4.  On the next parse call, that line will be complete and parse successfully
    

If you advance the offset past a failed line, you'll permanently skip that entry.

#### What to Extract

From each assistant message entry, you need:

-   The text content (the actual response the user saw)
    
-   A unique identifier for deduplication
    
-   A timestamp
    

Ignore streaming chunks (entries without a `stop_reason` or equivalent finality marker). You want the final consolidated message, not every incremental token.

### Event Grouping

With transcript parsing, your timeline now has `AssistantMessage` events interleaved with hook events. But a flat chronological list is still hard to read. A single assistant turn might generate 15 tool calls -- you want to see "Assistant said X, then made 15 tool calls" as a collapsible group, not 16 separate entries.

#### The Grouping Model

Each assistant response starts a **group**. All tool calls and other events within that turn are **children** of that group. The assistant message is the **leader**.

**Add a group column to events and a tracking column on sessions:**

```
ALTER TABLE session_events ADD COLUMN message_group_id VARCHAR(255);
ALTER TABLE sessions ADD COLUMN current_group_uuid VARCHAR(255);
```

#### Assigning Groups During Parsing

When the transcript parser creates an `AssistantMessage` event, it starts a new group:

```
-- Inside parseNewEntries, when creating an AssistantMessage:
groupUuid = generateUUID()
event.message_group_id = groupUuid
session.current_group_uuid = groupUuid
```

When hook handlers record events, they inherit the session's current group:

```
-- Inside recordEvent (called by all hook handlers):
db.session_events.create(
    ...
    message_group_id: session.current_group_uuid,
    ...
)
```

This works because of the parse order: `parseNewEntries` runs first in every hook handler, updating `current_group_uuid` before the hook event is recorded.

#### The Timing Problem

Here's where it gets tricky. Hook events and transcript entries have an inherent race condition:

1.  Claude generates a response (written to transcript)
    
2.  Claude calls a tool (fires `PostToolUse` hook)
    
3.  Your hook handler runs `parseNewEntries`, reads the assistant message, creates the group
    
4.  Your hook handler records the `PostToolUse` event with the new group ID
    

This usually works. But sometimes the transcript write lags behind the hook, so step 3 doesn't find the new message yet. The `PostToolUse` gets recorded with the _previous_ group's UUID or no group at all.

#### The Fixup Pass

To handle this, run a fixup pass on page load that reassigns ungrouped events to the correct group based on turn boundaries.

The logic:

```
function fixupGroups(session):
    -- Only process events that should be grouped but aren't
    ungrouped = session.events
        .where(message_group_id IS NULL)
        .where(event_type NOT IN ("SessionStart", "SessionEnd", "UserPromptSubmit"))
        .orderBy(created_at)

    if ungrouped is empty:
        return

    -- Get all assistant messages (group leaders) ordered by time
    leaders = session.events
        .where(event_type = "AssistantMessage")
        .orderBy(created_at)

    if leaders is empty:
        return

    -- Get user message timestamps to define turn boundaries
    userMessageTimes = session.events
        .where(event_type = "UserPromptSubmit")
        .orderBy(created_at)
        .pluck(created_at)

    -- For each ungrouped event, find which turn it belongs to,
    -- then assign it to the last leader in that turn
    for each event in ungrouped:
        turnIndex = 0
        for each i, userTime in userMessageTimes:
            if event.created_at >= userTime:
                turnIndex = i + 1

        -- Find the last leader in this turn
        bestLeader = null
        for each leader in leaders:
            leaderTurn = 0
            for each i, userTime in userMessageTimes:
                if leader.created_at >= userTime:
                    leaderTurn = i + 1
            if leaderTurn == turnIndex:
                bestLeader = leader

        if bestLeader is not null:
            event.message_group_id = bestLeader.message_group_id
            event.save()
```

**Why turn boundaries?** A single session has multiple back-and-forth exchanges. Without turn boundaries, a late-arriving tool call from turn 3 might get assigned to the leader in turn 5. User prompt timestamps create natural dividers -- everything between prompt N and prompt N+1 belongs to the same turn.

**Call fixup after parsing on page load:**

```
function showSessionDetail(sessionId):
    session = db.sessions.find(sessionId)
    parseNewEntries(session)
    fixupGroups(session)
    events = session.events.orderBy(created_at)
    -- render page
```

### Session Detail Page

Now you have everything needed for a proper timeline view.

#### Building Grouped Entries

Transform the flat event list into a grouped structure for rendering:

```
function buildTimeline(events):
    groups = ordered map
    ungrouped = []

    -- First pass: identify leaders and initialize groups
    for each event in events:
        if event.event_type == "AssistantMessage" and event.message_group_id:
            groups[event.message_group_id] = {
                leader: event,
                children: []
            }

    -- Second pass: assign children to their groups
    for each event in events:
        if event is a leader:
            continue
        if event.message_group_id and event.message_group_id in groups:
            -- Skip Stop events as children -- they duplicate the
            -- AssistantMessage content and clutter the group
            if event.event_type == "Stop":
                continue
            groups[event.message_group_id].children.append(event)
        else:
            ungrouped.append(event)

    return { groups, ungrouped }
```

**Why two passes?** Due to the timing issues described above, a child event might appear in the event list _before_ its leader (the `PostToolUse` hook fires before the transcript is parsed). A single-pass approach would miss the group because the leader hasn't been seen yet. Two passes guarantee every child finds its leader regardless of ordering.

**Why exclude Stop events from children?** The `Stop` hook payload includes `last_assistant_message`, which is the same text as the `AssistantMessage` leader. Showing both is redundant. Store the `Stop` event (it's useful for debugging), but don't render it inside the group.

#### Rendering the Timeline

The page should show events in chronological order, with grouped events rendered as collapsible sections:

```
for each entry in timeline (chronological order):

    if entry is UserPromptSubmit:
        render user message bubble with prompt text

    if entry is a group leader (AssistantMessage):
        render assistant message bubble with text content
        if group has children:
            render collapsible toggle: "N tool calls"
            if expanded:
                for each child in group.children:
                    render tool call row: tool_name, summary of input
                    -- e.g., "Edit  app/Models/User.php"
                    -- e.g., "Bash  npm run test"

    if entry is SessionStart or SessionEnd:
        render subtle divider with timestamp
```

**Tool call display tips:**

-   For `Read` / `Edit` / `Write`: show the `file_path` from `tool_input`
    
-   For `Bash`: show the `command` from `tool_input` (truncate long commands)
    
-   For everything else: show the tool name and a one-line summary of input
    
-   Collapse tool input details behind a click -- most of the time you just want to see _which_ tools were called, not the full parameters
    

#### What the Detail Page Should Show

Beyond the timeline, the session header should display:

| Field | Source |
| --- | --- |
| Session name | `session.name` (will be null until Phase 3 adds auto-naming) |
| Project | Colored badge from `projects.name` and `projects.color` |
| Status | `active` / `idle` badge |
| Working directory | `session.cwd`, trimmed |
| Duration | `ended_at - started_at`, or "ongoing" |
| Event count | Total events in session |

### Updating the Session List

Now that sessions have projects, update the list page:

```
SELECT
    s.id,
    s.session_id,
    s.cwd,
    s.status,
    s.started_at,
    s.ended_at,
    p.name AS project_name,
    p.color AS project_color,
    COUNT(e.id) AS event_count
FROM sessions s
LEFT JOIN projects p ON p.id = s.project_id
LEFT JOIN session_events e ON e.session_id = s.id
GROUP BY s.id, p.name, p.color
ORDER BY s.updated_at DESC;
```

Add a project filter dropdown so you can scope the list to one client at a time.
