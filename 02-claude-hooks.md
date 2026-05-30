# Claude Hooks

## What Are Hooks?

Claude Code hooks are event-driven callbacks that fire at specific points during a Claude Code session. They let you push session data to external systems in real time -- no polling, no log scraping.

Hooks are configured in `~/.claude/settings.json` (global) or per-project in `.claude/settings.local.json`. Claude Code supports two hook transport types:

| Type | How it works |
| --- | --- |
| `http` | Claude Code makes an HTTP POST directly. You provide a URL and optional headers. |
| `command` | Claude Code runs a shell command. Payload is piped to stdin. |

**Use** `http` **whenever possible.** It's simpler, more reliable, and doesn't depend on `curl` or `jq` being installed. Use `command` only when you need to transform the payload or call a non-HTTP system.

## Available Hook Events

| Event | When it fires | Payload highlights |
| --- | --- | --- |
| `SessionStart` | Claude Code session begins | `session_id`, `cwd`, `transcript_path` |
| `UserPromptSubmit` | User sends a message | `session_id`, `cwd`, `prompt` |
| `PostToolUse` | After a tool executes | `session_id`, `tool_name`, `tool_input`, `cwd` |
| `Stop` | Assistant finishes a response | `session_id`, `last_assistant_message`, `cwd` |
| `SessionEnd` | Session closes | `session_id`, `cwd`, `transcript_path` |

Every event payload includes `session_id` and `cwd`. The `transcript_path` field points to the JSONL file on disk where the full conversation transcript is written incrementally.

## Configuration Example

This is what a working global configuration looks like using native HTTP hooks with bearer token auth:

```
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "http://your-app.test/api/hooks/session-start",
            "headers": {
              "Authorization": "Bearer your-secret-token"
            }
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "http",
            "url": "http://your-app.test/api/hooks/tool-use",
            "headers": {
              "Authorization": "Bearer your-secret-token"
            }
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "http://your-app.test/api/hooks/tool-use",
            "headers": {
              "Authorization": "Bearer your-secret-token"
            }
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "http://your-app.test/api/hooks/response-complete",
            "headers": {
              "Authorization": "Bearer your-secret-token"
            }
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "http://your-app.test/api/hooks/session-end",
            "headers": {
              "Authorization": "Bearer your-secret-token"
            }
          }
        ]
      }
    ]
  }
}
```

**Key points:**

-   Each event maps to an array of hook groups. Each hook group has a `hooks` array containing one or more transports.
    
-   `PostToolUse` supports an optional `matcher` field to filter by tool name (empty string = match all).
    
-   `UserPromptSubmit` and `PostToolUse` can share the same endpoint. Use the `hook_event_name` field in the payload to distinguish them server-side.
    

## Receiving Hooks Server-Side

### Authentication

Protect your endpoints with bearer token verification. The token in your hook config must match the token your server expects:

```
// Middleware: compare header token against env var
$token = config('services.claude_hooks.token'); // from CLAUDE_HOOKS_TOKEN env
if (!$token || $request->bearerToken() !== $token) {
    return response()->json(['error' => 'Unauthorized'], 401);
}
```

### Route Design

You don't need a 1:1 mapping between hook events and endpoints. A practical pattern is 4 routes:

| Endpoint | Handles |
| --- | --- |
| `POST /api/hooks/session-start` | `SessionStart` |
| `POST /api/hooks/tool-use` | `PostToolUse` and `UserPromptSubmit` |
| `POST /api/hooks/response-complete` | `Stop` |
| `POST /api/hooks/session-end` | `SessionEnd` |

`PostToolUse` and `UserPromptSubmit` share `/tool-use` because they're structurally similar -- both are mid-session events with a payload. The `hook_event_name` field in the JSON body tells you which is which.

### Payload Handling

Claude Code sends the full event as a JSON POST body. Your controller gets everything via `$request->all()`. Key fields to extract:

```
session_id        - Unique session identifier (string, UUID-like)
cwd               - Working directory where Claude Code is running
transcript_path   - Absolute path to the JSONL transcript file on disk
hook_event_name   - The event type string (e.g., "PostToolUse", "UserPromptSubmit")
tool_name         - Name of the tool (PostToolUse only, e.g., "Read", "Edit", "Bash")
tool_input        - Tool input parameters (PostToolUse only, object)
prompt            - The user's message text (UserPromptSubmit only)
last_assistant_message - Final assistant text (Stop only)
```

**Store the entire payload.** Fields vary between events and new fields may be added. Storing the full JSON lets you extract new data later without redeployment.

### Session Lifecycle

A typical session flows through these states:

```
SessionStart  -->  [UserPromptSubmit --> PostToolUse* --> Stop]*  -->  SessionEnd
```

Design your session tracking around `session_id` as the primary key:

1.  **SessionStart** -- Create or find the session. Set status to `active`.
    
2.  **Mid-session events** -- Record the event. Create the session if the start hook hasn't arrived yet (hooks can arrive out of order).
    
3.  **SessionEnd** -- Mark session `idle`. This is the right time to trigger post-processing (summaries, link extraction, etc.).
    

### Idempotency and Race Conditions

Things to watch for:

-   **Duplicate starts**: `SessionStart` can fire more than once for the same session (e.g., Claude resumes a session). Use `firstOrCreate` semantics on `session_id`.
    
-   **Out-of-order arrival**: A `PostToolUse` can arrive before `SessionStart` if the start hook is slow. Your tool-use handler should create the session if it doesn't exist.
    
-   **Transcript path backfill**: Not every event includes `transcript_path`. If a session was created without one, backfill it from the first event that provides it.
    
-   **Status protection**: If your app dispatches background jobs that set the session to `processing`, don't let a re-fired `SessionStart` overwrite that to `active`.
    

## The Transcript File

The `transcript_path` field points to a JSONL file that Claude Code writes incrementally during the session. Each line is a complete JSON object representing one API interaction.

This is separate from hooks -- hooks give you discrete events in real time, while the transcript gives you the full conversation history including the raw API responses and streaming chunks.

**Why use both?** Hooks tell you _what's happening now_ (tool calls, status changes). The transcript tells you _what was said_ (full assistant text, token usage). A complete logging system uses hooks as the real-time event stream and parses the transcript file for content extraction.

### Parsing the Transcript

The transcript file is written to while Claude is actively streaming. Key considerations:

-   **Read incrementally**: Track a byte offset per session. On each parse, read from the last offset. This avoids re-processing the entire file on every hook event.
    
-   **Handle partial lines**: The last line may be incomplete (still being written). If `json_decode` fails, stop parsing and preserve the offset before that line. The next parse will re-read it once complete.
    
-   **Deduplicate**: Use the `message.id` or unique identifiers in transcript entries to avoid creating duplicate records.
    

## Quick Start Checklist

1.  Pick a secret token for hook authentication
    
2.  Set it as an environment variable in your app (e.g., `CLAUDE_HOOKS_TOKEN`)
    
3.  Create 4 API endpoints behind bearer token middleware
    
4.  Add the `hooks` block to `~/.claude/settings.json` with `type: "http"`
    
5.  Store each event's full JSON payload in your database
    
6.  Handle session creation idempotently (out-of-order hooks, duplicate starts)
    
7.  Trigger post-processing (summaries, link extraction) on `SessionEnd`
