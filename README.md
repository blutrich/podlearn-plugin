<div align="center">

# 🎧 PodLearn for Claude Code

**Turn any podcast or YouTube video into searchable transcripts and AI lessons — without leaving Claude Code.**

[![Claude Code plugin](https://img.shields.io/badge/Claude_Code-plugin-d97757)](https://code.claude.com/docs/en/plugins)
[![MCP included](https://img.shields.io/badge/MCP-server_bundled-5436DA)](https://podlearn.vercel.app)
[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-3DA639)](./LICENSE)

</div>

One install gives Claude Code the **PodLearn skill *and* MCP server** — discover,
transcribe, search, and learn from podcasts and YouTube, with cited transcripts
and AI-generated lessons. It replaces the old two-step setup (`curl … | bash` for
the skill **plus** a manual `claude mcp add` for the MCP). Installs, updates, and
the token prompt are all managed by Claude Code.

---

## ⚡ Install

> **1. Get your token** (free) at **[podlearn.vercel.app/settings](https://podlearn.vercel.app/settings)** — a `pl_mcp_…` token. Keep it handy.

**2. Add the marketplace and install** — in Claude Code:

```
/plugin marketplace add blutrich/podlearn
/plugin install podlearn@podlearn
```

**3. Paste your token when prompted.** Claude Code asks for your *PodLearn MCP
token* on install — paste the `pl_mcp_…` from step 1. It's stored in your system
keychain (never a settings file, never an env var), and the bundled MCP
authenticates with it automatically.

**4. Restart Claude Code** (or `claude reload`), then try:

> 💬 *"What podcasts do you have about AI agents?"*
> 💬 *"Transcribe this and make me a lesson: ‹youtube or podcast URL›"*
> 💬 *"Find the best quotes about pricing from ‹episode›"*

---

## 📦 What's inside

| | |
|---|---|
| **Skill** (`podlearn`) | Routes your intent through the MCP tools so you never think about tool dispatch. |
| **MCP server** | `search_podcasts`, `search_youtube`, `search_episodes`, `list_feed_episodes`, `transcribe_url`, `transcribe_feed` (bulk/season), `get_transcription`, `search_transcription`, `generate_lesson`, and more. |

## 🔄 Manage

```
/plugin marketplace update podlearn      # pull the latest
/plugin uninstall podlearn@podlearn      # remove the plugin
/plugin marketplace remove podlearn      # remove the marketplace
```

## 🔐 Notes

- The MCP authenticates with **your** token (entered on the install prompt,
  stored in the keychain). Auth error? Re-check the token in `/plugin` config or
  generate a fresh one at [podlearn.vercel.app/settings](https://podlearn.vercel.app/settings).
- Paid actions (`transcribe_url`, `transcribe_feed`, `generate_lesson`) spend
  PodLearn credits — the skill confirms cost before charging.

## 📄 License

[AGPL-3.0](./LICENSE) — © 2026 Ofer Blutrich. Built on the
[PodLearn](https://podlearn.vercel.app) platform.
