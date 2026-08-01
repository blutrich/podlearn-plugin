---
name: podlearn
description: Research, summarize, extract from, and learn from podcast and video transcripts via the PodLearn MCP server. Use this skill whenever the user mentions a podcast, podcast guest, podcast topic, episode, transcript, audio show, YouTube video/talk, or learning from podcasts — including phrasings like "summarize Lex Fridman's latest", "what did Tim Ferriss say about X", "find quotes about AI from podcasts", "research what podcasters think about Y", "make a lesson from this episode", "transcribe this talk", or just pastes a podcast OR YouTube URL (youtu.be / youtube.com). Trigger even when the user doesn't say "podcast" explicitly — if the request involves audio/video content that needs transcribing, searching, summarizing, or learning from, this skill applies. Composes the PodLearn MCP tools (search_podcasts, search_episodes, list_episodes, transcribe_url, get_transcription, search_transcription, generate_lesson, get_lesson_content, etc.) so the user doesn't have to think about routing.
---

# PodLearn — Podcast Transcript Skill

Turn podcasts into citable, searchable knowledge. This skill orchestrates the PodLearn MCP server's tools so the user can ask plain-English questions about podcast content and get cited answers without writing tool-routing logic.

## Setup (one-time)

This skill consumes the **PodLearn MCP server** — **25 tools**, live at `https://podlearn.ai/mcp` (Streamable HTTP, the modern spec 2025-03-26+ transport, negotiating protocol `2025-11-25`). Configure it once:

1. Sign in at <https://podlearn.ai> (free trial = 2 episode credits)
2. Go to **Settings → API Tokens → Generate New Token**. Copy the `pl_mcp_YOUR_TOKEN_HERE`-shaped string (it is shown once).
3. Install the MCP server in Claude Code:

```sh
claude mcp add --transport http podlearn \
  https://podlearn.ai/mcp \
  --header "Authorization: Bearer pl_mcp_YOUR_TOKEN_HERE"
```

Then `claude mcp list` should show **PodLearn Transcripts** as connected.

> `https://podlearn.ai/mcp` is the canonical URL — it's the server's own advertised identity (its 401 `WWW-Authenticate` names this resource) and the identity the connector icon ships under. The underlying Supabase endpoint
> (`https://httiyebjgxxwtgggkpgw.supabase.co/functions/v1/mcp-server-streamable`) still works and remains valid if you're already configured against it. The legacy endpoints `/mcp-server` (vanilla HTTP) and `/mcp-server-sse` (2024-11-05 SSE) still work too and expose the same 25 tools — use them only if your MCP client doesn't support Streamable HTTP yet.

If the MCP isn't connected when this skill runs, stop and tell the user to run the install command above — don't proceed without it.

**Using this from OpenAI Codex?** See `agents/openai.yaml` in this skill's directory (Codex-native metadata, additive — Claude Code ignores it) and `docs/use-podlearn-from-codex.md` in the repo for the full OAuth/bearer walkthrough.

## How to think about a request

There are three rough shapes of user question. Pick the workflow that fits; don't try to do all three:

| If the user asks… | Workflow | Read |
|---|---|---|
| "summarize / what did X say about Y in episode N" | **Single-episode** — find one specific episode, fetch its transcript, synthesize. | `references/single-episode.md` |
| "research what guests think about X" / "across podcasts, who has said…" | **Cross-episode research** — search across transcripts, return cited synthesis. | `references/research.md` |
| "make a lesson from this URL" / "transcribe this and teach me" | **From-URL ingestion** — kick off transcription, poll, then learn. | `references/from-url.md` |

For anything else (extract quotes, find timestamps, generate study guide, browse the library), default to the single-episode workflow — most asks are about one episode.

## Core moves

Whatever workflow you pick, you'll be composing a small set of MCP tools. The names, arg shapes, and purposes — **all 25**, grouped:

**Discovery** (no cost — pure discovery):
- **`search_podcasts(query)`** — first move when the user names a podcast/host. Searches BOTH the PodLearn library AND PodcastIndex.org (4M+ podcasts globally) + resolves Spotify/Apple URLs. Each result has a `source` field: `local` means episodes are already in PodLearn (use `list_episodes` / `search_episodes` next); `podcastindex` / `spotify` means the podcast is discoverable but no episodes are yet ingested — use `transcribe_url(audio_url, title)` to ingest specific episodes. The `counts_by_source` summary tells you at a glance whether you need to ingest or can read existing data.
- **`list_podcasts(limit?, offset?, category?)`** — browse podcasts already in the PodLearn library (most-recently-added first), optionally filtered by category. Use for "what's in the library" browsing; for discovery beyond the library use `search_podcasts`.
- **`get_podcast_details(podcast_id)`** — full metadata for one podcast already in the library (episode count, etc.). The podcast must be `source: local` from a `search_podcasts` result.
- **`search_episodes(query, status?, limit?, offset?)`** — cross-podcast episode search by title/description AND by podcast name/author. Use this when the user names a TOPIC or HOST but you don't have a podcast_id. Returns matching episodes with `transcription_status` so you can see at a glance which are ready to read vs need transcription. `status: 'transcribed'` filters to ready episodes; `status: 'untranscribed'` filters to "available to transcribe" (anything not yet completed). NOTE: this only searches the LOCAL `episodes` table — episodes from PodcastIndex podcasts that haven't been ingested won't appear. For those, see `list_feed_episodes`.
- **`list_feed_episodes(feed_id, max?, since?, before?)`** — fetch the full canonical episode list for a podcast directly from PodcastIndex (the upstream RSS index). NO COST — pure discovery. Use after `search_podcasts` returns a podcast (use the row's `id` as the `feed_id`). Each episode is enriched with `in_library`, `episode_id` (if already ingested), and `transcription_status`. Pagination is time-based: pass `before: <unix-seconds>` to page back. This is how you show the user "all 200 Lex Fridman episodes" even though only one is currently in PodLearn.
- **`search_youtube(query, max?)`** — discover YouTube videos by topic across all of YouTube (not just the PodLearn library / PodcastIndex). NO COST — pure discovery. Use when the user wants a talk/interview/lecture and `search_podcasts` / `search_episodes` find nothing (a lot of long-form content is YouTube-first). Returns video rows each with a ready-to-transcribe `url` → feed to `transcribe_url(url, title)`. NOTE: requires a server-side YouTube Data API key; until that's configured it returns `{error: 'youtube_api_key_unauthorized'}` — fall back to asking the user for a URL.
- **`browse_episodes(query?, limit?)`** — an interactive episode picker: a clickable card grid on hosts that render MCP Apps, a markdown table everywhere else. Use as an alternative to a plain-text list when the user wants to *browse* rather than get a specific answer (e.g. "show me some episodes I could learn from"). Optional `query` filters by title/host.

**Transcription** (paid actions — see guardrails below):
- **`list_episodes(podcast_id, limit?, offset?, status?)`** — episodes of one podcast. **`podcast_id` is REQUIRED** — there's no "list everything" mode. Use `status: 'completed'` to filter for episodes that already have transcripts.
- **`get_episode_details(episode_id)`** — full metadata for one episode. `transcription_status` and `lesson_generation_status` tell you what's been processed.
- **`transcribe_url(audio_url, title, podcast_id?, feed_id?, podcast_name?, language?)`** — ingest and transcribe audio/video NOT yet in the library. Use when the user pastes an episode or YouTube URL OR when `search_episodes` returns 0 matches and the user names a specific source. Pass `feed_id` or `podcast_name` when the parent show is already known (e.g. from `search_podcasts` / `list_feed_episodes`) so the episode lands under its real show instead of "External audio". Raw YouTube watch URLs (youtu.be / youtube.com/watch) work directly — the backend tries the caption track, then falls through to Gemini-native video transcription (no yt-dlp/cookies needed). Auto-creates the episode row, deducts 1 credit atomically (or uses subscription), refunds the credit if transcription fails. **Paid action — confirm cost first (see guardrails).**
- **`transcribe_feed(feed_id, max?, since?, before?)`** — the BULK / whole-season version of `transcribe_url`. Resolves a feed's episodes (pass the `feed_id` from `search_podcasts` / `list_feed_episodes`), skips ones already transcribed (free), and kicks off the rest in one call — up to `max` (hard cap 25). COSTS 1 credit per newly enqueued episode (or subscription quota) — confirm the cost with the user first. ENQUEUE-AND-POLL: returns each enqueued `episode_id` plus counts (`enqueued`, `skipped_completed`, `remaining_untranscribed`, `credits_used`); poll `get_episode_details(episode_id)` until `transcription_status === 'completed'`. Use when the user wants a season / many episodes rather than one at a time. `remaining_untranscribed > 0` means more are available — raise `max` or page back with `before`.
- **`start_transcription(episode_id)`** — kicks off transcription for an episode ALREADY in the library. Returns immediately; transcript becomes available later. Default model is Gemini (`gemini-3.1-flash-lite`), with AssemblyAI as fallback. **Paid action — confirm cost first (see guardrails).**

**Read & search** (free, unlimited for content already in the library):
- **`get_transcription(episode_id, offset?, max_chars?)`** — the transcript text. **This is the foundational primitive** — most synthesis flows pull this first, then reason locally over it. **Paginated at 120,000 chars/call** (Claude.ai/Desktop caps a tool result at ~150k chars). If the response has `has_more: true`, call again with `offset: <next_offset>`.
- **`search_transcription(episode_id, query)`** — keyword search inside ONE transcript. Returns matching chunks with surrounding context. Use this when the user asked about a topic and you want the relevant sections, not the whole hour.
- **`get_transcription_segments(episode_id)`** — timestamped segments (`start_time`, `end_time`, `content` rows). Use when the user wants timestamps (e.g., "jump to where they discuss X").

**Lessons & sharing:**
- **`generate_lesson(episode_id)`** — AI-generated lesson (key takeaways + structured notes) from a completed transcript. **Paid action on first generation — confirm cost first (see guardrails).** Cached lesson returned free on subsequent calls.
- **`list_lessons(limit?, offset?)`** — the caller's saved AI-generated lessons.
- **`get_lesson_content(lesson_id)`** — full content of one saved lesson. Use to retrieve prior synthesis, not to create new ones.
- **`generate_linkedin_post(lesson_id?, episode_id?)`** — turns an EXISTING lesson into a LinkedIn-ready share post. The lesson must already exist; if the user says "make me a LinkedIn post about episode X" and no lesson exists yet, run `generate_lesson` first, then this. Accepts `lesson_id` directly or `episode_id` (uses the most recent lesson for that episode by the caller).

**Account & library:**
- **`get_subscription_status()`** — the caller's subscription plan + active/expired status + period end. Check before kicking off a paid transcription if the user seems credit-constrained.
- **`get_user_credits()`** — the caller's remaining transcription credits. Check before kicking off a paid transcription if the user seems credit-constrained.
- **`get_episode_usage(limit?, offset?)`** — the caller's episode-access usage history for the current billing period.
- **`get_saved_items()`** — the caller's bookmarked episodes and lessons. **Takes no arguments.**
- **`save_item(item_type, item_id)`** — bookmark an episode or lesson for the caller. `item_type` is `episode` or `lesson`.
- **`remove_saved_item(saved_item_id)`** — permanently remove a bookmark. **The only tool the server annotates `destructiveHint: true` — always confirm with the user before calling it.** `saved_item_id` comes from `get_saved_items`.

For full per-tool I/O schemas, ask the MCP via `tools/list`.

## Core principle: cite everything

Every answer that synthesizes podcast content MUST cite. The agent on the user's side cannot verify your claims unless they can trace each statement back to:

- Podcast name + episode title (always)
- Episode ID (for follow-up tool calls)
- Approximate timestamp or transcript section (when known)

Citations look like:

> The Tim Ferriss Show — "How I Built This" episode (id: `6c279…`, ~14:30): "We rebuilt the product six times before…"

Without citations the user has no way to drill into the source, and the skill becomes hallucination-prone. Make citations cheap to follow.

## Behavioral guardrails

**The server's own words on paid actions** (this is the MCP server's `instructions` field, quoted verbatim — trust it over any paraphrase):

> Agent-facing podcast transcript library. Always cite episode_id in synthesis. For paid actions (start_transcription, transcribe_url, generate_lesson) confirm cost with the user before invoking — credit gates enforce server-side but the user should see the cost choice explicitly.

The three paid tools, by name: **`start_transcription`**, **`transcribe_url`**, and **`generate_lesson`**. Each costs 1 credit (or burns subscription quota) — `generate_lesson` only on its first call for an episode; cached lessons are free. `remove_saved_item` is the only tool the server marks `destructiveHint: true`; it isn't a paid action, but confirm with the user before calling it too — it's a permanent delete.

**MANDATORY: check the database BEFORE any paid transcription.** This is a hard
gate — never call `transcribe_url` or `start_transcription` without first
proving the episode isn't already transcribed. Both cost 1 credit; a duplicate
is wasted money AND a duplicate row. The check, in order:
1. `search_episodes(query, status: 'transcribed')` with the title/host keywords
   — searches the LOCAL episodes table across all podcasts.
2. If that's empty, `search_podcasts(query)` → if a `local` podcast matches,
   `list_episodes(podcast_id, status: 'completed')` to scan its episodes.
3. For a pasted URL with a known title (e.g. YouTube — resolve via the oEmbed
   endpoint `https://www.youtube.com/oembed?url=<URL>&format=json`), search by
   that exact title.
4. Only if ALL of the above return no completed match do you transcribe.
If you find a completed match, jump straight to `get_transcription(episode_id)`
— do not transcribe. State to the user: "Already in the library (episode_id …),
reusing it — no credit spent."

**Don't waste the user's credits.** Transcription costs money. Before calling
`transcribe_url` / `start_transcription` / `generate_lesson` (first call):
- Run the MANDATORY DB check above (transcription only).
- Confirm cost with the user: "this will use 1 credit, OK?" and wait — credit
  gates enforce server-side but the user should see the cost choice explicitly.

**Don't fabricate episode IDs.** Episode IDs come from `search_podcasts → list_episodes → get_episode_details` or from `search_transcription` results. Never invent a UUID.

**Don't iterate over the entire library.** There is no `list_all_episodes()`. Cross-episode research goes through `search_transcription` on a focused set, not a brute-force loop.

**Handle "not yet transcribed" gracefully.** If the user asks about an episode whose transcript isn't ready, offer them the path forward (start transcription? wait? pick a different episode?) — don't silently fail.

**Don't tell the user about MCP plumbing unless they ask.** They asked about a podcast; surface insight, not tool names.

## Output format

For research/summarization requests, default to this shape (the user can ask for different):

```markdown
## [Question or summary topic]

### Key points
- [Cited insight 1]
- [Cited insight 2]

### Sources
- [Podcast / Episode title — episode_id]
- [Podcast / Episode title — episode_id]

### Follow-ups (optional)
[1-2 questions the user might want to dig into next, each with the episode_id pre-filled so it's a 1-step ask]
```

For quote extraction, default to numbered list with episode + timestamp on each.

For "make a lesson from URL", point the user at the `generate_lesson` tool result (this is what `lesson_generation_status` tracks in the episode record). Don't try to generate the lesson client-side from raw transcript — the server has dedicated processing for that.

## Protocol note

The server negotiates MCP protocol `2025-11-25`. Don't promise or assume behavior from any later protocol revision (e.g. `2026-07-28`) unless you've verified the server has moved — check `initialize`'s `protocolVersion` if in doubt.

## What this skill is NOT

- Not a podcast player. We don't play audio.
- Not a transcription orchestrator. `start_transcription` is one of several tools, not the centerpiece.
- Not a one-shot Q&A wrapper. The skill chains tool calls and synthesizes — don't reduce it to a single `get_transcription` and dump.

## When in doubt

Ask the user one focused question. Better to clarify "did you mean Lex's episode 421 with Sam Altman, or the recent one with Geoffrey Hinton?" than to fetch 8 transcripts and waste their time.
