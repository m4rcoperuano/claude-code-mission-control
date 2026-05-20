# Phase 3: Auto-Naming, Summaries, and Link Extraction in Session Management

_Source: https://sendjim.atlassian.net/wiki/spaces/EESVK/pages/497745921_

## Phase 3: Auto-Naming, Summaries, and Link Extraction

Phase 2 gave you readable session timelines. Phase 3 makes sessions _browsable_ -- you can scan a list of 50 sessions and quickly find the one where you fixed that auth bug or opened that PR, without clicking into each one.

### The Problem Phase 3 Solves

After a week of use, your session list looks like this:

| Status | Project | Events | Started |
| --- | --- | --- | --- |
| idle | SendJim | 47 | 2 hours ago |
| idle | SendJim | 12 | 3 hours ago |
| idle | ClientX | 89 | yesterday |
| idle | ClientX | 23 | yesterday |
| idle | Personal | 6 | 2 days ago |

Every row is "idle, some project, some events." You have to open each session and read the transcript to know what happened. This doesn't scale.

After Phase 3:

| Status | Project | Name | Summary | Links |
| --- | --- | --- | --- | --- |
| idle | SendJim | Fix Auth Token Refresh | Fixed expired token handling in OAuth middleware, added retry logic | PR #342 (merged) |
| idle | SendJim | Update User Seed Data | Updated company seeder with new test accounts |  |
| idle | ClientX | PROJ-891 Migrate Payment Gateway | Replaced Stripe v2 with v3 SDK across 12 files, updated webhook handlers | PR #156 (open), PROJ-891 |
| idle | ClientX | Debug Flaky E2E Tests | Investigated timeout issues in Playwright suite, added explicit waits |  |

### Auto-Naming

Sessions need names. Humans are bad at naming things proactively, so do it automatically.

#### When to Name

There are two opportunities:

1.  **Eagerly, when the first user message is sent.** If your app has a "send message" feature (Phase 4), name the session before Claude starts working. The user's prompt is the best signal for what the session is about. This gives you a name immediately.
    
2.  **On session end.** When `SessionEnd` fires, you have the full conversation. Dispatch a background job that checks whether the session still has no name, and if so, generates one.
    

Both use the same approach: send a short excerpt to a fast, cheap LLM and ask for a title.

#### Implementation

**Add a name column if you don't have one already:**

```
ALTER TABLE sessions ADD COLUMN name VARCHAR(255);
```

**The naming job:**

```
function generateSessionName(session):
    if session.name is not null:
        return  -- already named

    -- Get the first user prompt
    firstPrompt = session.events
        .where(event_type = "UserPromptSubmit")
        .orderBy(created_at)
        .first()

    text = firstPrompt.payload.prompt or firstPrompt.payload.message

    -- Fall back to first assistant message if no user prompt
    if text is empty or too short:
        firstAssistant = session.events
            .where(event_type = "AssistantMessage")
            .orderBy(created_at)
            .first()
        text = firstAssistant.payload.text

    if text is empty or too short (< 5 chars):
        return  -- not enough to name

    excerpt = first 500 characters of text

    name = callLLM(
        model: "claude-haiku-4-5-20251001",  -- fast and cheap
        maxTokens: 20,
        prompt: "Give this task a 3-4 word title. Reply with ONLY the title, no punctuation, no quotes.\n\nTask: {excerpt}"
    )

    if name:
        session.name = truncate(name, 60)
        session.save()
```

**Fallback when no API key is configured:**

```
-- Use the first line of the user's message, truncated
firstLine = first line of text, trimmed
if length > 60:
    name = first 57 chars + "..."
else:
    name = firstLine
```

#### Gotcha: Safety Prefixes

If your app prepends safety rules to user messages (like sunny-bots does to prevent destructive database commands), strip that prefix before passing the text to the naming LLM. Otherwise you'll get sessions named "Critical Safety Rules" instead of "Fix Auth Token Refresh."

```
-- Strip known prefix before extracting naming text
text = regex_replace(text, /^CRITICAL SAFETY RULES:.*?refuse and explain why\.\s*/s, "")
```

### Summaries

A name tells you _what_ the session was about. A summary tells you _what happened_.

**Add a summary column:**

```
ALTER TABLE sessions ADD COLUMN summary TEXT;
```

**The summary job (runs after naming, same background job):**

```
function generateSessionSummary(session):
    -- Collect all assistant message text
    text = session.events
        .where(event_type = "AssistantMessage")
        .orderBy(created_at)
        .pluck(payload.text)
        .join("\n\n")

    if text is too short (< 50 chars):
        return

    excerpt = first 4000 characters of text

    summary = callLLM(
        model: "claude-haiku-4-5-20251001",
        maxTokens: 150,
        prompt: "Summarize what was accomplished in this Claude Code session in 2-3 sentences. Be specific about files changed, features built, or bugs fixed. Reply with ONLY the summary.\n\nSession transcript excerpt:\n{excerpt}"
    )

    if summary:
        session.summary = summary
        session.save()
```

**Why assistant text and not user prompts?** The assistant messages describe what was actually done -- "I edited `UserController.php` to add retry logic" -- while user prompts describe what was requested, which may differ from what was accomplished.

**Why limit to 4000 characters?** Haiku is fast and cheap, but you're still paying per token. 4000 characters is enough context to capture what happened in most sessions without sending a novel. Long sessions with 50+ tool calls will have repetitive assistant text anyway.

#### Dispatching

Trigger both naming and summary generation from the same background job, dispatched on `SessionEnd`:

```
-- In your session-end handler, after updating status:
dispatchBackground(generateSessionNameAndSummary, session)
```

Or if your session-end handler already dispatches work, chain it:

```
function handleGenerateSessionSummary(session):
    generateSessionName(session)      -- name first (fast)
    generateSessionSummary(session)   -- then summarize
    -- then link extraction (below)
```

### Link Extraction

Sessions frequently reference external resources -- GitHub PRs, Jira tickets, commit URLs. Extracting these automatically creates a cross-reference between your sessions and your project management tools.

#### GitHub Links

**Add a JSON column for links:**

```
ALTER TABLE sessions ADD COLUMN github_links JSONB DEFAULT '[]';
```

**Scan session events for GitHub URLs:**

```
function extractGitHubLinks(session):
    seen = set()
    links = []

    -- Scan assistant messages and user prompts
    textEvents = session.events
        .where(event_type IN ("AssistantMessage", "UserPromptSubmit"))

    for each event in textEvents:
        text = event.payload.text or event.payload.prompt or ""
        for each match in findGitHubUrls(text):
            if match.url not in seen:
                seen.add(match.url)
                links.append(match)

    -- Also scan tool inputs (Bash commands often reference PRs)
    toolEvents = session.events.where(tool_input IS NOT NULL)

    for each event in toolEvents:
        text = stringify(event.tool_input)
        for each match in findGitHubUrls(text):
            if match.url not in seen:
                seen.add(match.url)
                links.append(match)

    session.github_links = links
    session.save()
```

**The URL parser:**

```
pattern = /https:\/\/github\.com\/([a-zA-Z0-9_.-]+)\/([a-zA-Z0-9_.-]+)
           (?:\/(pull|issues|commit)\/([a-zA-Z0-9]+))?/

function findGitHubUrls(text):
    matches = []
    for each match of pattern in text:
        owner = match[1]
        repo = match[2]
        type = match[3]  -- "pull", "issues", "commit", or null
        number = match[4]

        -- Build canonical URL from captured groups
        -- (the raw match may include trailing /files, /commits, etc.)
        url = "https://github.com/{owner}/{repo}"
        if type:
            url += "/{type}/{number}"

        matches.append({
            url: url,
            owner: owner,
            repo: repo,
            type: "pr" if type == "pull"
                  else "issue" if type == "issues"
                  else "commit" if type == "commit"
                  else "repo",
            number: int(number) if type in ("pull", "issues") else null,
            sha: number if type == "commit" else null
        })
    return matches
```

**Why canonicalize the URL?** Claude often references PRs with trailing paths like `https://github.com/org/repo/pull/123/files` or `https://github.com/org/repo/pull/123/commits`. You want to deduplicate these down to the base PR URL.

#### Jira Links

Same pattern, different regex:

```
ALTER TABLE sessions ADD COLUMN jira_links JSONB DEFAULT '[]';
```

```
pattern = /https:\/\/([a-zA-Z0-9_-]+)\.atlassian\.net\/browse\/([A-Z][A-Z0-9_]+-[0-9]+)/

function findJiraUrls(text):
    matches = []
    for each match of pattern in text:
        workspace = match[1]
        key = upper(match[2])    -- normalize "proj-123" to "PROJ-123"
        project = key.split("-")[0]
        number = int(key.split("-")[1])

        url = "https://{workspace}.atlassian.net/browse/{key}"

        matches.append({
            url: url,
            workspace: workspace,
            key: key,
            project: project,
            number: number
        })
    return matches
```

#### PR Status Enrichment (Optional But Valuable)

Extracted PR links are more useful if you know whether they're open, merged, or in draft. If you have GitHub API access, enrich PR links after extraction.

**Store a GitHub token per project:**

The cleanest place for this is in the project's `settings.local.json` (which you may already be storing in your projects table). Read the token from the project's settings:

```
token = session.project.settings_json.env.GITHUB_TOKEN
```

**Fetch status for each PR link:**

```
function refreshPrStatus(session):
    token = session.project?.settings_json?.env?.GITHUB_TOKEN
    if not token:
        return

    links = session.github_links
    updated = false

    for each link in links:
        if link.type != "pr":
            continue

        status = githubApi.getPullRequest(token, link.owner, link.repo, link.number)

        if status:
            link.pr_state = status.state      -- "open" or "closed"
            link.pr_merged = status.merged     -- boolean
            link.pr_draft = status.draft       -- boolean
            link.pr_title = status.title
            updated = true

    if updated:
        session.github_links = links
        session.save()
```

**Dispatch this after link extraction, but only if PR links exist:**

```
function handleGenerateSessionSummary(session):
    generateSessionName(session)
    generateSessionSummary(session)
    extractGitHubLinks(session)
    extractJiraLinks(session)

    hasPrLinks = session.github_links.any(link => link.type == "pr")
    if hasPrLinks:
        dispatchBackground(refreshPrStatus, session)
```

The PR status fetch is a separate background job because it makes external API calls that may be slow or rate-limited. Don't hold up naming and summary generation for it.

### Updating the Session List

Your list page can now show much more useful information:

```
SELECT
    s.id,
    s.name,
    s.status,
    s.summary,
    s.github_links,
    s.jira_links,
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

**Render links inline on the list page.** For GitHub PRs, show a small badge with the PR number and status (green for merged, yellow for open, gray for draft). For Jira tickets, show the ticket key as a clickable link. This lets you scan the session list and immediately see which sessions produced PRs and whether they've been merged.

### Choosing an LLM for Naming and Summaries

You need a model that's fast, cheap, and good enough at following simple instructions. You're making two calls per session end (name + summary) with small inputs and tiny outputs.

| Consideration | Recommendation |
| --- | --- |
| Model | Claude Haiku (or equivalent fast/cheap tier from any provider) |
| Latency | < 2 seconds per call |
| Cost | Negligible -- under 1000 input tokens per call |
| Failure handling | Log the error, skip naming/summary. Don't retry aggressively. |

**Important: use a separate API key from your Claude Code CLI.** If your CLI authenticates via OAuth subscription (the default for Claude Code), and you give your dashboard app an `ANTHROPIC_API_KEY` environment variable, the CLI subprocess might pick it up and switch auth methods. Name your env var something else (e.g., `SUNNY_BOTS_API_KEY`, `DASHBOARD_API_KEY`) to avoid this.
