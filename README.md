# Mixtape

A Flask + SQLAlchemy social music API — share songs, build playlists, track listening streaks, and see what friends are playing.

Built for **CodePath AI201 Project 5: Mixtape Bug Hunt**. I reproduced, root-caused, and fixed all **five** open issues in the service layer, one conventional commit per fix, with full RCA write-ups in [`submission.md`](./submission.md).

**Repo:** [jaeoh91/AI201-Project5](https://github.com/jaeoh91/AI201-Project5) · **Branch:** `bugfix/mixtape`

---

## What I shipped

All five reported bugs lived in `services/` (routes stay thin). Fixes:

| # | Symptom | Root cause (short) | Fix |
|---|---------|----------------------|-----|
| **5** | Last playlist song missing | `songs[:-1]` dropped the final item | Return full list |
| **3** | Duplicate songs in search | Unused `outerjoin` on `song_tags` multiplied rows | Remove join |
| **4** | No notify on song rating | `rate_song()` never called `create_notification()` | Mirror playlist-add notify path |
| **2** | “Listening Now” shows stale friends | `RECENT_THRESHOLD` was 24h | Narrow to 30 minutes |
| **1** | Streak resets on Sundays | `weekday() != 6` blocked Sunday increments | Remove Sunday guard |

Each fix includes reproduction steps, navigation path, side-effect checks, and pytest verification. Full detail: [`submission.md`](./submission.md).

After these changes: **`.venv/bin/pytest tests/` → 13 passed**.

---

## Skills demonstrated

- **Debugging discipline** — reproduce before editing; symptom → route → service → root cause
- **SQLAlchemy / query literacy** — join multiplication, ORM uniquing vs raw row counts, association tables
- **Time / boundary reasoning** — calendar streaks, recency windows, both sides of off-by-one / cutoff bugs
- **API side effects** — notifications as intentional follow-on behavior after writes
- **Testing & verification** — pytest suites + manual HTTP / shell reproduction + boundary cases
- **Technical writing** — RCA entries suitable for a bug tracker or postmortem
- **Git hygiene** — feature branch `bugfix/mixtape`, one `fix:` commit per bug

---

## Stack

- Python · Flask · Flask-SQLAlchemy · pytest · SQLite

---

## Quick start

```bash
source .venv/bin/activate          # existing uv venv preferred
# uv pip install -r requirements.txt   # if needed

.venv/bin/python seed_data.py      # reset + seed instance/mixtape.db

FLASK_APP=app:create_app .venv/bin/flask run --port 5050
```

Use `http://127.0.0.1:5050` (not `localhost` on macOS). Do **not** run `python app.py` — use the app factory.

```bash
.venv/bin/pytest tests/ -v
```

---

## API surface

| Area | Prefix | Examples |
|------|--------|----------|
| Songs | `/songs` | search, rate, listen |
| Playlists | `/playlists` | create, list songs, add song |
| Users | `/users` | profile, streak, notifications |
| Feed | `/feed` | listening-now, activity |

Seeded users include `nova`, `darius`, `simone`, `kenji`, `aaliya`. IDs are UUIDs — look them up after seeding (e.g. via `sqlite3 instance/mixtape.db`).

---

## Project layout

```text
├── app.py                 # Flask app factory
├── models.py              # User, Song, Playlist, Rating, ListeningEvent, …
├── routes/                # Thin blueprints (no business logic)
├── services/              # Business logic — where the bugs lived
│   ├── streak_service.py
│   ├── feed_service.py
│   ├── search_service.py
│   ├── notification_service.py
│   └── playlist_service.py
├── tests/                 # streaks, search, playlists
├── seed_data.py
├── submission.md          # Codebase map + RCA for all five fixes
└── requirements.txt
```

---

## Example checks (after seed)

```bash
# Playlist should return all songs (Issue #5)
curl "http://127.0.0.1:5050/playlists/<playlist_id>/songs"

# Search should not duplicate multi-tag songs (Issue #3)
curl "http://127.0.0.1:5050/songs/search?q=Anthem"
```
