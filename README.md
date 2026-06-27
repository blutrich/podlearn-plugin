<div align="center">

# 🎧 PodLearn for Claude Code

### Stop listening. Start learning.

**The podcast & video knowledge engine, right inside Claude Code.**
Drop a link. Get a verbatim transcript, a searchable archive, and an AI lesson in seconds.

[![Claude Code plugin](https://img.shields.io/badge/Claude_Code-plugin-d97757)](https://code.claude.com/docs/en/plugins)
[![MCP included](https://img.shields.io/badge/MCP-server_bundled-5436DA)](https://podlearn.ai)
[![Free to start](https://img.shields.io/badge/free-2_credits_to_try-3DA639)](https://podlearn.ai)
[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-blue)](./LICENSE)

</div>

---

Some of the best thinking on the planet is locked inside hours of audio you'll
never have time for. PodLearn unlocks it. Hand Claude any **podcast or YouTube
URL** and it transcribes the whole thing (yes, even long talks with no captions),
makes every word searchable, and distills the ideas into a structured lesson.
No tab-switching, no copy-paste, no "I'll get to it later."

One install adds the PodLearn **skill *and* MCP server** to Claude Code. The
skill handles the routing; the MCP does the work. You just talk.

## ✨ Why people use it

- ⏱️ **Hours → minutes.** A 2-hour interview becomes a transcript + lesson you skim in five.
- 🔎 **Find the exact moment.** Full-text search across every transcript, with citations. *"Where did they talk about pricing?"* jumps you straight there.
- ▶️ **YouTube that actually works.** Paste any video, captions or not. Conference talks, lectures, channel deep-dives, all transcribed natively.
- 📚 **Bulk a whole season.** `transcribe_feed` ingests an entire podcast feed in one shot, not episode-by-episode.
- 🧠 **Lessons, not just text.** Auto-generated key takeaways and structured notes you can act on.
- 🤖 **Agent-native.** It's a real MCP server, so every call composes with the rest of your Claude workflow (turn a lesson into a LinkedIn post, feed quotes into a doc, whatever).

## 💬 What you can ask

> *"What podcasts do you have about AI agents?"*
> *"Transcribe this and make me a lesson: ‹youtube or podcast URL›"*
> *"Find the best quotes about pricing from ‹episode›"*
> *"Transcribe the last 10 episodes of ‹podcast› and summarize the themes."*
> *"Turn that lesson into a LinkedIn post."*

## ⚡ Install (60 seconds)

> **1. Grab your token** (free, 2 credits to try) at **[podlearn.ai/settings](https://podlearn.ai/settings)** — a `pl_mcp_…` token. Keep it handy.

**2. Add the marketplace** — in Claude Code (run this on its own, wait for confirmation):

```
/plugin marketplace add blutrich/podlearn-plugin
```

**3. Install the plugin** — then run:

```
/plugin install podlearn@podlearn
```

> ⚠️ Run them **one at a time** — they're two separate slash commands. Pasting
> both lines at once makes Claude treat the second as part of the first.

**4. Paste your token when prompted.** Claude Code asks for your *PodLearn MCP
token* on install. It's stored in your system keychain (never a settings file,
never an env var), and the bundled MCP authenticates automatically — **no
separate `claude mcp add`**, the plugin registers the server for you.

**5. Restart Claude Code** (or `claude reload`) and start asking.

That's it. The plugin ships the skill **and** the MCP, so there's no separate
`claude mcp add` step and nothing to wire up by hand.

## 📦 What's inside

| | |
|---|---|
| **Skill** (`podlearn`) | Routes your intent through the MCP tools so you never think about which one to call. |
| **MCP server** | `search_podcasts` · `search_youtube` · `search_episodes` · `list_feed_episodes` · `transcribe_url` · `transcribe_feed` (bulk/season) · `get_transcription` · `search_transcription` · `generate_lesson` · `generate_linkedin_post` · and more. |

## 🔌 Just the MCP server (without the plugin)

The plugin already bundles the server, so most people don't need this. But if you
want the MCP on its own — in **Claude Desktop**, **Cursor**, **ChatGPT**, or any
other MCP client, or in Claude Code without the skill — add it directly.

**Claude Code** (one command, with your token from [podlearn.ai/settings](https://podlearn.ai/settings)):

```bash
claude mcp add --transport http podlearn \
  https://httiyebjgxxwtgggkpgw.supabase.co/functions/v1/mcp-server-streamable \
  --header "Authorization: Bearer pl_mcp_YOUR_TOKEN"
```

**Any MCP client** (Desktop, Cursor, etc.) — point it at the Streamable HTTP endpoint:

| Field | Value |
|---|---|
| Transport | Streamable HTTP (`http`) |
| URL | `https://httiyebjgxxwtgggkpgw.supabase.co/functions/v1/mcp-server-streamable` |
| Header | `Authorization: Bearer pl_mcp_YOUR_TOKEN` |

Generate the `pl_mcp_…` token at [podlearn.ai/settings](https://podlearn.ai/settings).
Full tool reference: [podlearn.ai/mcp-docs](https://podlearn.ai/mcp-docs).

## 🔄 Manage

```
/plugin marketplace update podlearn      # pull the latest
/plugin uninstall podlearn@podlearn      # remove the plugin
/plugin marketplace remove podlearn      # remove the marketplace
```

## 🔐 Notes

- **Your data, your token.** The MCP authenticates with the token you entered on
  the install prompt (stored in your keychain). Auth error? Re-check it in
  `/plugin` config or generate a fresh one at [podlearn.ai/settings](https://podlearn.ai/settings).
- **Transparent pricing.** Reading and searching transcripts already in your
  library is free. Paid actions (`transcribe_url`, `transcribe_feed`,
  `generate_lesson`) spend PodLearn credits, and the skill confirms the cost
  before charging. Start with 2 free credits; top up or subscribe at
  [podlearn.ai](https://podlearn.ai).

## 🌐 Also available

PodLearn runs as a web app at **[podlearn.ai](https://podlearn.ai)** and as an
MCP server for Claude Desktop, ChatGPT, Cursor, and any MCP-compatible client.
This plugin is the fastest way to use it from Claude Code.

## 📄 License

[AGPL-3.0](./LICENSE) — © 2026 Ofer Blutrich. Built on the
[PodLearn](https://podlearn.ai) platform.
