# Project 5: Mixtape Bug Hunt — Submission

# Milestone 1: Codebase Map

## Setup

- Virtual environment created using `uv`, dependencies installed from `requirements.txt`
- Database seeded with `python seed_data.py` (5 users, 13 songs, 3 playlists, 10 tags).
- Working branch `bugfix/mixtape` created.
- App started with `FLASK_APP=app:create_app flask run` 

## Main Files and Functionality

### `app.py` — Flask application. 

- Creates the single shared `db = SQLAlchemy()` instance at module level (this is why `python app.py` causes a double-import — the models import `db` from `app`, so the module must only be imported once, via `app:create_app`). 
- `create_app()` configures a SQLite database (`instance/mixtape.db` by default, overridable via `DATABASE_URL`), registers four blueprints with URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()`.


### `models.py` — 6 SQLAlchemy models plus 3 plain association tables:
- `User` — has `listening_streak` (an integer stored directly on the user, not computed) and `last_listened_at`. Friendships are a symmetric many-to-many relationship through the `friendships` table; the seed script inserts both directions of each friendship explicitly.
- `Song` — a song *is* a share: it has `shared_by`, `shared_at`, and `share_note` built in. There is no separate "share" model.
- `ListeningEvent` — one row per listen (`user_id`, `song_id`, `listened_at`). This is the raw data behind both the feed and streaks.
- `Rating` — separate table with a unique constraint on `(user_id, song_id)`, so re-rating updates the existing row rather than creating a new one.
- `Playlist` — songs attach through the `playlist_entries` association table, which carries extra columns: `position` (explicit ordering, not insertion order), `added_by`, and `added_at`.
- `Notification` — generic: `user_id` (recipient), `notification_type` (a string like `song_added_to_playlist`), `body`, `read` flag. Nothing links a notification back to the song/playlist that caused it — the context lives only in the body text.
- `Tag` — many-to-many with `Song` through `song_tags`.

All primary keys are UUID strings, and all timestamps are stored as UTC via `datetime.now(timezone.utc)` defaults.

### `routes/` — four blueprints, one per feature area:

- `songs.py` — `GET /songs/search?q=`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`.
- `playlists.py` — create playlist, get playlist metadata, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`.
- `users.py` — get user, `GET /users/<id>/streak`, get notifications, mark notification read.
- `feed.py` — `GET /feed/<id>/listening-now` and `GET /feed/<id>/activity`.

### `services/` — all business logic:
- `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares calendar dates: same day = no change, exactly one day = increment, otherwise reset to 1. (I think this is where error #1 should lie)
- `feed_service.py` — `get_friends_listening_now()` queries listening events by the user's friends newer than a `RECENT_THRESHOLD` cutoff, then deduplicates so each friend appears once with their most recent song. `get_activity_feed()` is the same idea but limit-based instead of time-based.
- `search_service.py` — `search_songs()` does a case-insensitive `ilike` match on title/artist, with an `outerjoin` to `song_tags`.
- `notification_service.py` — despite the name, this module owns two *actions* (`add_to_playlist()` and `rate_song()`) plus the notification CRUD. `add_to_playlist()` appends the song to the playlist and then creates a notification for the song's original sharer (skipped if the sharer added it themselves).
- `playlist_service.py` — playlist creation and retrieval; `get_playlist_songs()` joins through `playlist_entries` ordered by `position`.

`seed_data.py` — drops and recreates all tables, then seeds 5 users (nova, darius, simone, kenji, aaliya) with friendships, songs with 0/1/3+ tags, listening events both recent (last 30 min) and old (up to 14 days), pre-set streaks, 3 playlists of 7 songs each with explicit positions, and one example `song_added_to_playlist` notification.

### `tests/` — pytest suites for streaks, search, and playlists.

### Data flow — a friend adds my shared song to a playlist

1. Client sends `POST /playlists/<playlist_id>/songs` with `{"song_id": ..., "added_by": ...}`.
2. `routes/playlists.py::add_song()` validates that both fields are present, then calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`. (Interestingly, the playlist route calls into the *notification* service, not the playlist service, because the side effect — notifying the sharer — is considered the main event.)
3. `add_to_playlist()` looks up the song, the adding user, and the playlist (raising `ValueError` for any missing one, which the route converts to a 400).
4. It appends the song to `playlist.songs` if it isn't already there and commits. Note: appending through the relationship means SQLAlchemy inserts into `playlist_entries`, whose `position` column is `nullable=False` — the explicit position handling only happens in the seed script.
5. If the song's original sharer is a different user than the adder, `create_notification()` writes a `Notification` row addressed to `song.shared_by` with type `song_added_to_playlist` and a human-readable body, and commits.
6. The sharer later sees it via `GET /users/<id>/notifications`, which returns notifications newest-first, optionally filtered to unread.



### Data flow — a listen updates a streak

`POST /songs/<id>/listen` → `routes/songs.py::listen()` → `streak_service.record_listening_event()`, which inserts a `ListeningEvent` with the current UTC time and then calls `update_listening_streak(user, now)`. That function compares `now.date()` against `user.last_listened_at.date()` (normalizing naive timestamps to UTC first): 0 days apart is a no-op, 1 day increments the streak, anything else resets it to 1. The streak is a stored counter on `User`, so it's only ever as correct as the last update — `GET /users/<id>/streak` just reads the column.

### Patterns I noticed

- **Lightweight routes, heavyweight services.** Every route does only input parsing and status-code mapping; services raise `ValueError` for anything not found or invalid, and routes translate that to 400/404 JSON. No business logic lives in `routes/`.
- **Serialization via** `to_dict()`**.** Every model defines its own `to_dict()`, and services return plain dicts (or lists of dicts) rather than model objects — except the write paths (`rate_song`, `record_listening_event`, `create_playlist`), which return model instances the routes then call `.to_dict()` on.
- **UTC-aware datetimes throughout**, but SQLite stores them naive, so services that compare times have to re-attach `timezone.utc` when reading back (streak service does this explicitly).
- **Association tables instead of model classes** for friendships, tags, and playlist entries — so extra columns like `position` are only reachable through raw table queries (`playlist_entries.c.position`), not ORM attributes.
- **Service naming is by feature, not by entity** — e.g. `rate_song()` lives in `notification_service.py` because rating is expected to trigger a notification.



### The five open issues and my plan


| #   | Issue                                             | Affected service          |
| --- | ------------------------------------------------- | ------------------------- |
| 1   | My listening streak keeps resetting               | `streak_service.py`       |
| 2   | Friends Listening Now shows people from yesterday | `feed_service.py`         |
| 3   | The same song keeps showing up twice in search    | `search_service.py`       |
| 4   | Notified on playlist-add but not on rating        | `notification_service.py` |
| 5   | The last song in a playlist never shows up        | `playlist_service.py`     |


Having read all five service files, the issues cluster into recognizable patterns: 
- #1 and #2 both live in time/boundary logic (date arithmetic and recency cutoffs), 
- #3 and #5 both live in query/result-set handling (a join that can multiply rows, and a slice on the returned list), 
- and #4 is a missing side effect (compare `rate_song()` against `add_to_playlist()` — one creates a notification, the other doesn't).

**Rough plan:** I'll tackle **#5 (playlist)** first because the symptom is the most deterministic and easiest to reproduce, then **#3 (search duplicates)** since it's the same "inspect the query/result" skillset, then **#4 (missing rating notification)** because the working playlist-add notification gives me a reference implementation to pattern-match against. #1 and #2 are my backups; both need time manipulation to reproduce, which makes them slower to verify.

# Milestone 2: Bug Fixes

## Issue #5 — The last song in a playlist never shows up

**How I reproduced it.** The seed data creates 3 playlists with 7 songs each (explicit positions 1–7). I queried the database directly to understand the expected results, then queried the API on the running server to get the incorrect output:

- `sqlite3 instance/mixtape.db "select pe.position, s.title from playlist_entries pe join song s on s.id = pe.song_id where pe.playlist_id='<id>' order by pe.position;"` → 7 rows, ending with *Free Throws* at position 7.
- `GET /playlists/<id>/songs` → `count: 6`, list ends at *Golden Hour* (position 6). *Free Throws* is missing.

Same result on all three seeded playlists: the DB has 7 entries, the API returns 6, and the missing song is always the highest-position one.

**How I found the root cause.** Route `routes/playlists.py::get_songs()` is a thin wrapper — it just calls `playlist_service.get_playlist_songs()` and serializes. In that function the query itself is correct (join through `playlist_entries`, filter by playlist, `order_by(asc(position))` — the DB check above confirms the query's row set is right). The bug is on the return line: `return [song.to_dict() for song in songs[:-1]]`. The moment of confidence was seeing that `[:-1]` slice — combined with the fact that the missing song is always the *last* one in position order, which is exactly what a `[:-1]` on a position-sorted list drops. 

**Root cause.** `get_playlist_songs()` slices the ordered result list with `songs[:-1]` before serializing. In Python, `list[:-1]` returns every element *except the last one*, so the song with the highest `position` is silently dropped from every response. (The likely intent was `songs[:]` or just `songs`; `[:-1]` may have been a leftover from debugging or a misunderstanding of slice syntax.) The docstring even says "This function returns all songs in the playlist," contradicting the code.

**Fix and side-effect check.** One-line change in `services/playlist_service.py`: `songs[:-1]` → `songs`. Verified afterward:

- Re-ran the reproduction on a fresh server: all three playlists now return `count: 7` with the correct last song (*Free Throws*, *Harlem Renaissance*, *Lagos to London*), still in position order — first song unchanged (both sides of the boundary checked).
- Boundary cases via direct service calls in an app context: a 1-song playlist returns that 1 song (before the fix it would have returned `[]`), and an empty playlist still returns `[]` (no negative-index crash). Temporary test rows were removed afterward.
- `.venv/bin/pytest tests/` → 12 passed, 1 failed. The failure is `test_streak_increments_on_sunday`, which is known open issue #1 (streak service) and is unrelated to this change; all playlist and search tests pass.

**AI usage.** AI agent (Cursor) performed the reproduction, trace, fix, and verification following the workflow in AGENTS.md; I reviewed the diff and the RCA.

**Side note (not one of the five, not fixed).** While boundary-testing with a freshly created playlist, `POST /playlists/<id>/songs` returned a 500: `IntegrityError: NOT NULL constraint failed: playlist_entries.position`. `notification_service.add_to_playlist()` appends via `playlist.songs.append(song)`, which inserts a `playlist_entries` row without a `position`, but the column is `nullable=False` — only the seed script sets positions explicitly. So adding a song through the API can never succeed. Noting it here per the project rules instead of fixing it.

## Issue #3 — The same song keeps showing up twice in search (inconsistently)

**How I reproduced it.** The seed data has songs with 0, 1, and 3+ tags precisely for this bug ("Crown Heights Anthem" has 3 tags). I first confirmed the row multiplication at the SQL level — running the service's exact SQL directly against the database:

```
sqlite3 instance/mixtape.db "select s.title from song s left join song_tags st on st.song_id = s.id
  where s.title like '%Anthem%' or s.artist like '%Anthem%';"
```

returns **three identical `Crown Heights Anthem` rows** — one per tag. Then, in an app context, I executed the service's query in three ways: the legacy `Query.all()` the code uses returned 1 result, but executing the *same statement* row-by-row (`session.execute(q.statement)`) returned 3, and the identical join written 2.0-style (`select(Song)` + `.scalars()`) also returned 3 duplicates.

**How I found the root cause.** Route `routes/songs.py::search()` → `search_service.search_songs()`. The function did an `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` before filtering on title/artist. The moment of confidence was the SQL check above: an outer join to `song_tags` produces one row per (song, tag) pair, so a 3-tag song matches as 3 rows. Crucially, nothing in the query ever *uses* the join — the filter only touches `Song.title`/`Song.artist`, and the tags in the response come from `Song.to_dict()`, which reads the `song.tags` relationship, not the join.

**Root cause.** The `outerjoin` to `song_tags` in `search_songs()` multiplies each matching song by its tag count (an N-tag song becomes N identical rows; the docstring's "along with their associated tags" suggests the join was added under the mistaken belief it was needed to load tags). The "inconsistently" part of the symptom is explained by SQLAlchemy's legacy-`Query` entity uniquing: when full entities are fetched via `Query.all()`, the ORM deduplicates identical rows client-side, masking the bug — but the exact same statement executed any other way (raw rows, 2.0-style `select()`, or after a library migration) returns the duplicates. The query was silently fetching and discarding 3x the rows either way.

**Fix and side-effect check.** Removed the unused `.outerjoin(song_tags, ...)` line in `services/search_service.py` — the filter and the tag serialization are both unaffected by it. Verified afterward:

- Raw SQL without the join returns exactly one row per matching song.
- Service calls for a 3-tag song ("Anthem"), a 0-tag song ("Midnight"), a multi-song artist match ("Elara Moon"), a broad query ("a" → all 13 songs, no duplicates), and a no-match query (returns `[]`) — all correct.
- Over HTTP on a fresh server: `GET /songs/search?q=Anthem` returns `count: 1` with all 3 tags present in the result, and `q=a` returns 13 unique titles. `GET /songs/<id>` (the other function in the same service) still returns the song with its tags.
- `.venv/bin/pytest tests/` → 12 passed, 1 failed; all 5 search tests pass, and the one failure is `test_streak_increments_on_sunday` (known open issue #1, unrelated).

**AI usage.** AI agent (Cursor) performed the reproduction, the three-way query execution comparison that pinned down the uniquing behavior, the fix, and the verification; I reviewed the diff and the RCA.

## Issue #4 — Notified on playlist-add but not on rating

**How I reproduced it.** The seed data already includes a working `song_added_to_playlist` notification for nova (darius added nova's "Midnight Drive" to a playlist). I looked up IDs from the DB, then called `rate_song(darius, midnight_drive_id, 5)` directly in an app context and checked nova's notifications via `get_notifications()`:

- **Before:** 1 notification, type `song_added_to_playlist`.
- **After rating:** still 1 notification — the rating was saved to the `Rating` table, but no new row appeared in `Notification`.

Same pattern as the bug report: playlist-add notifies the sharer; rating does not.

**How I found the root cause.** Both actions live in the same file and are wired through the same routes layer. `routes/playlists.py::add_song()` → `notification_service.add_to_playlist()`, and `routes/songs.py::rate()` → `notification_service.rate_song()`. Reading the two functions side by side: `add_to_playlist()` saves the playlist change, then — if `song.shared_by != added_by_user_id` — calls `create_notification(...)` with type `song_added_to_playlist`. `rate_song()` validates input, creates/updates the `Rating`, commits, and returns — it never calls `create_notification()` at all. The moment of confidence was seeing that the helper and the notification type string `song_rated` already exist in `create_notification()`'s docstring, but nothing in `rate_song()` invokes it.

**Root cause.** `rate_song()` is missing the notification side effect that `add_to_playlist()` already implements. After saving a rating, the function returns without checking whether the rater is someone other than the song's original sharer (`song.shared_by`) and without calling `create_notification()`.

**Fix and side-effect check.** Added the same guard-and-notify block from `add_to_playlist()` to the end of `rate_song()`, after the rating commit:

- If `song.shared_by != user_id`, call `create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.")`.

Verified afterward:

- Friend rates sharer's song → sharer gets a `song_rated` notification with the correct body.
- Self-rating (nova rates her own "Midnight Drive") → notification count unchanged.
- Re-rating the same song → a new notification is created (consistent with `add_to_playlist()`, which also notifies on every call even when the song is already in the playlist).
- Invalid score still raises `ValueError` before any notification is created.
- `.venv/bin/pytest tests/` → 12 passed, 1 failed; the failure is `test_streak_increments_on_sunday` (known open issue #1, unrelated).

**AI usage.** AI agent (Cursor) performed the reproduction, compared the two functions in `notification_service.py`, implemented the fix, and ran the side-effect checks; I reviewed the diff and the RCA.