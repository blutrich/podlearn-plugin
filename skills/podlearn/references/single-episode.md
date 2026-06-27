# Single-episode workflow

Use when the user is interested in ONE specific episode: "summarize Lex Fridman's latest", "what did Tim Ferriss say about AI in episode 421", "extract the best quotes from episode X", "find the timestamp where they discuss Y".

## Flow

1. **Identify the episode.** Three sub-paths:
   - **User named the podcast and episode**: `search_podcasts(query: <podcast name>)` → pick the right `podcast.id` → `list_episodes(podcast_id, status: 'completed', limit: 10)` → pick the episode by title or ordinal (`latest` = the first returned). Confirm with the user if ambiguous.
   - **User gave you an episode_id directly**: skip ahead.
   - **User gave you a podcast but said "latest"**: same as the first sub-path with `limit: 1`.

2. **Get the goods.** Two main shapes depending on the ask:
   - **Whole-episode synthesis** (summarize, study guide, key takeaways): `get_transcription(episode_id)`. Get the full text. Reason locally.
   - **Focused look-up** (find a topic, find a quote, find a timestamp): `search_transcription(episode_id, query: <topic>)`. Returns just the matching chunks. Faster, cheaper on context.

3. **Add timestamps if asked.** If the user wants "where in the episode does X happen" — `get_transcription_segments(episode_id)` returns `(start_time, end_time, content)` rows. Map your synthesis findings to segments.

4. **Cite, then synthesize.** Format per SKILL.md output format.

## Common edge cases

- **`transcription_status !== 'completed'`** in `get_episode_details`. Don't call `get_transcription` — it'll error or return empty. Tell the user: "this episode hasn't been transcribed yet. Want me to start it? (1 credit)"

- **User asks "what was that quote about Y"** — don't grep the full transcript yourself. Use `search_transcription(episode_id, query: 'Y')`. The server does this faster and more accurately.

- **User asks for "best 5 quotes"** — `get_transcription` (full text) then YOU select 5. There's no built-in quote-ranker. Show your selection criteria in the response (e.g., "I chose these for novelty/relevance/density of insight").

- **Episode is in Hebrew** — the transcript shape is identical; just synthesize in the user's language. The Hebrew flag is set per-episode automatically; you don't have to do anything.

## Cost shape

Per single-episode synthesis:
- 1 `search_podcasts` (if needed): cheap
- 1 `list_episodes`: cheap
- 1 `get_transcription`: ~50KB-300KB depending on episode length. This dominates context.
- 0-1 `search_transcription`: tiny
- Total: usually 1-2 tool calls + a longish read.
