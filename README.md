# UptimeLens plugins for Claude Code

Official [Claude Code](https://code.claude.com) plugin marketplace for [UptimeLens](https://uptimelens.com) — uptime monitoring, status pages and on-call alerting.

The `uptimelens` plugin connects the UptimeLens MCP server and adds two skills that turn plain requests into the right sequence of its tool calls, with confirmation before anything that changes who gets paged or what is monitored.

| Skill | What it does |
|---|---|
| `uptimelens-on-call` | Find who is on call, acknowledge, snooze, resolve or escalate active pages, cover or swap a shift, and inspect escalation runs and policies. |
| `uptimelens-monitoring` | Create monitors of any type that needs no credentials (HTTP, TCP, DNS, ping, heartbeat for cron jobs and more), recommend what to monitor after a deploy, pause or resume checks, schedule maintenance windows, model monitor dependencies, and open, update or resolve incidents and write their postmortems. |

## Install

In Claude Code:

```
/plugin marketplace add uptimelens/claude-plugins
/plugin install uptimelens@uptimelens
```

Or from your shell:

```bash
claude plugin marketplace add uptimelens/claude-plugins
claude plugin install uptimelens@uptimelens
```

## Sign in

Installing the plugin also adds the UptimeLens MCP server. Run `/mcp` in Claude Code, choose the **UptimeLens** server, select **Authenticate**, sign in to your UptimeLens account and approve access.

Prefer an API token? Create one under **Configuration → API Tokens** in the dashboard, add the server yourself with the token as a header, and disable the plugin's UptimeLens server in `/mcp` so the tools are not listed twice:

```bash
claude mcp add --transport http UptimeLens https://api.uptimelens.com/public/mcp --header "Authorization: Bearer YOUR_API_TOKEN"
```

If you added the server by hand before version 0.3.0, remove that copy (`claude mcp remove UptimeLens`) and use the plugin's.

Read tools work on every plan. Tools that change state — creating monitors, acknowledging pages, overriding a rotation, opening incidents — require a Pro plan or higher, and every tool only sees your own account.

## Try it

- "Who's on call for the database team right now?"
- "Is anything paging right now?"
- "Monitor https://api.example.com."
- "Make sure we hear about it if the nightly backup job stops running."
- "We're deploying tonight from 22:00 to 23:00 — don't page us."

Claude loads the matching skill on its own. You can also invoke one directly, e.g. `/uptimelens:uptimelens-on-call`.

## Updates

Run `/plugin marketplace update uptimelens` to pick up new versions.

## Support

Questions and feedback: [uptimelens.com](https://uptimelens.com).
