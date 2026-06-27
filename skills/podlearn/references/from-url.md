# From-URL ingestion workflow

Use when the user pastes a podcast/YouTube URL (Spotify / Apple / RSS / direct
MP3 / youtu.be) and wants to transcribe / summarize / lesson-ify the episode.

## Flow

1. **Resolve a title for the URL.**
   - YouTube: `curl -s "https://www.youtube.com/oembed?url=<URL>&format=json"`
     gives `title` + `author_name` with no cost and no auth.
   - Spotify/Apple/RSS: ask the user for the episode title + show name if the
     page doesn't expose it.

2. **MANDATORY database check (hard gate — never skip).** Before spending a
   credit, prove it isn't already transcribed:
   - `search_episodes(title_keywords, status: 'transcribed')` — local table,
     all podcasts.
   - If empty, `search_podcasts(keywords)` → if a `local` podcast matches,
     `list_episodes(podcast_id, status: 'completed')`.
   - If a completed match exists → jump to step 5 with that `episode_id` and
     tell the user "Already in the library, reusing it — no credit spent."
   - Only transcribe if every check comes back with no completed match.

3. **Check credits.** `get_user_credits()` (+ `get_subscription_status()` if
   unsure). If zero and not subscribed, say so before starting. Transcription
   is 1 credit (auto-refunded on failure).

4. **Transcribe — just pass the URL.** `transcribe_url(audio_url, title, language?)`
   is the primary ingestion path. It auto-creates the episode row, deducts the
   credit atomically, and refunds on failure. Gemini is the default model.

   **YouTube: pass the raw watch URL directly** (`youtu.be/…`,
   `youtube.com/watch…`). The backend handles it (fixed 2026-05-29):
   - First tries the video's caption track (free, instant, real timestamps).
   - On ANY failure (rate-limit / bot-check / no captions) it falls through to
     **Gemini-native** transcription — Gemini fetches the video over Google's
     own network, so there's no datacenter-IP block, no cookies, no yt-dlp.
   - Verified end-to-end: a raw `youtu.be` URL with no usable captions returns
     a clean verbatim transcript via `transcription_service: gemini_video`.

   Direct audio files (mp3/m4a/RSS enclosure URLs) also work as-is.

   ### Last-resort manual path (only if `transcribe_url` itself fails)
   You almost never need this now. Use it only if the backend returns a hard
   failure for a specific YouTube video (e.g. Gemini can't access it). Requires
   a residential IP — yt-dlp is bot-blocked from datacenter IPs:
   ```sh
   yt-dlp -f 'bestaudio/best' --no-playlist -o '/tmp/ep-raw.%(ext)s' '<URL>'
   ffmpeg -y -i /tmp/ep-raw.* -ac 1 -ar 16000 -b:a 64k /tmp/ep.mp3
   curl -sf -F "file=@/tmp/ep.mp3" https://tmpfiles.org/api/v1/upload
   #   → insert /dl/ into the returned url, then transcribe_url(that /dl/ URL, …)
   ```

5. **Poll status.** `get_episode_details(episode_id)` → `transcription_status`
   goes `processing` → `completed` (≈30-90s for short clips, longer for 2hr+).
   Poll no faster than every ~15-30s. Watch for `failed` with a
   `transcription_error` message.

6. **Verify it's not corrupt.** Gemini occasionally looped a paragraph (fixed
   2026-05-28 with a dedup pass, but old rows exist). Sanity-check
   `length(full_transcript)` is plausible for the duration; if a 30-min video
   returns 250k+ chars, it's the loop bug — flag and re-transcribe.

7. **Fetch the transcript.** `get_transcription(episode_id)` (paginated, 120k
   chars/call — honor `has_more` / `next_offset`).

8. **Lesson generation** (if the user asked for a lesson):
   - Check `lesson_generation_status` in `get_episode_details`.
   - If a lesson exists, `get_lesson_content(lesson_id)` from `list_lessons()`.
   - If `pending`, call the `generate_lesson` MCP tool (server-side pipeline).
     Don't hand-roll the canonical lesson from raw transcript — but a clearly
     labelled "AI summary (not a canonical PodLearn lesson)" is fine if the
     user just wants a quick synthesis.

## Cost shape

- DB check (`search_episodes` / `search_podcasts` / `list_episodes`): free.
- `get_user_credits` / `get_subscription_status`: free.
- `transcribe_url`: 1 credit (refunded on failure).
- Polling `get_episode_details`: free, a handful of calls.
- `get_transcription`: medium (full text).

Transcription is the slowest workflow — set expectations up front.
