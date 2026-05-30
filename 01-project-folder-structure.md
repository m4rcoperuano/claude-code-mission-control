# Project Folder Structure

## Folder Structure

Your folder structure is key to getting Claude to work across various projects on your machine. Each client project gets its own `.claude` directory, which is where Claude reads project-specific settings, instructions, and skills.

For example:

```
~/Projects/
  SendJim/
    .claude/
      settings.local.json    # SendJim-specific tokens and permissions
      CLAUDE.md               # Project context, tech stack, conventions
      skills/
        jira-to-pr/SKILL.md
  ClientX/
    .claude/
      settings.local.json    # ClientX-specific tokens and permissions
      CLAUDE.md
      skills/
  Personal/
    .claude/
      settings.local.json
      CLAUDE.md
      skills/
```

Store your repositories in their respective client folder. As you get more complex tasks, claude may access one or more repositories inside that client folder.

### What each file does

| File | Purpose |
| --- | --- |
| `settings.local.json` | Client-specific configuration -- API tokens, environment variables, permission overrides. This is what isolates one client's credentials from another. |
| `CLAUDE.md` | Project instructions that Claude reads at the start of every session. Tech stack, conventions, key commands, architecture notes. This is how Claude "knows" the project. |
| `skills/` | Reusable multi-step workflows encoded as markdown. |

### Global vs. project-level

Your global config at `~/.claude/settings.json` applies to every session regardless of which project you're in. This is where hooks live (since you want session logging everywhere). Each project's `.claude/settings.local.json` overrides or extends the global config with client-specific values.

### Skills

Skills are markdown files that encode repeatable workflows. For example, a "Jira to PR" skill might encode:

1.  Read the Jira ticket for requirements
    
2.  Create and checkout a feature branch
    
3.  Implement the changes
    
4.  Run tests and lint
    
5.  Commit, push, and open a PR using the team's PR template
    

To create a skill, just ask Claude: _"Create a skill for our Jira-to-PR workflow."_ It will write the `SKILL.md` file into your project's `.claude/skills/` directory.
