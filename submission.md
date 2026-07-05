# Mixtape — Codebase Map

## Bug fixes

### Issue #1: My listening streak keeps resetting

**How I reproduced it** — `tests/test_streaks.py` already contains a test, `test_streak_increments_on_sunday`, that encodes the exact failure condition: call `update_listening_streak(user, saturday)` then `update_listening_streak(user, sunday)` (consecutive calendar days) and assert the streak goes from 1 to 2. Running `pytest tests/test_streaks.py` before touching any code, this test failed — the streak came back as 1 instead of 2 — while the other four streak tests passed. That isolated the bug to the specific case of a consecutive-day listen where the second day is a Sunday.

**How I found the root cause** — Started at `services/streak_service.py`, since it's the file named in the README's issue table for Issue #1. `record_listening_event` just calls `update_listening_streak(user, now)` and commits, so the logic had to be in `update_listening_streak`. Read its docstring first ("If the user listened yesterday: streak increments by 1") and then compared that rule against the actual branching:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
The docstring's rule has no mention of a day-of-week exception, but the code adds one (`today.weekday() != 6`). That mismatch between documented behavior and implemented behavior, combined with the failing Sunday test, was the moment of confidence — the increment path was gated by a condition that shouldn't exist.

**The root cause** — In Python, `date.weekday()` returns `6` for Sunday. The `elif` branch that increments the streak required `days_since_last == 1 and today.weekday() != 6` — meaning it explicitly refused to increment on any day that landed on a Sunday. When `days_since_last == 1` but `today.weekday() == 6` (i.e., the user listened yesterday and today is Sunday), the condition was `False`, so execution fell through to the `else` branch and reset `listening_streak` to `1` instead of incrementing it. Any streak that was supposed to carry over into a Sunday got wiped every week.

**My fix and side-effect check** — Removed the `and today.weekday() != 6` clause so the branch reads `elif days_since_last == 1:` with no day-of-week condition, matching the documented rule exactly (consecutive day → increment, anything else → reset to 1, same day → no change). After the fix, all 5 tests in `tests/test_streaks.py` pass, including `test_streak_increments_on_sunday`. I re-checked the other three branches (`days_since_last == 0` no-op, `user.last_listened_at is None` initial case, and the `else` reset-to-1 case) to confirm none of them referenced `weekday()` or depended on the removed clause, so the fix is isolated to the one faulty condition and doesn't change behavior for same-day repeats, skipped days, or first-time listens.

### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it** — There's no `test_feed.py`, so this one wasn't caught by an existing test. I traced through `services/feed_service.py:get_friends_listening_now` against `seed_data.py`'s listening events, which include an "older events" bucket at `timedelta(hours=2 + i * 8)` for `i in range(8)` — i.e., events 2, 10, 18, 26, 34, 42, 50, and 58 hours old. With `RECENT_THRESHOLD = timedelta(hours=24)`, anything under 24 hours old (2h, 10h, 18h) passes the cutoff filter and shows up in "Friends Listening Now" alongside the genuinely-recent (10–20 minutes old) events. An event 18 hours old is very likely to have happened "yesterday" from the viewer's clock (e.g., it's 2am and the event was at 8am the day before), so a rolling 24-hour window will regularly surface things a user perceives as being from the previous day — matching the reported symptom.

**How I found the root cause** — I initially suspected the comparison itself, `ListeningEvent.listened_at >= cutoff` (feed_service.py:42), since it's exactly the kind of line where a naive-vs-timezone-aware datetime mismatch commonly breaks SQLite comparisons. I checked how `models.py` writes `listened_at` (always `datetime.now(timezone.utc)`, timezone-aware) and how `cutoff` is computed (`datetime.now(timezone.utc) - RECENT_THRESHOLD`, also timezone-aware), and confirmed both go through SQLAlchemy's SQLite `DATETIME` type the same way when bound into the query — so the stored column values and the `cutoff` parameter are coerced into the same naive-UTC string representation, and the `>=` comparison orders them correctly. That ruled out the comparison line itself. With the filter logic and dedup-by-most-recent-event both checking out, the only remaining lever that could produce "yesterday" results from a filter that's supposedly for "now" was the threshold value itself — `RECENT_THRESHOLD = timedelta(hours=24)` at line 13. Cross-referencing `seed_data.py`'s own comment, which calls its 30-minutes-old bucket the one that "should appear in listening now," confirmed the constant, not the query, was mistuned.

**The root cause** — `RECENT_THRESHOLD` was set to `timedelta(hours=24)`, a full rolling day. A trailing 24-hour window doesn't respect calendar-day boundaries, so any event from up to 23 hours 59 minutes ago still counts as "now," even when that timestamp falls on the previous calendar day from the viewer's perspective. The function's own docstring calls this "recently," not "within the last day," so the threshold value contradicted the feature's intent even though the comparison logic around it was correct.

**My fix and side-effect check** — Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`, matching the "recent" window `seed_data.py` was already designed around. Ran the full suite (`pytest tests/`) afterward: all `test_streaks.py` and `test_search.py` cases still passed. Two `test_playlists.py` failures showed up (`test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`, both expecting 5 songs but getting 4) — these are unrelated to this change; they're Issue #5 (`get_playlist_songs` dropping the last song), reproducible independently of any feed_service edit, and untouched by this fix.

## Main files and their responsibilities

**`app.py`** — Application factory (`create_app`). Instantiates the shared `db = SQLAlchemy()` object (imported by every other module to avoid circular imports), configures the SQLite URI, registers the four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and calls `db.create_all()` at startup.

**`models.py`** — All SQLAlchemy models plus three association tables:
- `User` — has `listening_streak` / `last_listened_at` (streak state lives directly on the user row, not in a separate table), and a self-referential many-to-many `friends` relationship through the `friendships` table.
- `Song` — owns `shared_by` (FK to `User`) and `shared_at`; tags come through the `song_tags` association table.
- `ListeningEvent` — one row per (user, song, timestamp) listen. This is the raw log that both the streak service and the feed service read from — there's no separate "streak" table, it's derived from this event stream.
- `Rating` — one row per (user, song), enforced by a `UniqueConstraint`, so rating twice updates the existing row instead of creating a second one.
- `Playlist` — songs relationship goes through `playlist_entries`, which (unlike `song_tags`) is a table with extra columns (`position`, `added_by`, `added_at`), not a plain SQLAlchemy `secondary=` pair-of-FKs table. This is what gives playlists explicit ordering.
- `Notification` — flat table with a `notification_type` string and a pre-rendered `body` string (no templating at read time — the message text is baked in when the notification is created).

**`routes/`** — Thin Flask blueprints. Every route: parses request args/JSON, calls exactly one service function, and translates the result (or a caught `ValueError`) into a JSON response with a status code. No business logic here.
- `songs.py` — search, get-by-id, rate, listen.
- `playlists.py` — create, get metadata, get songs, add song.
- `users.py` — get profile, get streak, list/read notifications.
- `feed.py` — listening-now, activity feed.

**`services/`** — Where all the actual logic and DB queries live.
- `streak_service.py` — `record_listening_event` (writes a `ListeningEvent`, then calls `update_listening_streak`) and `get_streak`.
- `feed_service.py` — `get_friends_listening_now` (last 24h, deduplicated to one entry per friend) and `get_activity_feed` (last N events, no time filter).
- `search_service.py` — `search_songs` (title/artist `ILIKE` match) and `get_song`.
- `notification_service.py` — `create_notification` (generic insert), `add_to_playlist` (mutates `playlist.songs`, then notifies the original sharer), `rate_song` (upserts a `Rating`), plus `get_notifications` / `mark_as_read`. Despite the module name, this file also owns the rating and playlist-add write paths — notification creation is folded into the same functions that perform the underlying action, rather than being a separate step the caller triggers.
- `playlist_service.py` — `create_playlist`, `get_playlist`, `get_playlist_songs` (joins through `playlist_entries`, orders by `position`), `get_user_playlists`.

**`seed_data.py`** — Rebuilds the DB from scratch (`drop_all` + `create_all`) and inserts 5 users with friendships, 25 songs (deliberately split into 0-tag / 1-tag / 3+-tag groups), listening events split into "recent" (last 30 min) and "older" (1–14 days) buckets, 3 playlists, and one pre-existing "song added to playlist" notification. The recent/older split in the listening events and the tag-count split in the songs look intentionally shaped to exercise specific service behavior once exercised through the feed and search endpoints.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`. No `test_feed.py` or `test_notifications.py`, even though `feed_service.py` and `notification_service.py` both exist and are two of the five tracked issues — those two are the ones without a safety net.

## Data flow: adding a song to a playlist (and the sharer notification)

1. Client sends `POST /playlists/<playlist_id>/songs` with `{song_id, added_by}` → `routes/playlists.py:add_song`.
2. The route validates that both fields are present, then calls straight into `notification_service.add_to_playlist(playlist_id, song_id, added_by)` — note this lives in `notification_service`, not `playlist_service`, even though it mutates a playlist.
3. Inside `add_to_playlist` (`services/notification_service.py:35`):
   - Loads `Song`, `User` (adder), and `Playlist` by ID, raising `ValueError` (→ routed to a 400) if any is missing.
   - If the song isn't already in `playlist.songs`, appends it and commits. This append goes through the `playlist_entries` secondary table defined in `models.py`, but nothing in this path sets an explicit `position` — SQLAlchemy just inserts a row via the association proxy.
   - If `song.shared_by != added_by_user_id` (i.e., you didn't add your own shared song), calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)`.
4. `create_notification` (`services/notification_service.py:13`) inserts a `Notification` row and commits.
5. Later, the sharer polls `GET /users/<user_id>/notifications` → `routes/users.py:notifications` → `notification_service.get_notifications`, which queries `Notification` filtered by `user_id` (and `read=False` if `unread_only`), ordered newest-first, and returns the list as dicts via `Notification.to_dict()`.

For contrast, the *rating* flow (`POST /songs/<song_id>/rate` → `routes/songs.py:rate` → `notification_service.rate_song`) follows the same shape — load song/user, upsert a `Rating` row, commit — but never calls `create_notification` at all. The two "friend interacted with my song" actions are handled by sibling functions in the same file, but only one of them reaches the notification step.

## Patterns noticed

- **Routes never touch models or `db` directly except `users.py`'s `GET /<user_id>`** (which calls `db.session.get(User, ...)` inline instead of going through a service). Every other route delegates immediately to a service function; routes only parse input and shape the JSON response.
- **Errors are communicated via `ValueError`**, raised deep in the service layer and caught at the route layer to produce a 400/404 with `{"error": str(e)}`. There's no custom exception hierarchy — every "not found" and every "invalid input" raises the same `ValueError` type, and the HTTP status code (400 vs 404) is chosen per-route rather than encoded in the exception.
- **State derivation vs. stored state is split inconsistently.** `listening_streak`/`last_listened_at` are stored fields on `User` that get mutated in place (`update_listening_streak`), whereas the "listening now" feed and activity feed are recomputed on every request straight from the `ListeningEvent` log. Ratings are stored/upserted state (one row per user+song); notifications are append-only log entries that get flagged `read`.
- **`notification_service.py` is really two things glued together**: generic notification CRUD (`create_notification`, `get_notifications`, `mark_as_read`) plus two unrelated write-actions (`add_to_playlist`, `rate_song`) that happen to *also* want to notify someone. That's why the playlist-add mutation logic lives outside `playlist_service.py`.
- **Association tables are used two different ways**: `song_tags` and `friendships` are bare join tables (`secondary=` with just the two FK columns), while `playlist_entries` carries extra metadata (`position`, `added_by`, `added_at`) and is queried directly with explicit `join()`/`order_by()` in `playlist_service.py` rather than through the ORM `secondary` relationship — `Playlist.songs` (used in `notification_service.add_to_playlist`) and the manual `playlist_entries` join (used in `get_playlist_songs`) are two different paths into the same underlying table.
