---
name: podlearn
description: Research, summarize, extract from, and learn from podcast and video transcripts via the PodLearn MCP server. Use this skill whenever the user mentions a podcast, podcast guest, podcast topic, episode, transcript, audio show, YouTube video/talk, or learning from podcasts — including phrasings like "summarize Lex Fridman's latest", "what did Tim Ferriss say about X", "find quotes about AI from podcasts", "research what podcasters think about Y", "make a lesson from this episode", "transcribe this talk", or just pastes a podcast OR YouTube URL (youtu.be / youtube.com). Trigger even when the user doesn't say "podcast" explicitly — if the request involves audio/video content that needs transcribing, searching, summarizing, or learning from, this skill applies. Composes the PodLearn MCP tools (search_podcasts, search_episodes, list_episodes, transcribe_url, get_transcription, search_transcription, generate_lesson, get_lesson_content, etc.) so the user doesn't have to think about routing.
---

# PodLearn — Podcast Transcript Skill

Turn podcasts into citable, searchable knowledge. This skill orchestrates the PodLearn MCP server's tools so the user can ask plain-English questions about podcast content and get cited answers without writing tool-routing logic.

## Setup (one-time)

This skill consumes the **PodLearn MCP server**. Configure it once:

1. Sign in at <https://podlearn.ai> (free trial = 2 episode credits)
2. Go to **Settings → API Tokens → Generate New Token**. Copy the `pl_mcp_…` string (it is shown once).
3. Install the MCP server in Claude Code:

```sh
claude mcp add --transport http podlearn \
  https://httiyebjgxxwtgggkpgw.supabase.co/functions/v1/mcp-server-streamable \
  --header "Authorization: Bearer pl_mcp_YOUR_TOKEN_HERE"
```

Then `claude mcp list` should show **PodLearn Transcripts** as connected.

> The endpoint above is the modern Streamable HTTP transport (spec 2025-03-26+). The legacy endpoints `/mcp-server` (vanilla HTTP) and `/mcp-server-sse` (2024-11-05 SSE) still work and expose the same 22 tools — use them only if your MCP client doesn't support Streamable HTTP yet.

If the MCP isn't connected when this skill runs, stop and tell the user to run the install command above — don't proceed without it.

## How to think about a request

There are three rough shapes of user question. Pick the workflow that fits; don't try to do all three:

| If the user asks… | Workflow | Read |
|---|---|---|
| "summarize / what did X say about Y in episode N" | **Single-episode** — find one specific episode, fetch its transcript, synthesize. | `references/single-episode.md` |
| "research what guests think about X" / "across podcasts, who has said…" | **Cross-episode research** — search across transcripts, return cited synthesis. | `references/research.md` |
| "make a lesson from this URL" / "transcribe this and teach me" | **From-URL ingestion** — kick off transcription, poll, then learn. | `references/from-url.md` |

For anything else (extract quotes, find timestamps, generate study guide), default to the single-episode workflow — most asks are about one episode.

## Core moves

Whatever workflow you pick, you'll be composing a small set of MCP tools. The names and purposes:

- **`search_podcasts(query)`** — first move when the user names a podcast/host. Searches BOTH the PodLearn library AND PodcastIndex.org (4M+ podcasts globally) + resolves Spotify/Apple URLs. Each result has a `source` field: `local` means episodes are already in PodLearn (use `list_episodes` / `search_episodes` next); `podcastindex` / `spotify` means the podcast is discoverable but no episodes are yet ingested — use `transcribe_url(audio_url, title)` to ingest specific episodes. The `counts_by_source` summary tells you at a glance whether you need to ingest or can read existing data.
- **`search_episodes(query, status?)`** — cross-podcast episode search by title/description AND by podcast name/author. Use this when the user names a TOPIC or HOST but you don't have a podcast_id. Returns matching episodes with `transcription_status` so you can see at a glance which are ready to read vs need transcription. `status: 'transcribed'` filters to ready episodes; `status: 'untranscribed'` filters to "available to transcribe" (anything not yet completed). NOTE: this only searches the LOCAL `episodes` table — episodes from PodcastIndex podcasts that haven't been ingested won't appear. For those, see `list_feed_episodes`.
- **`list_feed_episodes(feed_id, max?, before?)`** — fetch the full canonical episode list for a podcast directly from PodcastIndex (the upstream RSS index). NO COST — pure discovery. Use after `search_podcasts` returns a podcast (use the row's `id` as the `feed_id`). Each episode is enriched with `in_library`, `episode_id` (if already ingested), and `transcription_status`. Pagination is time-based: pass `before: <unix-seconds>` to page back. This is how you show the user "all 200 Lex Fridman episodes" even though only one is currently in PodLearn.
- **`search_youtube(query, max?)`** — discover YouTube videos by topic across all of YouTube (not just the PodLearn library / PodcastIndex). NO COST — pure discovery. Use when the user wants a talk/interview/lecture and `search_podcasts` / `search_episodes` find nothing (a lot of long-form content is YouTube-first). Returns video rows each with a ready-to-transcribe `url` → feed to `transcribe_url(url, title)`. NOTE: requires a server-side YouTube Data API key; until that's configured it returns `{error: 'youtube_api_key_unauthorized'}` — fall back to asking the user for a URL.
- **`list_episodes(podcast_id, limit?, offset?, status?)`** — episodes of one podcast. **`podcast_id` is REQUIRED** — there's no "list everything" mode. Use `status: 'completed'` to filter for episodes that already have transcripts.
- **`get_episode_details(episode_id)`** — full metadata for one episode. `transcription_status` and `lesson_generation_status` tell you what's been processed.
- **`get_transcription(episode_id)`** — the transcript text. **This is the foundational primitive** — most synthesis flows pull this first, then reason locally over it.
- **`search_transcription(episode_id, query)`** — keyword search inside ONE transcript. Returns matching chunks with surrounding context. Use this when the user asked about a topic and you want the relevant sections, not the whole hour.
- **`get_transcription_segments(episode_id)`** — timestamped segments. Use when the user wants timestamps (e.g., "jump to where they discuss X").
- **`start_transcription(episode_id)`** — kicks off transcription for an episode ALREADY in the library. Returns immediately; transcript becomes available later. Default model is Gemini (`gemini-3.1-flash-lite`), with AssemblyAI as fallback. Costs 1 credit (or burns subscription quota). Do not call this without explicit user authorization.
- **`transcribe_url(audio_url, title, podcast_id?, language?)`** — same idea but for audio/video NOT yet in the library. Use when the user pastes an episode or YouTube URL OR when `search_episodes` returns 0 matches and the user names a specific source. Raw YouTube watch URLs (youtu.be / youtube.com/watch) work directly — the backend tries the caption track, then falls through to Gemini-native video transcription (no yt-dlp/cookies needed). Auto-creates the episode row, deducts 1 credit atomically (or uses subscription), refunds the credit if transcription fails. The MCP path enforces this credit gate; the frontend route currently doesn't (HIGH-severity backlog item). Always confirm cost with the user before invoking — show their current credit balance via `get_user_credits` first if you're unsure.
- **`transcribe_feed(feed_id, max?, since?, before?)`** — the BULK / whole-season version of `transcribe_url`. Resolves a feed's episodes (pass the `feed_id` from `search_podcasts` / `list_feed_episodes`), skips ones already transcribed (free), and kicks off the rest in one call — up to `max` (hard cap 25). COSTS 1 credit per newly enqueued episode (or subscription quota) — confirm the cost with the user first. ENQUEUE-AND-POLL: returns each enqueued `episode_id` plus counts (`enqueued`, `skipped_completed`, `remaining_untranscribed`, `credits_used`); poll `get_episode_details(episode_id)` until `transcription_status === 'completed'`. Use when the user wants a season / many episodes rather than one at a time. `remaining_untranscribed > 0` means more are available — raise `max` or page back with `before`.
- **`get_subscription_status()` / `get_user_credits()`** — check before kicking off a paid transcription if the user seems credit-constrained.
- **`list_lessons()` / `get_lesson_content(lesson_id)`** — the user's saved AI-generated lessons. Use to retrieve prior synthesis, not to create new ones.
- **`generate_linkedin_post({lesson_id})` / `({episode_id})`** — turns an EXISTING lesson into a LinkedIn-ready share post. The lesson must already exist; if the user says "make me a LinkedIn post about episode X" and no lesson exists yet, run `generate_lesson` first, then this. Accepts `lesson_id` directly or `episode_id` (uses the most recent lesson for that episode by the caller).

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
`transcribe_url` / `start_transcription`:
- Run the MANDATORY DB check above.
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

## What this skill is NOT

- Not a podcast player. We don't play audio.
- Not a transcription orchestrator. `start_transcription` is one of several tools, not the centerpiece.
- Not a one-shot Q&A wrapper. The skill chains tool calls and synthesizes — don't reduce it to a single `get_transcription` and dump.

## When in doubt

Ask the user one focused question. Better to clarify "did you mean Lex's episode 421 with Sam Altman, or the recent one with Geoffrey Hinton?" than to fetch 8 transcripts and waste their time.
