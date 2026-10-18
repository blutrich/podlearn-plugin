---
name: podlearn
description: Research, summarize, extract from, and learn from podcast and video transcripts via the PodLearn MCP server. Use this skill whenever the user mentions a podcast, podcast guest, podcast topic, episode, transcript, audio show, YouTube video/talk, or learning from podcasts — including phrasings like "summarize Lex Fridman's latest", "what did Tim Ferriss say about X", "find quotes about AI from podcasts", "research what podcasters think about Y", "make a lesson from this episode", "transcribe this talk", or just pastes a podcast OR YouTube URL (youtu.be / youtube.com). Trigger even when the user doesn't say "podcast" explicitly — if the request involves audio/video content that needs transcribing, searching, summarizing, or learning from, this skill applies. Composes the PodLearn MCP tools (search_podcasts, search_episodes, list_episodes, transcribe_url, get_transcription, search_transcription, generate_lesson, get_lesson_content, etc.) so the user doesn't have to think about routing.
---

# PodLearn — Podcast Transcript Skill

Turn podcasts into citable, searchable knowledge. This skill orchestrates the PodLearn MCP server's tools so the user can ask plain-English questions about podcast content and get cited answers without writing tool-routing logic.

## Setup (one-time)

This skill consumes the **PodLearn MCP server**: **25 tools** at `https://podlearn.ai/mcp` (Streamable HTTP, protocol `2025-11-25`). Sign in with OAuth; no key to copy.

| Host | Connect |
|---|---|
| Claude Code (local) | `claude mcp add --transport http podlearn https://podlearn.ai/mcp`, then `/mcp` → podlearn → **Authenticate** (or `claude mcp login podlearn`) |
| Claude.ai, Claude Desktop, Claude mobile | Settings → Connectors → Add custom connector → `https://podlearn.ai/mcp` → Connect |
| Claude Code on the web / phone | Use the claude.ai connector above. The cloud sandbox cannot reach podlearn.ai, so a server added inside the session never connects. |
| Codex CLI | `codex mcp add podlearn --url https://podlearn.ai/mcp`, then `codex mcp login podlearn --scopes mcp:read,mcp:write` |
| stdio-only clients | The pinned **mcp-remote 0.14.3** bridge config on https://podlearn.ai/mcp-docs |

During sign-in the client listens on a local callback port. Keep the client waiting until it reports success; if the browser lands on `localhost:<port>/…callback` with "refused to connect", the client stopped waiting. Start the sign-in again.

**Permissions.** **mcp:read** covers the 18 read tools. **mcp:write** is required by seven: **start_transcription**, **transcribe_url**, **transcribe_feed**, **generate_lesson**, **generate_linkedin_post**, **save_item**, **remove_saved_item**. A connection registered read-only cannot be upgraded by re-authorizing; add PodLearn again as a new connection that requests both scopes. Never silently replace a denied OAuth grant with a full-access token.

**Personal tokens** (headless clients only): create one in **Settings → API Tokens** and send it only as **Authorization: Bearer**. Never in a URL, chat, committed config, logs, or process arguments. The server ignores tokens placed in the URL query string (see Connection problems). Codex can read one from the environment with `--bearer-token-env-var PODLEARN_MCP_TOKEN`.

> OAuth tokens are bound to the exact URL used during authorization. Prefer the canonical `https://podlearn.ai/mcp`; a direct Supabase endpoint is a separate resource needing its own sign-in. Legacy HTTP/SSE transports expose 24 tools (no episode picker).

### Connection problems

Diagnose before telling the user PodLearn is down:

| Symptom | Cause | What to do |
|---|---|---|
| Server shows connected but every tool call hangs (no error after a minute or more) | An old config puts the token in the URL query string. Discovery works anonymously, then each call gets 401 and the client waits for a browser sign-in that never completes. | Stop the call. Tell the user to remove that server and add `https://podlearn.ai/mcp` again with OAuth. |
| Two PodLearn connections are available (e.g. a local `podlearn` server and a claude.ai connector) and one hangs or 401s | Each connection has its own credentials | Use the one that answers; name the broken one to the user |
| 401 / "Authentication required" | Sign-in expired or token revoked | Sign in again at the same URL (`/mcp` → Authenticate, or `codex mcp login podlearn`) |
| 403 insufficient_scope on a write tool | Read-only connection | New connection requesting both scopes; still confirm cost before spending |
| Paid call returns "failed (credit refunded)" | Server-side transcription failure; the server refunded | Report the exact error once; do not retry in a loop or switch to another paid tool to compensate |
| Account or entitlement denial | Out of credits or plan limit | `get_user_credits` / `get_subscription_status`, then tell the user |
| A discovery dependency fails (e.g. YouTube search key) | Upstream service, not the connection | Use another discovery tool or ask for a source URL; never start paid processing to work around it |

If no PodLearn connection is available at all, say so and point to the setup table. Do not invent transcripts or tool results.

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

**Read & search** (no transcription credit for existing content; account access still applies):
- **`get_transcription(episode_id, offset?, max_chars?)`** — the transcript text. **This is the foundational primitive** — most synthesis flows pull this first, then reason locally over it. **Paginated at 120,000 chars/call** (Claude.ai/Desktop caps a tool result at ~150k chars). If the response has `has_more: true`, call again with `offset: <next_offset>`.
- **`search_transcription(episode_id, query)`** — keyword search inside ONE transcript. Returns matching chunks with surrounding context. Use this when the user asked about a topic and you want the relevant sections, not the whole hour.
- **`get_transcription_segments(episode_id, start_time?, end_time?)`** — timestamped segments (`start_time`, `end_time`, `content` rows). Use when the user wants timestamps (e.g., "jump to where they discuss X").

**Lessons & sharing:**
- **`generate_lesson(episode_id)`** — AI-generated lesson (key takeaways + structured notes) from a completed transcript. **Paid action on first generation — confirm cost first (see guardrails).** Cached lesson returned free on subsequent calls.
- **`list_lessons(limit?, offset?, episode_id?)`** — the caller's saved AI-generated lessons.
- **`get_lesson_content(lesson_id)`** — full content of one saved lesson. Use to retrieve prior synthesis, not to create new ones.
- **`generate_linkedin_post(lesson_id?, episode_id?)`** — turns an EXISTING lesson into a LinkedIn-ready share post. The lesson must already exist; if the user says "make me a LinkedIn post about episode X" and no lesson exists yet, run `generate_lesson` first, then this. Accepts `lesson_id` directly or `episode_id` (uses the most recent lesson for that episode by the caller).

**Account & library:**
- **`get_subscription_status()`** — the caller's subscription plan + active/expired status + period end. Check before kicking off a paid transcription if the user seems credit-constrained.
- **`get_user_credits()`** — the caller's remaining transcription credits. Check before kicking off a paid transcription if the user seems credit-constrained.
- **`get_episode_usage(limit?, offset?)`** — the caller's episode-access usage history for the current billing period.
- **`get_saved_items(limit?, offset?)`** — the caller's bookmarked episodes and lessons, with pagination.
- **`save_item(item_type, item_id)`** — bookmark an episode or lesson for the caller. `item_type` is `episode` or `lesson`.
- **`remove_saved_item(saved_item_id)`** — permanently remove a bookmark. **The only tool annotated `destructiveHint: true` — obtain authorization unless the user already requested this exact removal.** `saved_item_id` comes from `get_saved_items`; this does not delete the transcript.

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

**The current server initialize instruction**, quoted verbatim. The complete rule below also covers bulk transcription as its tool description requires:

> Agent-facing podcast transcript library. Always cite episode_id in synthesis. For paid actions (start_transcription, transcribe_url, generate_lesson) confirm cost with the user before invoking — credit gates enforce server-side but the user should see the cost choice explicitly.

That verbatim initialize instruction omits **transcribe_feed**. Apply the complete four-tool spending guard: **start_transcription**, **transcribe_url**, **transcribe_feed**, and **generate_lesson**. Single transcription costs one credit or subscription quota unit. The lesson tool advertises one credit/quota unit on first generation, but its current handler checks entitlement without an explicit debit; that advertised amount is unverified and needs an actual usage check. Do not report a lesson credit as spent without evidence. Cached lessons avoid another generation. Bulk transcription skips completed episodes and charges one credit per newly enqueued episode, with a hard cap of 25 per call. Set an explicit maximum and explain the total bound before enqueueing. Transcription credits are refunded on failure by the server; do not promise unobserved quota refunds or refunds for every paid operation.

Write consent and spending authorization are separate. Honor an existing explicit authorization for the same episodes and bounded cost; do not ask again merely because another tool call is needed. Otherwise explain the cost and wait for authorization. **remove_saved_item** permanently deletes a bookmark (not the transcript); use the returned saved-item ID and obtain deletion authorization unless the user already requested that exact removal.

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
`transcribe_url` / `start_transcription` / `transcribe_feed` / `generate_lesson` (first generation):
- Run the MANDATORY DB check above (transcription only).
- Reuse existing explicit spending authorization if its scope and maximum cover
  the operation. Otherwise state the single or bulk cost bound and wait for authorization.

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
