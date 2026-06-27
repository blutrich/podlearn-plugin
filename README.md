# PodLearn for Claude Code

One install gives Claude Code the **PodLearn skill + MCP server** — research,
transcribe, search, and learn from podcasts and YouTube videos, with cited
transcripts and AI-generated lessons.

This replaces the old two-step setup (`curl … | bash` for the skill **and** a
manual `claude mcp add` for the MCP). Installs and updates are now managed by
Claude Code's plugin system.

## Install

**1. Get your token** (free) — generate a personal MCP token at
**https://podlearn.vercel.app/settings**. It looks like `pl_mcp_…`.

**2. Set it as an environment variable** so the bundled MCP can authenticate.
Add this to your `~/.zshrc` (or `~/.bashrc`) so it persists, then restart your
terminal:

```bash
export PODLEARN_MCP_TOKEN=pl_mcp_your_token_here
```

**3. Add the marketplace and install the plugin** inside Claude Code:

```
/plugin marketplace add blutrich/podlearn
/plugin install podlearn@podlearn
```

**4. Restart Claude Code** (or `claude reload`). Try:

> "What podcasts do you have about AI agents?"
> "Transcribe this and make me a lesson: <youtube or podcast URL>"

## What you get

- **Skill** (`podlearn`) — routes your intent through the MCP tools so you don't
  have to think about which tool to call.
- **MCP server** — `search_podcasts`, `search_youtube`, `search_episodes`,
  `list_feed_episodes`, `transcribe_url`, `transcribe_feed` (bulk/season),
  `get_transcription`, `search_transcription`, `generate_lesson`, and more.

## Update / remove

```
/plugin marketplace update podlearn
/plugin uninstall podlearn@podlearn
/plugin marketplace remove podlearn
```

## Notes

- The MCP authenticates with **your** token (`${PODLEARN_MCP_TOKEN}`), resolved
  from your shell environment when Claude Code starts. If a tool returns an auth
  error, make sure the env var is set and relaunch.
- Paid actions (`transcribe_url`, `transcribe_feed`, `generate_lesson`) use your
  PodLearn credits — the skill confirms cost before spending.

## License

AGPL-3.0 — see the [PodLearn repository](https://github.com/blutrich/podlearn).
