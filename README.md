# UptimeLens plugins for Claude Code

Official [Claude Code](https://code.claude.com) plugin marketplace for [UptimeLens](https://uptimelens.com) — uptime monitoring, status pages and on-call alerting.

The `uptimelens` plugin adds two skills that turn plain requests into the right sequence of UptimeLens MCP tool calls, with confirmation before anything that changes who gets paged or what is monitored.

| Skill | What it does |
|---|---|
| `uptimelens-on-call` | Find who is on call, acknowledge, snooze, resolve or escalate active pages, cover or swap a shift, and inspect escalation runs and policies. |
| `uptimelens-monitoring` | Create HTTP/HTTPS monitors, recommend what to monitor after a deploy, pause or resume checks, schedule maintenance windows, model monitor dependencies, and open, update or resolve incidents and write their postmortems. |

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

## Connect the UptimeLens MCP server

The skills orchestrate the tools of the UptimeLens MCP server, so connect it once:

```bash
claude mcp add --transport http UptimeLens https://api.uptimelens.com/public/mcp
```

Then run `/mcp` in Claude Code, choose **UptimeLens → Authenticate**, sign in to your UptimeLens account and approve access. Naming the server `UptimeLens` matches how the skills refer to its tools.

Prefer an API token? Create one under **Configuration → API Tokens** in the dashboard and pass it as a header instead:

```bash
claude mcp add --transport http UptimeLens https://api.uptimelens.com/public/mcp --header "Authorization: Bearer YOUR_API_TOKEN"
```

Read tools work on every plan. Tools that change state — creating monitors, acknowledging pages, overriding a rotation, opening incidents — require a Pro plan or higher, and every tool only sees your own account.

## Try it

- "Who's on call for the database team right now?"
- "Is anything paging right now?"
- "Monitor https://api.example.com."
- "We're deploying tonight from 22:00 to 23:00 — don't page us."

Claude loads the matching skill on its own. You can also invoke one directly, e.g. `/uptimelens:uptimelens-on-call`.

## Updates

Run `/plugin marketplace update uptimelens` to pick up new versions.

## Support

Questions and feedback: [uptimelens.com](https://uptimelens.com).
