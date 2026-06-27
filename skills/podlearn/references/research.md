# Cross-episode research workflow

Use when the user wants to know what multiple podcasts/guests think about a topic: "what have podcast guests said about AI agents this year", "compile a literature review of what people say about Y", "what's the consensus across podcasts about Z".

## Flow

This workflow is **search-driven, not enumerate-driven**. There is no way to iterate the whole library — the server has thousands of episodes. Don't try.

1. **Bound the search.** Ask the user (briefly, if it's not already clear):
   - Which podcasts? (If unspecified, pick 3-5 popular ones via `search_podcasts` for relevant queries — e.g., "AI", "tech", "venture")
   - Time range? ("this year", "since 2025", "anything recent")
   - How exhaustive? (Most users want 3-5 cited insights, not a 50-source bibliography. Ask if unclear.)

2. **Find candidate podcasts.** `search_podcasts(query)` for each relevant phrasing of the topic. Dedupe by `podcast.id`. You usually end up with 3-8 candidate shows.

3. **Find candidate episodes within those podcasts.** For each podcast, `list_episodes(podcast_id, status: 'completed', limit: 5)`. The skill is **biased toward `status: 'completed'`** — episodes not yet transcribed don't help research and shouldn't be triggered automatically.

4. **Probe transcripts for the topic.** For each episode candidate, `search_transcription(episode_id, query: <topic>)`. This is the expensive step in tokens — it returns the relevant chunks, not full transcripts. Cap at ~10 episodes total to stay within reasonable context.

5. **Filter by signal.** Most candidates will be irrelevant. Keep episodes where `search_transcription` returned substantive content (multiple matches, dense discussion) and drop ones that only mention the topic in passing.

6. **Synthesize.** Cluster the surviving findings by sub-theme. Write a cited synthesis. Output format from SKILL.md.

## Common edge cases

- **Topic is too narrow** — `search_transcription` returns 0 across all candidates. Tell the user: "I searched [N podcasts] and didn't find substantive discussion. Want me to widen the search to [adjacent topic]?"

- **Topic is too broad** — every transcript matches, but most matches are surface mentions. Tighten the query (e.g., "AI agents" → "autonomous agents in production"). Tell the user you tightened the query and what you found.

- **User asked "across all podcasts"** — politely scope down. Say something like "the library has thousands of episodes; let me search the most relevant 5-10 first and we can expand." Don't try to brute-force.

## Cost shape

Per cross-episode research:
- 1-3 `search_podcasts`: cheap
- 3-8 `list_episodes`: cheap
- 5-10 `search_transcription`: medium each (each returns ~1-5KB of relevant chunks)
- 0 `get_transcription` (full text) — unless the user wants to drill into ONE specific episode after the research. Then it's a single-episode follow-up flow.

This workflow is intentionally lightweight in tokens. The expensive part of the library is the full transcripts; the smart move is to use `search_transcription` to surface relevant slices instead.

## Anti-pattern

❌ "Let me fetch the full transcript of every episode and read them all."

This burns the user's context budget on irrelevant content and takes forever. The whole point of `search_transcription` is to avoid this.
