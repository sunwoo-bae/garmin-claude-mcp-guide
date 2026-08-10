# Garmin + Claude MCP Guide

A guide for connecting Garmin Connect data to Claude (Code / Desktop) via MCP so it can act as a running coach.
Written from firsthand experience setting it up and hitting several failures and limitations, so it focuses on
pitfalls not covered in the official docs.

## Why you need this

Garmin doesn't provide an official MCP server. The official Garmin Connect API requires developer partner
approval, so individuals can't just use it. Instead, several community MCP servers exist, built on top of
[python-garminconnect](https://github.com/cyberjunky/python-garminconnect) (unofficial, logs in the same way the
Garmin mobile app does). This guide is based on the most comprehensive one,
[Taxuspt/garmin_mcp](https://github.com/Taxuspt/garmin_mcp) (110+ tools, MIT license).

> For reference, **Strava has offered an official MCP connector since 2026-06-01**
> (`https://mcp.strava.com/mcp`, subscribers only, read-only). It lacks Garmin-only metrics like sleep, Body
> Battery, and training readiness, but works on mobile immediately with no separate install. If you don't need
> those metrics, the Strava connector alone may be enough instead of this guide.

## Other options

- [leewnsdud/garmin-connect-mcp](https://github.com/leewnsdud/garmin-connect-mcp) — a smaller, 24-tool server.
  Its advantages are **shoe mileage tracking** and automatic PII filtering in responses, but it hasn't been
  updated in a while (2026-02) and has a small user base (⭐5). If shoe mileage is all you need, it's better to
  register your gear directly in the Garmin Connect app and query it with this guide's server via
  `get_gear`/`get_activity_gear` rather than switching servers.

## Prerequisites

- macOS + [Homebrew](https://brew.sh)
- A Garmin Connect account
- Claude Code and/or the Claude Desktop app

## 1. Install `uv`

```bash
brew install uv
```

## 2. Log in to Garmin (one-time, run directly in your terminal)

**Never let an AI assistant run this step for you.** You have to enter your email/password directly, and if the
assistant runs this command your credentials could end up in the conversation log.

```bash
uvx --python 3.12 --from git+https://github.com/Taxuspt/garmin_mcp garmin-mcp-auth
```

It will ask for your email/password (and an MFA code if you have one). On success, an OAuth token is saved to
`~/.garminconnect`. **You won't need to enter your password anywhere after this.**

- Accounts without MFA can use the `GARMIN_EMAIL`/`GARMIN_PASSWORD` env vars instead, but the interactive method
  above is recommended for security.
- You may see a `mobile+cffi returned 429 ... IP rate limited by Garmin` warning during auth — the server retries
  automatically through another method. As long as you see `SUCCESS` at the end, it's fine.
- The token is valid for **about 6 months**. When it expires, re-authenticate with `garmin-mcp-auth --force-reauth`.

## 3. Register with Claude Code

```bash
claude mcp add garmin -s user -- uvx --python 3.12 --from git+https://github.com/Taxuspt/garmin_mcp garmin-mcp
```

Registering with `-s user` makes it available from any project folder when you open Claude Code on this machine.

**You need to restart your session right after registering/authenticating for the tools to load.** Sessions
already open won't pick it up.

Verify:

```bash
claude mcp get garmin   # Status should be: ✔ Connected
```

## 4. Register with the Claude Desktop app (optional)

Config file: `~/Library/Application Support/Claude/claude_desktop_config.json`

There may be existing content, so **merge in only the `mcpServers` entry — don't overwrite the file**:

```json
{
  "mcpServers": {
    "garmin": {
      "command": "/opt/homebrew/bin/uvx",
      "args": [
        "--python", "3.12",
        "--from", "git+https://github.com/Taxuspt/garmin_mcp",
        "garmin-mcp"
      ]
    }
  }
}
```

> ⚠️ **`command` must be an absolute path** (`/opt/homebrew/bin/uvx`, confirm with `which uvx`). The Desktop app
> runs without a login shell environment, so just writing `uvx` means it can't find it on PATH and the connection
> will fail.

It shares the **same token** (`~/.garminconnect`) as Claude Code, so no re-authentication is needed. After saving
the config, you need to **fully quit (⌘Q) and relaunch** the Desktop app for it to take effect.

## 5. Usage

Ask in natural language and Claude will call the relevant tools on its own.

```
Show me my last 5 running activities
How's my training load and recovery this week?
Analyze the pace and heart rate splits from my last run
What's my predicted 10K time?
```

110+ tools are registered, covering activities, health, training, workouts, gear, and more. It's worth checking
`/mcp` or the tool list early on to see which tools actually got picked up.

## What happens on mobile — this is the biggest pitfall

**The iOS/Android Claude apps can't use local MCP servers.** It's not that the Desktop app doesn't work — mobile
OSes themselves don't allow apps to run arbitrary background processes. The Claude mobile app can only attach
to **remote (HTTP-exposed) MCP servers**. This server is a local stdio process, so it doesn't qualify.

### What actually happens (a pitfall I hit firsthand)

If you ask a Garmin-related question on mobile, Claude doesn't call any tools — it **fabricates an answer from
memory (recall) of a previous conversation.** The problem is that Claude can **confidently claim it "just looked
this up" when that's false.** A real example:

> Mobile Claude: "Yes, that's possible... the running record I just looked up was also retrieved correctly from
> this mobile session."
> → In reality, no tool had ever been called — it reconstructed numbers from an earlier conversation.

This reproduces even in a new chat. **Don't trust Garmin-related answers from mobile without verification, even
if the numbers look accurate.**

### Three alternatives

| Method | Description | Constraints |
|---|---|---|
| **Remote Control** | Open a session on your Mac with `/remote-control` (`/rc`) and drive it remotely from the mobile app. Execution actually happens on the Mac, so local MCP works as-is | The Mac must be on and not asleep |
| **Official Strava MCP** | A remote server hosted directly by Strava. Register it as a `claude.ai` connector and it works on mobile immediately | Requires a Strava subscription, lacks Garmin-only metrics (sleep, Body Battery, etc.), read-only |
| **Self-host remotely** | Run this server in `streamable-http` mode and expose it | Not recommended — see the security section below |

## Security notes

- This server has **no built-in authentication.** Anyone who knows the URL can access it.
- It **includes write tools**: `delete_workout`, `add_weigh_in`, `create_manual_activity`, `upload_workout`, etc.
  — it's not a read-only server.
- So **default to not exposing it to the internet.** If you really need remote hosting:
  - Put it behind a reverse proxy and add an auth layer (e.g. bearer token)
  - Use the `GARMIN_ENABLED_TOOLS` env var to allow only read-only tools
- `~/.garminconnect` and `~/.garminconnect_base64` are your account access credentials. Never commit or share them.

## If you want to merge in data from an old device (a different wearable)

If you've ever switched devices (e.g., a different brand watch → Garmin), the old data can't be pulled in through
this MCP server. To make use of that period's data:

1. Export the data from the old app (usually CSV)
2. Have Claude Code parse it directly and generate a summary markdown file — **don't let the Desktop/mobile
   Claude parse the raw CSV directly.** It's often nested JSON, and an LLM can misread it while still confidently
   stating wrong values.
3. Upload only the cleaned-up summary file (e.g., to Google Drive), so cloud/mobile Claude reads this file
   instead of the original.

This lets mobile Claude answer accurately from **actual file content** instead of memory recall.

> A note when registering a file via the Drive API: many connectors have no way to overwrite an existing file in
> place, and no delete function either. Every content change creates a new file, so either manually delete old
> versions or just avoid updating too frequently in the first place.

## Maintenance

| Item | Method |
|---|---|
| Token expiry (~6 months) | Re-run `garmin-mcp-auth --force-reauth` |
| Saving context | Allowlist only the tools you need with `GARMIN_ENABLED_TOOLS` (misspelled names are silently ignored, so check the actual list after registering) |
| Removing the server | `claude mcp remove garmin -s user` |

## References

- [Taxuspt/garmin_mcp](https://github.com/Taxuspt/garmin_mcp) — the MCP server this guide uses
- [cyberjunky/python-garminconnect](https://github.com/cyberjunky/python-garminconnect) — the underlying library
- [Strava MCP Connector docs](https://support.strava.com/en-us/articles/15401531-strava-mcp-connector)
- [Claude Code Remote Control docs](https://code.claude.com/docs/en/remote-control)

## License

This guide itself is free to use and modify. Each tool (garmin_mcp, python-garminconnect) is governed by its own
repository's license.
