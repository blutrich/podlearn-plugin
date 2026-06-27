# PodLearn for Claude Code

One install gives Claude Code the **PodLearn skill + MCP server** — research,
transcribe, search, and learn from podcasts and YouTube videos, with cited
transcripts and AI-generated lessons.

This replaces the old two-step setup (`curl … | bash` for the skill **and** a
manual `claude mcp add` for the MCP). Installs and updates are now managed by
Claude Code's plugin system.

## Install

**1. Get your token** (free) — generate a personal MCP token at
**https://podlearn.vercel.app/settings**. It looks like `pl_mcp_…`. Keep it handy.

**2. Add the marketplace and install the plugin** inside Claude Code:

```
/plugin marketplace add blutrich/podlearn
/plugin install podlearn@podlearn
```

**3. Paste your token when prompted.** On install, Claude Code asks for your
*PodLearn MCP token* — paste the `pl_mcp_…` token from step 1. It's stored
securely in your system keychain (not in any settings file), and the bundled
MCP authenticates with it automatically. No environment variables to set.

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

- The MCP authenticates with **your** token, which you entered when prompted on
  install (stored in your system keychain). If a tool returns an auth error,
  re-check the token in `/plugin` config, or generate a fresh one at
  podlearn.vercel.app/settings.
- Paid actions (`transcribe_url`, `transcribe_feed`, `generate_lesson`) use your
  PodLearn credits — the skill confirms cost before spending.

## License

AGPL-3.0 — see the [PodLearn repository](https://github.com/blutrich/podlearn).
