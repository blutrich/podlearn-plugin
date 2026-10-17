# From-URL ingestion workflow

Use when a user provides a podcast, direct audio, or YouTube URL and wants
a summary, transcript, or lesson. The core workflow uses MCP tools on CLI,
desktop, and web; it does not require local media downloads or external uploads.

1. Resolve the show and title with the source page or available discovery tools.
   If neither is available, ask for the title. Do not make shell access a prerequisite.
2. Check for a completed transcript before spending: `search_episodes` with title
   keywords and `status: 'transcribed'`, then `search_podcasts` and
   `list_episodes(podcast_id, status: 'completed')` when a local show matches.
   Use the returned IDs. Reuse a match with `get_transcription` and say that
   no new transcription credit was spent.
3. Check `get_user_credits` and `get_subscription_status`. A new transcription
   can spend one credit or subscription quota. Reuse existing explicit bounded
   spending authorization; otherwise explain the cost and obtain it first.
   OAuth also needs mcp:write; cost authorization does not grant that scope.
4. If an episode already exists but is incomplete, use `start_transcription`.
   Otherwise call `transcribe_url(audio_url, title, podcast_id?, feed_id?, podcast_name?, language?)`.
   Pass the original accessible audio/video URL and known show context. Do not
   upload private media to an unrelated service or claim every URL is supported.
5. Poll `get_episode_details` for completed or failed status, with a reasonable
   delay (roughly 15–30 seconds). Report failure or a still-running job honestly;
   do not start another charged job automatically. A 401 needs reconnection;
   insufficient-scope 403 needs explicit write consent. Entitlement/provider
   errors require their own recovery, not a full-access token bypass.
6. Fetch `get_transcription`, following has_more and next_offset. Cite the
   show, title, episode ID, and timestamps or sections available in the result.
   Suspected corruption should be reported before authorizing any reprocessing.
7. For a requested lesson, check `list_lessons(episode_id: <returned ID>)` and
   retrieve existing content with `get_lesson_content`. If none exists, first
   lesson generation is advertised as another credit/quota unit, but the current
   handler checks entitlement without an explicit debit. Treat the amount as
   unverified, retain spending authorization, and report actual usage before
   claiming a charge. Check authorization before `generate_lesson`.

For a season or feed, use free `list_feed_episodes` discovery first. Set a
bounded max (at most 25) before `transcribe_feed`, explain one credit per newly
enqueued episode or quota use, and obtain authorization for that maximum.
Completed episodes are skipped. Poll the returned episode IDs and inspect actual
enqueued/credit counts rather than claiming the maximum was spent.

Reading existing transcripts, discovery, and polling spend no transcription
credit, though account access and plan restrictions still apply. The server
refunds transcription credits on failure; do not infer quota or lesson refunds
from that rule. Cached lessons return without another credit charge.
