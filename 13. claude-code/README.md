# Lesson 13: Using Claude Code efficiently

[Claude Code](https://code.claude.com/docs) is Anthropic's AI coding agent. It runs in your terminal (and in VS Code, JetBrains, a desktop app and the web). You describe a task in plain English, and Claude reads files, runs commands, edits code and reports back.

This lesson covers:

1. [Installation](#1-installation)
2. [Your first session](#2-your-first-session)
3. [`/init` and CLAUDE.md](#3-init-and-claudemd)
4. [Memory](#4-memory)
5. [Skills](#5-skills)
6. [Plugins and the plugin manager](#6-plugins-and-the-plugin-manager)
7. [Connectors (MCP): Jira, GitHub and more](#7-connectors-mcp-jira-github-and-more)
8. [Running terminal commands and scripts](#8-running-terminal-commands-and-scripts)
9. [Letting Claude keep working: `/goal` and `/loop`](#9-letting-claude-keep-working-goal-and-loop)
10. [How Claude Code consumes tokens](#10-how-claude-code-consumes-tokens)
11. [`/clear`, `/compact` and other context commands](#11-clear-compact-and-other-context-commands)
12. [Subagents](#12-subagents)
13. [Checklist: an efficient session](#13-checklist-an-efficient-session)
14. [Bonus: cutting cost in your own Claude API apps](#14-bonus-cutting-cost-in-your-own-claude-api-apps)
15. [Cheat sheet](#15-cheat-sheet)

The `examples` folder has things you can try: a skill ([section 5](#5-skills)), a plugin ([section 6](#6-plugins-and-the-plugin-manager)), and a safety hook plus a Python script that calls Claude ([section 8](#8-running-terminal-commands-and-scripts)).

---

## 1. Installation

You need a paid Claude account (Pro, Max, Team or Enterprise) or a [Claude Console](https://console.anthropic.com/) account with API credits. The free plan does not include Claude Code.

**System requirements:** macOS 13+, Windows 10 (1809+) or Windows Server 2019+, Ubuntu 20.04+, Debian 10+ or Alpine 3.19+, and at least 4 GB of RAM.

### Native installer (recommended)

It needs no Node.js and updates itself in the background.

- **macOS, Linux, WSL:**

  ```bash
  curl -fsSL https://claude.ai/install.sh | bash
  ```

- **Windows PowerShell:**

  ```powershell
  irm https://claude.ai/install.ps1 | iex
  ```

- **Windows CMD:**

  ```bat
  curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
  ```

On native Windows, also install [Git for Windows](https://git-scm.com/downloads/win). It is optional, but it gives Claude Git Bash to run commands in. Without it, Claude uses PowerShell.

### Other ways to install

| Method | Command | Updates |
|---|---|---|
| Homebrew (macOS) | `brew install --cask claude-code` | Manual: `brew upgrade claude-code` |
| WinGet (Windows) | `winget install Anthropic.ClaudeCode` | Manual: `winget upgrade Anthropic.ClaudeCode` |
| npm (needs Node.js 22+) | `npm install -g @anthropic-ai/claude-code` | Manual: `claude update` |

Do not use `sudo` with npm.

### Check the install

```bash
claude --version   # prints the version
claude doctor      # checks your setup without starting a session
claude update      # updates now
```

---

## 2. Your first session

Go into a project folder and start Claude:

```bash
cd ai-llm-course
claude
```

The first time, a browser window opens so you can log in. Pick your Claude account (subscription), or run `claude auth login --console` to bill an API key from the Console instead.

Now type a request, for example:

```
> explain what 11. MultiAgent/4_supervisor_agent.py does
```

A few keys you will use all the time:

| Key | What it does |
|---|---|
| `@` | Mention a file, e.g. `@README.md`. Its content is attached to your message. |
| `!` | Run a shell command yourself, e.g. `!git status`. The output goes into the conversation. |
| `Shift+Tab` | Cycle permission modes: normal, auto-accept edits, and **plan mode** (Claude plans first and edits nothing until you approve). |
| `Esc` | Stop Claude mid-answer. |
| `Esc Esc` | On an empty prompt: open the rewind menu to go back to an earlier point. |
| `/` | Show all commands. `/help` explains them. |

To pick up where you left off:

```bash
claude --continue          # or: claude -c   (most recent session in this folder)
claude --resume            # or: claude -r   (choose from a list)
```

To ask a single question without opening a session (useful in scripts):

```bash
claude -p "list the lessons in this repo"
cat error.log | claude -p "what caused this error?"
```

---

## 3. `/init` and CLAUDE.md

Every session starts with no memory of your project. **CLAUDE.md** is a Markdown file Claude reads at the start of every session. Put in it what a new teammate would need: how to install, how to run tests, code style, things to avoid.

Run this inside a session:

```
> /init
```

`/init` scans the codebase and writes a starter `CLAUDE.md` with the build and test commands and conventions it finds. If a CLAUDE.md already exists, it suggests improvements instead of overwriting it. Read the result and edit it. You know things the scan cannot.

Example `CLAUDE.md` for this course:

```markdown
# AI LLM Course

- Python 3, packages in requirements.txt, virtual env in ./venv
- Each numbered folder is one lesson. Scripts run on their own: `python "11. MultiAgent/1_tool_calling_agent.py"`
- API keys live in .env in the project root. Never print or commit them.
- Keep examples short and beginner-friendly. Comment the "why", not the "what".
```

### Where CLAUDE.md files can live

All of these load at startup and are combined:

| File | Scope | Shared through git? |
|---|---|---|
| `~/.claude/CLAUDE.md` | You, in every project | No |
| `./CLAUDE.md` or `./.claude/CLAUDE.md` | This project | Yes |
| `./CLAUDE.local.md` | You, in this project only | No (add it to `.gitignore`) |
| `subfolder/CLAUDE.md` | Loaded when Claude works in that folder | Yes |

Your company can also set a managed CLAUDE.md that applies to everyone.

You can pull other files in with `@`:

```markdown
See @README.md for setup and @docs/style.md for code style.
```

**Keep CLAUDE.md short (under about 200 lines).** It is sent with *every* request, so every line costs tokens on every turn (see [section 10](#10-how-claude-code-consumes-tokens)). Put long, task-specific instructions into a [skill](#5-skills) instead. Skills load only when needed.

---

## 4. Memory

Claude Code has two kinds of memory:

1. **CLAUDE.md files** (section 3). You write them.
2. **Auto memory.** Claude writes it. When it learns something worth keeping (your preferences, a correction you gave, a project decision), it saves a short note. It is on by default.

Auto memory is stored per project in:

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # index, loaded at the start of every session
└── <topic>.md         # one note per fact
```

Notes come in four types: **user** (who you are), **feedback** (how you want Claude to work), **project** (goals, decisions, deadlines) and **reference** (links to dashboards, tickets and docs).

### The `/memory` command

```
> /memory
```

It opens a menu where you can:

- open any of your CLAUDE.md files in your editor,
- turn auto memory on or off,
- browse what Claude has saved.

You can also just ask:

```
> remember that I always want type hints in Python code
> forget the note about the old deploy script
```

**Tip:** Check your memory files now and then. Delete notes that are wrong or out of date. They are loaded into every session, just like CLAUDE.md.

---

## 5. Skills

A **skill** is a packaged set of instructions for one kind of task, such as "review a PR", "write a release note" or "explain a file to a beginner". It is a folder with a `SKILL.md` file.

Why skills instead of putting everything in CLAUDE.md? **Progressive disclosure.** At startup Claude only sees each skill's short `description`. The full instructions load only when the skill is used. So you can have dozens of skills without paying for them on every turn.

### Where skills live

| Location | Available in |
|---|---|
| `~/.claude/skills/<name>/SKILL.md` | All your projects |
| `.claude/skills/<name>/SKILL.md` | This project (commit it to share with your team) |
| `<plugin>/skills/<name>/SKILL.md` | Wherever the plugin is installed |

### SKILL.md format

```markdown
---
name: explain-code
description: Explain a Python file to a beginner, step by step. Use when the user asks what a file does or how it works.
argument-hint: <file>
allowed-tools: Read
---

Explain the file $ARGUMENTS to someone who is new to Python and LLMs.

1. One sentence: what does this script do?
2. Walk through it top to bottom in small chunks...
```

The part between the `---` lines is the **frontmatter**. The useful fields:

| Field | Meaning |
|---|---|
| `name` | The command name: `/explain-code` |
| `description` | When to use the skill. Claude reads this to decide whether to load it on its own. Make it specific. |
| `argument-hint` | Shown in autocomplete, e.g. `<file>` |
| `allowed-tools` | Tools the skill may use without asking you |
| `disable-model-invocation: true` | Only you can run it (Claude never triggers it on its own). Good for deploys. |
| `model`, `effort` | Run the skill with a specific model or effort level |

Inside the body, `$ARGUMENTS` is replaced with whatever you type after the command.

### Using a skill

Two ways:

- **You call it:** `/explain-code "11. MultiAgent/4_supervisor_agent.py"`
- **Claude calls it:** you ask "what does the supervisor agent file do?" and Claude matches it to the skill's description.

Useful commands:

```
> /skills          # list all skills you have
> /skill-doctor    # how much context each skill costs and how often it is used
```

> Older Claude Code versions used **custom slash commands** in `.claude/commands/*.md`. They still work, but skills replaced them. If both have the same name, the skill wins.

### Try it

This lesson has the skill above in [`examples/.claude/skills/explain-code/SKILL.md`](examples/.claude/skills/explain-code/SKILL.md). Start Claude from the `examples` folder so it finds the skill:

```bash
cd "13. claude-code/examples"
claude
```

```
> /explain-code ../11.\ MultiAgent/1_tool_calling_agent.py
```

Then try `/skills` and find `explain-code` in the list.

---

## 6. Plugins and the plugin manager

A **plugin** bundles skills, subagents, hooks and MCP servers into one installable package. Plugins come from **marketplaces**, which are git repos (or local folders) that list plugins.

### The plugin manager

```
> /plugin
```

This opens the plugin manager. It has four tabs:

| Tab | What you do there |
|---|---|
| **Discover** | Browse and install plugins from your marketplaces |
| **Installed** | Enable, disable, update or uninstall plugins |
| **Marketplaces** | Add or remove marketplaces |
| **Errors** | See plugins that failed to load |

Anthropic's official marketplace, **`claude-plugins-official`**, is added automatically the first time you start Claude Code. Its plugins are not installed until you choose them.

### Examples

Install a plugin from the official marketplace:

```
> /plugin install commit-commands@claude-plugins-official
```

The format is `plugin-name@marketplace-name`. Claude Code asks for the **scope**:

- **user** (default): you, in every project
- **project**: everyone working on this repo (saved in `.claude/settings.json`)
- **local**: you, in this repo only

Add another marketplace, for example Anthropic's demo marketplace on GitHub:

```
> /plugin marketplace add anthropics/claude-code
```

A marketplace can also be a local folder (`./my-marketplace`) or any git URL.

Turn plugins off and on, or remove them:

```
> /plugin disable commit-commands
> /plugin enable commit-commands
> /plugin uninstall commit-commands
```

The same commands work from the shell, outside a session:

```bash
claude plugin marketplace add anthropics/claude-code
claude plugin install commit-commands@claude-plugins-official --scope project
claude plugin list
claude plugin disable commit-commands@claude-plugins-official
claude plugin uninstall commit-commands@claude-plugins-official
```

Plugin skills are namespaced by the plugin name: `/plugin-name:skill-name`.

**Install only what you use.** Every enabled plugin adds its skill descriptions, tools and MCP servers to the startup context, and that costs tokens on every request (see [section 10](#10-how-claude-code-consumes-tokens)).

### How a plugin is built

```
course-tools/                     # plugin root
├── .claude-plugin/
│   └── plugin.json               # manifest (only this file goes here)
├── skills/
│   └── lesson-summary/
│       └── SKILL.md
├── agents/                       # optional: subagents
├── hooks/hooks.json              # optional: hooks
└── .mcp.json                     # optional: MCP servers
```

`plugin.json` needs only a `name`:

```json
{
  "name": "course-tools",
  "description": "Helpers for the AI LLM course",
  "version": "1.0.0"
}
```

A marketplace is a folder with `.claude-plugin/marketplace.json` that lists plugins:

```json
{
  "name": "ai-llm-course",
  "owner": { "name": "Higgs Labs" },
  "plugins": [
    {
      "name": "course-tools",
      "source": "./plugins/course-tools",
      "description": "Helpers for the AI LLM course"
    }
  ]
}
```

### Try it

[`examples/plugin-marketplace`](examples/plugin-marketplace) is a small marketplace with one plugin, `course-tools`, which has one skill, `lesson-summary`.

**Option A: load the plugin for one session only** (good while you build a plugin):

```bash
claude --plugin-dir "13. claude-code/examples/plugin-marketplace/plugins/course-tools"
```

```
> /course-tools:lesson-summary 10. LangGraph
```

**Option B: install it through the plugin manager**, as a user would. Start `claude` from the repo root, then:

```
> /plugin marketplace add "./13. claude-code/examples/plugin-marketplace"
> /plugin install course-tools@ai-llm-course
> /course-tools:lesson-summary 11. MultiAgent
```

Open `/plugin` and look at the **Installed** and **Marketplaces** tabs. Uninstall it when you are done:

```
> /plugin uninstall course-tools
> /plugin marketplace remove ai-llm-course
```

---

## 7. Connectors (MCP): Jira, GitHub and more

Out of the box, Claude Code sees your files and your terminal. **Connectors** let it reach other tools: read a Jira ticket, open a GitHub pull request, check an error in Sentry, search your Notion docs.

Connectors use the **Model Context Protocol (MCP)**, an open standard. A connector is an **MCP server** that gives Claude a set of tools, such as `get_issue`, `create_pull_request` or `search_pages`.

There are two kinds of MCP server:

| Kind | How it runs | Typical login | Example |
|---|---|---|---|
| **Remote** (`http`) | A URL hosted by the vendor | OAuth in the browser, or a token | Jira, Notion, Sentry, GitHub |
| **Local** (`stdio`) | A program on your computer that Claude Code starts | API key in an environment variable | Playwright (browser control), database tools |

### Adding a connector

From the shell (not inside a session):

```bash
# remote server
claude mcp add --transport http <name> <url>

# remote server that needs a token in a header
claude mcp add --transport http <name> <url> --header "Authorization: Bearer <token>"

# local server: everything after -- is the command that starts it
claude mcp add <name> -e API_KEY=<key> -- npx -y <package>
```

Then start `claude` and run `/mcp`. It shows every server and whether it is connected. Pick a server that says it needs authentication and log in through the browser.

The `--scope` flag decides who gets the connector:

| Scope | Who gets it | Stored in |
|---|---|---|
| `local` (default) | You, in this project only | `~/.claude.json` |
| `project` | Everyone who works on this repo | `.mcp.json` in the project root (commit it) |
| `user` | You, in all your projects | `~/.claude.json` |

### Common connectors

**Jira and Confluence (Atlassian).** One server covers both:

```bash
claude mcp add --transport http atlassian https://mcp.atlassian.com/v2/mcp
```

Run `/mcp`, pick `atlassian` and log in with your Atlassian account. Then:

```
> summarize PROJ-123 and list the open questions in the comments
> find all bugs assigned to me in the current sprint
> create a Jira task in PROJ: "Add retry logic to the Tavily search tool"
```

**GitHub.** GitHub's server takes a personal access token. Create one at GitHub → Settings → Developer settings → Personal access tokens, then:

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer $GITHUB_PAT"
```

```
> list the open PRs in higgs-labs/ai-llm-course that are waiting for review
> read the comments on PR #12 and fix what the reviewer asked for
```

> **Tip:** if you have the [GitHub CLI](https://cli.github.com/) (`gh`) installed and logged in, Claude can run `gh pr list`, `gh issue view 12` and similar commands directly in the terminal. You don't need the MCP server for that, so its tools don't take up any context.

**Notion and Sentry.** Both use OAuth. Add them, then log in with `/mcp`:

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
```

**Playwright (local).** Lets Claude open a browser, click around and take screenshots, for example to test a web app it just built:

```bash
claude mcp add playwright -- npx @playwright/mcp@latest
```

Connectors work best together. One prompt can use several:

```
> read PROJ-123 in Jira, implement it, open a GitHub PR, and add the PR link as a comment on the ticket
```

### Connectors from claude.ai

If you log in to Claude Code with a claude.ai account, the connectors you added on claude.ai (Settings → Connectors) show up in Claude Code automatically. They appear in `/mcp` with names like `claude.ai Atlassian`. You can disable any of them there.

### Sharing connectors with your team: `.mcp.json`

`claude mcp add --scope project ...` writes a `.mcp.json` file in the project root. You can also write it by hand:

```json
{
  "mcpServers": {
    "atlassian": {
      "type": "http",
      "url": "https://mcp.atlassian.com/v2/mcp"
    },
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": { "Authorization": "Bearer ${GITHUB_PAT}" }
    }
  }
}
```

`${GITHUB_PAT}` is read from each person's environment, so no one commits a token. The first time a teammate opens the project, Claude Code asks them to approve these servers.

### Using and managing connectors

Usually you just ask in plain English and Claude picks the right tool. There are two other ways to use them:

- **Resources:** type `@` and pick a resource a server offers, e.g. `@github:...`. It is attached like a file.
- **Prompts:** some servers offer ready-made prompts. They show up as commands: `/mcp__<server>__<prompt>`.

To manage connectors from the shell:

```bash
claude mcp list            # all servers and whether they connect
claude mcp get github      # details of one server
claude mcp remove github   # delete it
```

In a session, `/mcp` lets you log in or out, reconnect, and **disable** a server for this project without deleting it.

### Connectors and tokens

- **Tool definitions:** Claude Code loads MCP tool definitions only when they are needed (*tool search*), so having servers connected costs little up front. Still, disable servers you are not using with `/mcp`.
- **Tool results:** these are the expensive part. A big Jira search or a long GitHub diff goes into the context and is resent on every turn. Ask for what you need ("the 5 newest bugs", not "all bugs"). Claude Code warns when one MCP result is over 10,000 tokens and cuts it off at 25,000 (`MAX_MCP_OUTPUT_TOKENS` changes this).

### Safety

- Only add servers from vendors you trust. An MCP server can run code and see what you send it.
- Text that comes back from a connector (a Jira comment, a PR description) is data written by other people. It can contain instructions meant to trick Claude. Check what Claude does before you approve writes such as closing tickets or merging PRs.
- Prefer read-only tokens when you only need to read.

---

## 8. Running terminal commands and scripts

This goes both ways. Claude can run commands in your terminal (tests, Python scripts, git), and you can run Claude from your own scripts.

### Claude runs commands for you

When Claude needs to run something, it uses its **Bash tool**. (On native Windows without Git Bash it uses PowerShell.) For example:

```
> run "11. MultiAgent/1_tool_calling_agent.py" and fix any error
```

Claude runs `python "11. MultiAgent/1_tool_calling_agent.py"`, reads the output and the error, edits the file, and runs it again until it works.

By default Claude **asks before running a command**. You can allow it once, allow it for the rest of the session, or say no and tell it what to do instead.

### Permission modes

`Shift+Tab` switches the mode during a session. You can also start in a mode with `claude --permission-mode <mode>`.

| Mode | What Claude may do without asking |
|---|---|
| `default` | Read files. Asks before edits and commands. |
| `acceptEdits` | Also edit files and run simple file commands (`mkdir`, `mv`, `cp`) in the project |
| `plan` | Nothing. Read-only: it explores and writes a plan for you to approve |
| `auto` | Most things. A safety check runs in the background and stops risky actions |
| `bypassPermissions` | Everything. Only use it in a throwaway container or VM. (`--dangerously-skip-permissions` is the same) |

### Permission rules: stop answering the same prompt

If you approve `pytest` twenty times a day, write a rule instead. Run `/permissions`, or edit `.claude/settings.json` in the project:

```json
{
  "permissions": {
    "allow": ["Bash(python *)", "Bash(pytest *)", "Bash(git diff *)", "Bash(git status)"],
    "ask":   ["Bash(pip install *)"],
    "deny":  ["Bash(git push *)", "Read(./.env)"]
  }
}
```

- `*` is a wildcard: `Bash(git diff *)` allows any `git diff ...` command, but not `git push`.
- **deny wins over allow.** `Read(./.env)` stops Claude from reading your API keys.
- Rules apply to each part of a chained command. `git status && git push` still hits the `git push` deny rule.
- `.claude/settings.json` is shared through git. Put personal rules in `.claude/settings.local.json` or `~/.claude/settings.json`.

### Long-running commands and the background

A command is stopped after **2 minutes** by default (Claude can raise this to 10 minutes). For things that keep running, such as a web server, ask Claude to run it in the background:

```
> start `adk web` in "12. ADK" in the background, then check that http://127.0.0.1:8000 responds
```

You can also press `Ctrl+B` to move a command that is already running into the background. `/tasks` shows the background tasks and lets you stop them.

### Running a command yourself: `!`

Start your message with `!` to run a command yourself, without Claude deciding anything:

```
> !python "10. LangGraph/1_simple_graph.py"
```

The output is added to the conversation, so you can follow up with "why did it fail?". Remember that this output is resent on every turn like everything else. Don't `!cat` a huge log file.

### Sandbox

`/sandbox` turns on an OS-level sandbox for commands Claude runs. It limits which files they can write and which network hosts they can reach, so Claude can run more commands without asking. It works on macOS, Linux and WSL2.

### Hooks: block dangerous commands

A **hook** is a script Claude Code runs at a fixed moment, such as *before every tool call*. A `PreToolUse` hook on `Bash` sees each command before it runs. **If the hook exits with code 2, the command is blocked**, and whatever the hook printed to stderr is shown to Claude.

[`examples/.claude/hooks/block-dangerous.sh`](examples/.claude/hooks/block-dangerous.sh) blocks `rm -rf` and `git push --force`:

```bash
#!/bin/bash
# Claude Code sends the tool call as JSON on stdin.
cmd=$(jq -r '.tool_input.command')

if echo "$cmd" | grep -qE 'rm -rf|git push (-f|--force)'; then
  echo "Blocked by hook: '$cmd' is not allowed in this project." >&2
  exit 2
fi
exit 0
```

It is turned on in [`examples/.claude/settings.json`](examples/.claude/settings.json). That file also has a deny rule so Claude can't read `.env`. The hook part:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-dangerous.sh" }
        ]
      }
    ]
  }
}
```

Try it (needs `jq`; on macOS: `brew install jq`):

```bash
cd "13. claude-code/examples"
claude
```

```
> create a folder called tmp-test, then delete it with rm -rf
```

Claude creates the folder, but the hook stops the `rm -rf`. Use `/hooks` to see the hooks that are active.

### Running Claude from your scripts: `claude -p`

`-p` (print mode) runs one task with no chat window and prints the answer. This lets you use Claude inside shell scripts, Python programs, cron jobs and CI.

```bash
# a single question
claude -p "which lessons in this repo use LangGraph?"

# pipe data in
git diff | claude -p "review this diff for bugs. Be brief."
python "11. MultiAgent/3_tool_error_handling.py" 2>&1 | claude -p "explain this error"
```

Useful flags:

| Flag | What it does |
|---|---|
| `--output-format json` | Prints JSON: the answer is in `result`. It also has `session_id`, `total_cost_usd`, `num_turns` and `is_error` |
| `--allowedTools "Read,Bash(git diff *)"` | Tools it may use without asking. In `-p` mode there is no one to ask, so anything else is refused |
| `--permission-mode acceptEdits` | Let it edit files |
| `--model haiku` | Pick a cheaper model for simple jobs |
| `--max-budget-usd 0.50` | Stop if the run's estimated cost goes over this amount |
| `--append-system-prompt "..."` | Add your own instructions |
| `--resume <session_id>` | Continue an earlier run with its context |
| `--bare` | Skip CLAUDE.md, skills, plugins, MCP and memory. Faster and cheaper for scripts. Needs `ANTHROPIC_API_KEY` |

Read the JSON output with `jq`:

```bash
claude -p "summarize README.md in one line" --output-format json | jq -r '.result, .total_cost_usd'
```

Continue a run in a second step:

```bash
id=$(claude -p "read 10. LangGraph/1_simple_graph.py" --output-format json | jq -r '.session_id')
claude -p "now add a third node to it" --resume "$id" --permission-mode acceptEdits
```

The exit code is 0 on success and non-zero on failure, so scripts can check it.

### From Python

[`examples/scripts/ask_claude.py`](examples/scripts/ask_claude.py) wraps `claude -p` in a Python function and uses it to summarize a lesson folder:

```python
def ask_claude(prompt, allowed_tools="Read,Glob", model="haiku", max_budget_usd=0.25):
    cmd = [
        "claude", "-p", prompt,
        "--output-format", "json",
        "--allowedTools", allowed_tools,
        "--model", model,
        "--max-budget-usd", str(max_budget_usd),
    ]
    completed = subprocess.run(cmd, capture_output=True, text=True, stdin=subprocess.DEVNULL)
    ...
    return json.loads(completed.stdout)
```

Run it from the repo root (it uses your Claude account, so each run costs a little):

```bash
python "13. claude-code/examples/scripts/ask_claude.py" "10. LangGraph"
```

`claude -p` is the quick way to script Claude Code. If you want to build a full agent in Python with Claude Code's tools, look at the [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview).

### In GitHub Actions

Claude can also run in CI. Inside a session in your repo, run:

```
> /install-github-app
```

It installs the Claude GitHub App, stores your key as a repo secret, and opens a PR that adds a workflow. After you merge it, write `@claude` in a PR or issue comment, for example `@claude fix the failing test`, and Claude works on it in GitHub Actions and replies there.

---

## 9. Letting Claude keep working: `/goal` and `/loop`

Normally Claude finishes a turn and waits for you. Two commands let it keep going without you typing:

| | `/goal` | `/loop` |
|---|---|---|
| You say | "Keep working until this is true" | "Do this again every N minutes" |
| It stops when | The condition is met, judged impossible, or you clear it | You stop it, the session closes, or after 7 days |
| Good for | Finishing a task: make the tests pass, fix every lint error | Watching something: CI, a deploy, a queue |

### `/goal`: work until a condition is met

```
> /goal python "11. MultiAgent/3_tool_error_handling.py" runs without errors, and no other file is changed
```

Claude starts working right away. After each turn, a separate small, fast model reads the conversation and decides whether the condition is met:

- **Not met:** Claude starts another turn on its own. The checker's reason tells it what is still missing.
- **Met:** the goal is cleared and Claude stops.
- **Impossible:** the goal is cleared and marked as failed.

Commands:

```
> /goal              # status: the condition, turns so far, token spend, the checker's last reason
> /goal clear        # stop and remove the goal (stop, off and cancel also work)
```

`/clear` removes the goal too. If you `--resume` the session, an active goal comes back.

**Writing a good condition.** The checker **only reads the conversation**. It does not run commands or open files. So the condition must be something Claude can *show*:

| Weak | Better |
|---|---|
| "the code works" | "`pytest -q` exits 0, shown by running it" |
| "fix the tests" | "all tests in `tests/auth` pass and no test file is edited" |
| "make it faster" | "`python bench.py` prints under 200 ms, or stop after 15 turns" |

- Make it **measurable**, and say **how it is checked**.
- Add **constraints** ("don't change other files").
- Add a **turn limit**, such as "or stop after 15 turns", so a stuck task doesn't run forever.

**Permissions.** `/goal` only removes the need to say "continue". Claude still asks before tool calls your settings don't allow. To let a goal run unattended, start it in auto mode (`Shift+Tab`) or allow the commands it needs with [permission rules](#permission-rules-stop-answering-the-same-prompt).

It also works in scripts:

```bash
claude -p "/goal CHANGELOG.md has an entry for every PR merged this week"
```

**Cost.** Each check runs on a small, fast model and costs very little. The real cost is the extra turns: every turn resends the whole conversation (see [section 10](#10-how-claude-code-consumes-tokens)). A turn limit in the condition is your budget.

### `/loop`: repeat on a schedule

```
> /loop 5m check whether the GitHub Actions run for my last push has finished, and tell me if it failed
```

| Form | Example | What happens |
|---|---|---|
| Fixed interval | `/loop 10m run pytest -q and report new failures` | Runs every 10 minutes |
| Run a skill or command | `/loop 30m /explain-code app.py` | Runs the skill each time |
| No interval (self-paced) | `/loop check the deploy status` | Claude picks the wait (1 minute to 1 hour) after each run and tells you why |
| Nothing at all | `/loop` | Runs the prompt in `.claude/loop.md` (or a built-in maintenance prompt) |

- Units: `s`, `m`, `h`, `d`. The smallest step is one minute, and odd intervals are rounded to a nearby one. Claude tells you what it picked.
- A loop can run only skills Claude is allowed to start on its own. Built-in commands such as `/clear` or `/model`, and skills with `disable-model-invocation: true`, arrive as plain text instead.

**Stopping and listing loops.** Press `Esc` to cancel the next run of a self-paced loop. For any loop, just ask:

```
> what scheduled tasks do I have?
> cancel the deploy check
```

**How long a loop lives:**

- **Only while the session is open** and Claude is idle. Closing the terminal stops it. Missed runs are not caught up.
- Repeating loops end automatically **7 days** after you create them.
- `--resume` brings back fixed-interval loops but not self-paced ones. Start those again.
- To run something when your computer is off, use **`/schedule`** instead. It creates a *routine* that runs in Anthropic's cloud on a fresh copy of your repo, at most once an hour.
- To turn off loops completely, start Claude with `CLAUDE_CODE_DISABLE_CRON=1`.

**Cost.** Every run is a full turn that carries **the whole conversation** so far. A loop started at the end of a long session resends that long session every 5 minutes. So:

- Start loops in a **fresh session**, in a separate terminal.
- Keep the loop prompt short, and ask for short reports ("reply 'no change' if nothing changed").
- Pick an interval that matches how fast the thing really changes. A CI run that takes 10 minutes needs one check after 10 minutes, not ten checks one minute apart.

---

## 10. How Claude Code consumes tokens

This is the most important section for using Claude Code efficiently.

A code editor with a flat fee makes every task feel free. With an agent like Claude Code, **each task has its own cost, and it depends on how you work.** One session fixes a bug in a few turns. Another greps and reads a dozen extra files, and every later turn carries all of them along.

Being efficient does not mean using as few tokens as possible. It means making sure the tokens you use serve the request.

On a subscription you do not pay per token, but the same tokens count against your usage limits. So everything below applies to you either way.

### What decides the price of a token

You are billed per token, but what you really pay for is GPU time. Three things decide the price:

**1. The model.** Bigger models cost more per token, and that multiplies everything else. Use a big model for hard or unclear problems and a smaller one for routine work.

**2. Input or output.**

- **Input (prefill):** what the model reads: the system prompt, tool definitions, CLAUDE.md, your messages, file contents, command output.
- **Output (decode):** what the model writes: its *thinking*, tool calls and answer. **Output costs about 5x as much as input per token.**

Much of the output is *thinking*. The **effort level** controls how much Claude thinks per turn (see [Choosing model and effort](#choosing-model-and-effort) below).

**3. Whether it was cached.** See [Prompt caching](#prompt-caching).

### Choosing model and effort

These two settings decide most of what a turn costs. **Effort** sets how hard Claude works: how much it thinks, how many checks it runs, how far it explores before answering.

```
> /effort            # show the current level and the options
> /effort medium     # set it
```

| Level | Use it for |
|---|---|
| `low` | Routine, well-defined work: rename a variable, fix a typo, write a docstring, run a known command |
| `medium` | Everyday coding: a small feature, a clear bug fix |
| `high` | Unclear bugs, changes across many files, design decisions |
| `xhigh`, `max` | The hardest problems, where a wrong answer is expensive. Not every model offers every level. |
| `auto` | Go back to the default |

Miscalibration costs you either way:

- **Too high:** Claude overthinks. You pay more and wait longer, and the answer is sometimes *worse*. Extra thinking only helps while there is something left to find.
- **Too low:** Claude stops early, skips checks and answers from partial information. You then pay for the follow-up turns that fix it.

What Anthropic measured (from the sources below):

- On a set of very hard coding tasks, max effort scored about **2.7x** higher than low effort but cost about **3.5x** more per task. On hard problems, more effort really does pay.
- On a knowledge test, going from low to max raised the score from about 53% to 61%. **The last step up to max added only half a point for 46% more cost.**
- A newer, stronger model at **low** effort matched the older model at **high** effort for about **one third** of the cost.

How to choose:

1. **Model first.** In `/model`, pick a bigger model for hard or unclear problems and a smaller, cheaper one (such as Haiku) for routine work. The model's price multiplies everything else.
2. **Then try a strong model at a lower effort** before raising the effort. It often costs less than a weaker model at high effort.
3. **Raise effort only when you see Claude stopping too early,** for example when it skips checks or misses obvious causes.
4. **Set both at the start of a session or right after `/clear`.** Changing either mid-conversation invalidates the cache (see [What breaks the cache](#what-breaks-the-cache)). `/effort` also stays as your default for future sessions, so check it when you start.

For very simple sessions you can turn thinking off entirely: `MAX_THINKING_TOKENS=0 claude`. That is below `/effort low`. It works on all models except Fable 5. Skills can set their own `model` and `effort`, and subagents their own `model`, so routine jobs can run cheaply even if your main session is on `high`.

### Prompt caching

Claude has no memory between requests. **Every request resends the whole conversation**: tool definitions, system prompt, CLAUDE.md, every message, every file read and every command output so far.

Prompt caching makes this affordable. If a request starts with the same tokens as the previous one, the server reuses its saved work and only processes the new part at full price.

| Token type | Approximate price |
|---|---|
| Normal input | 1x |
| Cache read | **0.1x** |
| Cache write | 1.25x (5-minute cache) to 2x (1-hour cache), paid once per token |
| Output | about 5x |

Claude Code handles caching for you. You cannot turn it on, but **some of your actions break it.**

### Example: "fix the failing test in `utils.test.ts`"

This takes about five requests to the model:

```
Request 1: [tools + system prompt + CLAUDE.md + your message]             all new: full price, written to cache
Request 2: [ ...everything above... ] + test file                         old part cached (0.1x), test file new
Request 3: [ ...everything above... ] + file under test                   only the new file at full price
Request 4: [ ...everything above... ] + Edit result                       only the edit is new
Request 5: [ ...everything above... ] + `npm test` output                 only the test output is new
           -> tests pass, Claude writes a summary, done
```

Each request is longer than the last. Caching keeps the repeated part cheap. **Appending to the end is the cheapest thing that can happen.**

### What breaks the cache

The request is built in this order:

```
[ tool definitions ] [ system prompt ] [ CLAUDE.md ] [ conversation ... ]
  front ------------------------------------------------------> end
```

A change near the front invalidates everything after it. The next request is then processed again at full price.

| Action | Why it costs |
|---|---|
| `/model` mid-conversation | Each model has its own cache. The next turn re-processes everything at full price. (The `opusplan` setting switches model when entering/leaving plan mode, so it does this too.) |
| `/effort` mid-conversation | Effort is part of the cache key. Same effect. |
| Turning on fast mode mid-conversation | Also part of the cache key, and the re-processing is billed at fast-mode rates. Turn it on at the start. Turning it off is fine. |
| `/compact` | Replaces the conversation with a summary, so the old cache no longer matches. |
| Waiting too long | The cache expires after **1 hour** on a subscription, **5 minutes** with an API key (set `ENABLE_PROMPT_CACHING_1H=1` to get 1 hour). Resuming an old session usually re-processes everything. |

**Rule of thumb:** pick your model and effort level at a cheap moment, at the start of a session or right after `/clear`, not in the middle of a long conversation. Both settings carry over from your last session, so run `/model` and `/effort` once at the start to check them.

### How many tokens a session sends

Anything added to a session is **resent on every later turn until the session ends**. Cached resends are cheap but not free, and the content also takes up room the model has to think within (a cluttered context gives worse answers).

The total cost of a session depends on:

1. **how many tokens enter the context,**
2. **how many turns they stay there,**
3. **how many sessions run at the same time.**

#### 1. What enters the context

**Loaded at startup:** tool definitions, the system prompt, CLAUDE.md, MCP server tools, skill descriptions, plugins. Run `/context` in a fresh session to see the breakdown, then trim:

- Keep CLAUDE.md to specific, useful instructions. Move workflow details into skills.
- Turn off MCP servers you do not need with `/mcp`.
- Disable plugins you do not use with `/plugin`.

**File reads.** Vague requests make Claude search, and every search result stays in the context:

| You type | What happens |
|---|---|
| "the tests are failing" | Claude greps, opens several files, runs the tests... all of it stays |
| "fix the failing test in utils.test.ts" | One Read call |
| "fix the failing test in @utils.test.ts" | The file is attached up front. No Read call at all. |

Mention each file only once per conversation. Mentioning it again attaches another copy.

**Command output.** Output over 30,000 characters is saved to a file, and only a short preview enters the conversation (`BASH_MAX_OUTPUT_LENGTH` changes this limit). Output under that limit stays in full, so a test runner that prints hundreds of lines adds a lot. Put quiet versions of common commands in CLAUDE.md, for example:

```markdown
- Run one test file: `npx vitest run <file> --reporter=dot`
- Run Python tests: `pytest -q`
```

#### 2. How many turns it stays

Each new turn rereads all earlier turns, so **several short sessions are cheaper than one long one.** That is what `/clear` and `/compact` are for ([section 11](#11-clear-compact-and-other-context-commands)).

`/loop` runs a prompt on a timer ([section 9](#9-letting-claude-keep-working-goal-and-loop)). Each run is a full turn that carries the whole conversation, and misses the cache if more than an hour passed. Start loops in a fresh session in a separate terminal.

#### 3. Parallel sessions

Every session you run in parallel has its own context and its own cost. Run several only when the tasks really are independent.

---

## 11. `/clear`, `/compact` and other context commands

### `/clear`: start fresh

```
> /clear
```

Empties the conversation. CLAUDE.md, memory, skills and settings stay. Use it **whenever you switch to an unrelated task.** Nothing from the old task is resent.

If you might want the old conversation later, name it first, then find it with `/resume`:

```
> /rename fix-login-bug
> /clear
```

### `/compact`: summarize and continue

```
> /compact
> /compact keep the list of files we changed and the failing test names
```

Replaces the conversation with a shorter summary, so you keep working on the same task with less context. Use it **when one part of a task is done** but you still need the outcome.

- Tell it what to keep, as in the second example.
- If you always want the same things kept, add a `Compact instructions` section to CLAUDE.md.
- **Compact before you take a break, not after.** Writing the summary is cheap while the conversation is still cached. After an hour (5 minutes on an API key) the cache is gone and the whole conversation must be re-read at full price first.
- Claude Code also **auto-compacts** when the context is nearly full. On a 1M-context model you can make it compact earlier with `/autocompact 200k`.

### `/rewind`: undo the last few turns

```
> /rewind          # or press Esc twice on an empty prompt
```

Goes back to an earlier point in the conversation (and can undo file changes too). Use it when the last few turns went the wrong way. **It is cheaper than `/compact`**: everything before the rewind point is still cached, while `/compact` rewrites the whole conversation.

### Which one to use

| Situation | Command |
|---|---|
| Starting something unrelated | `/clear` |
| Same task, first part finished, context getting long | `/compact <what to keep>` |
| Last few turns were a dead end | `/rewind` |
| About to step away for a while mid-task | `/compact` before you go |

### See where the tokens go

```
> /context    # grid showing what fills the context: system, tools, MCP, memory, messages
> /usage      # token use and cost for this session (/cost is an alias)
```

---

## 12. Subagents

A **subagent** is a helper Claude starts for a sub-task. It has its own context window, system prompt, tools and CLAUDE.md, but **not your conversation**. It works through its own turns, and **only its final answer comes back** to your session. Everything else it read is thrown away.

Use one for jobs that produce lots of output you do not need to keep, for example:

```
> use a subagent to read the last 2000 lines of server.log and tell me the root cause of the 500 errors
```

Your session gets a short summary instead of 2000 log lines that would be resent on every turn.

For small tasks, starting a subagent costs more than it saves.

By default a subagent uses your main model. For a noisy job you repeat often, create a dedicated subagent with `/agents` and give it a cheaper model (`model: haiku` or `model: sonnet`).

---

## 13. Checklist: an efficient session

1. **Start right.** Check `/model` and `/effort` at the start. Turn on fast mode now if you want it.
2. **Check the startup load once.** Run `/context` in a fresh session. Trim CLAUDE.md, MCP servers and plugins you do not use.
3. **Be specific.** Name files, or better, `@`-mention them.
4. **Keep output small.** Use quiet test flags. Send noisy jobs to a subagent.
5. **One task per conversation.** `/clear` between tasks. `/rename` first if you might come back.
6. **Compact at the right time.** When a part is done, or before a break.
7. **Rewind instead of arguing.** If Claude went the wrong way, `/rewind` and rephrase.
8. **Plan big changes.** Use plan mode (`Shift+Tab`) so Claude agrees on the approach before it edits anything.
9. **Ask connectors for less.** "The 5 newest bugs", not "all bugs". Disable connectors you are not using.
10. **Write permission rules** for commands you always approve, so you aren't interrupted while Claude works.
11. **Put limits on unattended work.** Give a `/goal` a turn limit, and run `/loop` in a fresh session with a sensible interval.

---

## 14. Bonus: cutting cost in your own Claude API apps

The same ideas apply when **you** build apps on the Claude API (as in the earlier lessons). Anthropic's article on this names three levers.

### Lever 1: prompt caching

- Cache reads require a **byte-exact** match of the prompt start, on the **same model**, before the cache expires.
- **Put stable content first:** tool definitions and system prompt, then the growing conversation.
- **Keep changing values out of the system prompt.** A timestamp or request ID there breaks the cache on every call.
- **Keep tool definitions stable.** They sit at the very front. Load rarely used tools only when needed (`defer_loading`).
- **Do not change effort or thinking settings mid-conversation.** Switch model or effort only when the cache is already broken anyway (for example during compaction).
- **Use the 1-hour cache** when your agent waits more than 5 minutes for tools or subagents.
- **Pre-warm the cache:** send a request with `max_tokens: 0` at session start (for example while the user is still typing), so the first real request hits a warm cache.
- Check your **cache hit rate** in the Claude Console.

### Lever 2: clean up old prompt habits

Prompts written for older models often contain tricks that waste tokens on newer ones:

- "Double-check your work" / "verify twice": the model does exactly that, e.g. repeats lookups.
- "Be MAXIMALLY thorough", all-caps rules: extra tool calls and long answers.
- Forced step-by-step templates and scratchpads: they clash with the model's built-in thinking.
- Rules that contradict each other: newer models follow instructions more literally, so contradictions hurt more.
- Old settings such as manual thinking budgets: newer models may reject them.

In Anthropic's customer-support test, cleaning these up cut cost by another 9% and raised accuracy by about 2 points.

### Lever 3: pick the right effort

- **Too high:** the model overthinks. More cost and latency, and sometimes worse answers.
- **Too low:** the model stops early, skips checks and answers with partial information.
- **Try a stronger model at a lower effort.** It can beat a weaker model at high effort and cost less.
- **Measure on your own task.** If more effort does not raise your score, the task does not need more thinking.

### Let Claude Code do the audit

The `claude-api` skill has three commands for this:

| Command | What it does |
|---|---|
| `/claude-api prompt-audit` | Finds the anti-patterns above in your prompts, skills, tool descriptions and CLAUDE.md |
| `/claude-api cost-optimize` | Profiles your app's API spend and ranks savings: caching first, then trimming requests, limiting output, and batching |
| `/claude-api hillclimb` | Given an eval, tries model, effort and prompt changes and checks them on held-out test data |

In Anthropic's benchmarks, `cost-optimize` cut cost by roughly 24% to 73% with little or no change in score.

---

## 15. Cheat sheet

### Shell

| Command | What it does |
|---|---|
| `claude` | Start a session |
| `claude -c` / `claude -r` | Continue the last session / pick one to resume |
| `claude -p "..."` | One question, no session (for scripts) |
| `claude -p "..." --output-format json` | Same, as JSON (`result`, `session_id`, `total_cost_usd`) |
| `claude --permission-mode plan` | Start in a permission mode |
| `claude doctor` / `claude update` | Check the install / update |
| `claude plugin install <plugin>@<marketplace>` | Install a plugin |
| `claude --plugin-dir <path>` | Load a local plugin for one session |
| `claude mcp add --transport http <name> <url>` | Add a remote connector |
| `claude mcp add <name> -- <command>` | Add a local connector |
| `claude mcp list` / `claude mcp remove <name>` | List / remove connectors |
| `MAX_THINKING_TOKENS=0 claude` | Session with thinking off |

### In a session

| Command | What it does |
|---|---|
| `/init` | Generate CLAUDE.md for this project |
| `/memory` | Edit CLAUDE.md files, toggle and browse auto memory |
| `/skills` | List skills |
| `/plugin` | Plugin manager |
| `/mcp` | Connectors: status, log in, enable/disable |
| `/permissions` | Allow, ask and deny rules |
| `/tasks` | Background commands |
| `/sandbox` | Turn on the command sandbox |
| `/goal <condition>` | Keep working until the condition is met |
| `/goal`, `/goal clear` | Goal status / stop the goal |
| `/loop 5m <prompt>` | Repeat a prompt every 5 minutes (no interval: Claude picks) |
| `/schedule` | Cloud routine that runs without your computer |
| `/agents` | Manage subagents |
| `/model`, `/effort` | Set model and effort (do it at the start) |
| `/context` | What fills the context window |
| `/usage` | Tokens and cost so far |
| `/clear` | New, empty conversation |
| `/compact [what to keep]` | Summarize the conversation |
| `/rewind` | Go back to an earlier point |
| `/rename`, `/resume` | Name a session, reopen one |
| `/config`, `/hooks` | Settings, hooks |
| `/help` | All commands |

### Keys

| Key | What it does |
|---|---|
| `@file` | Attach a file |
| `!cmd` | Run a shell command |
| `Shift+Tab` | Cycle permission modes (including plan mode) |
| `Ctrl+B` | Move a running command to the background |
| `Esc` / `Esc Esc` | Stop Claude / rewind menu |
| `Ctrl+R` | Search your prompt history |

---

## Sources

- [Maximizing the value of your Claude Code sessions](https://claude.com/resources/articles/maximizing-the-value-of-your-claude-code-sessions)
- [Reducing cost and improving performance with Claude Platform](https://claude.com/resources/articles/reducing-cost-and-improving-performance-with-claude-platform)
- [Claude Code documentation](https://code.claude.com/docs): setup, memory, skills, plugins, commands
