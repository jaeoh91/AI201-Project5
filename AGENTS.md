# AGENTS.md — Mixtape Bug Hunt (AI201 Project 5)

Context and rules for AI agents working in this repo. Read this fully before touching anything.

## What this project is

Mixtape is a small Flask + SQLAlchemy social music app (share songs, collaborative playlists, listening streaks, activity feeds). It is the starter repo for a **bug hunt assignment**: five known bugs live in the `services/` layer. The goal is to reproduce, root-cause, fix, and document at least three of them, one commit per fix, with a root cause analysis (RCA) entry per bug in `submission.md`.

This is coursework about **debugging discipline**, not speed. The process is graded as much as the fix. Never skip the reproduce step, never batch multiple fixes into one commit, and never fix a bug before its reproduction is documented.

## Environment — use uv and the existing venv

- A virtual environment **already exists** at `.venv/`, created with **uv**. Do NOT create a new venv, do NOT delete or recreate `.venv/`, and do NOT touch the system Python.
- All dependencies from `requirements.txt` are already installed. If you ever need to add or reinstall a package, use `uv pip install <pkg>` (with the venv active) — not plain `pip`, not `pip3`, not system Python.
- Run Python things through the venv explicitly: `.venv/bin/python`, `.venv/bin/flask`, `.venv/bin/pytest` (or activate with `source .venv/bin/activate`).

## Running the app

```bash
FLASK_APP=app:create_app .venv/bin/flask run --port 5050
```

- **Never start the app with `python app.py`** — models import `db` from `app`, so running `app.py` as a script double-imports the module and crashes SQLAlchemy. Always use the app factory via `FLASK_APP=app:create_app`.
- The developer often already has the server running on **port 5050** in a terminal. Check for an existing running server before starting your own; if you start one, use a different port and shut it down when done.
- Use `http://127.0.0.1:<port>` (not `localhost`) — on macOS, `localhost` can resolve to IPv6 and hang.
- There is no root route; `GET /` returns 404 by design. Endpoints live under `/songs`, `/playlists`, `/users`, `/feed`.

## Database and seed data

- SQLite DB at `instance/mixtape.db`. Reset it any time with:

```bash
.venv/bin/python seed_data.py
```

  This drops and recreates all tables. Seed IDs are UUIDs regenerated on every reseed — never hardcode IDs; look them up (e.g. `sqlite3 instance/mixtape.db "select id from user where username='nova';"`).
- Seeded state: 5 users (nova, darius, simone, kenji, aaliya) with friendships, 13 songs (with 0, 1, and 3+ tags), 3 playlists of 7 songs each with explicit positions, listening events from the last 30 minutes and from 1–14 days ago, pre-set streaks, and one example `song_added_to_playlist` notification.
- Read `seed_data.py` before reproducing a bug — it was written so that the state needed to trigger each bug already exists (its comments even hint at which data exercises which issue).
- `flask shell` (with `FLASK_APP=app:create_app`) or a direct `.venv/bin/python` REPL with `app.create_app().app_context()` are the fastest ways to call service functions with controlled inputs.

## Architecture in one minute

- `app.py` — app factory; module-level `db = SQLAlchemy()` shared by everything.
- `models.py` — `User`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, `Tag`, plus plain association tables `friendships`, `song_tags`, and `playlist_entries` (which carries `position`, `added_by`, `added_at`).
- `routes/` — thin blueprints (`songs`, `playlists`, `users`, `feed`). They only parse input and map service `ValueError`s to 400/404. **No business logic here.**
- `services/` — all logic, and where all five bugs live: `streak_service`, `feed_service`, `search_service`, `notification_service` (also owns the `rate_song` and `add_to_playlist` actions), `playlist_service`.
- `tests/` — pytest suites for streaks, search, and playlists. Run with `.venv/bin/pytest tests/`.
- All timestamps are UTC (`datetime.now(timezone.utc)`), but SQLite stores them naive — code that compares times re-attaches `timezone.utc` when reading back.

A fuller codebase map (data flow traces, patterns) is in `submission.md` under Milestone 1.

## The five open issues

| # | Symptom | Service |
|---|---------|---------|
| 1 | Listening streak keeps resetting (reported on Sundays) | `services/streak_service.py` |
| 2 | "Friends Listening Now" shows people from yesterday | `services/feed_service.py` |
| 3 | Same song appears multiple times in search (inconsistently) | `services/search_service.py` |
| 4 | Notification sent on playlist-add but not on rating | `services/notification_service.py` |
| 5 | Last song in a playlist never shows up | `services/playlist_service.py` |

Planned order (from Milestone 1): **#5, then #3, then #4**, with #1/#2 as backups.

## Required workflow per bug (Milestones 2 and 3)

Do these steps **in order** for each bug. Do not start the next bug until the current bug's RCA entry is written.

1. **Reproduce first — before changing any code.** Trigger the reported behavior deliberately (HTTP request, `flask shell` call, or direct function call) and capture the actual wrong output. For condition-dependent bugs (#1 needs a specific weekday; #3 depends on which songs match), reason about what app state hits the code path and construct it if the seed data doesn't. If a bug genuinely won't reproduce, switch to another from the list rather than guessing.
2. **Trace symptom → root cause, top-down.** Start at the route, follow each call in order, and record the navigation path (which files, what led where). Verify the hypothesis by running the suspicious function with controlled inputs (temporary `print` debugging is fine — remove before committing). Name the *specific* condition/comparison/missing step, not just the suspicious area.
3. **Implement the smallest fix** that addresses the root cause. No refactoring of unrelated code, no drive-by cleanups, no formatting churn. If the fix touches multiple places, justify each.
4. **Side-effect check.** Re-run the reproduction to confirm the bug is gone, run `.venv/bin/pytest tests/`, and manually exercise related functionality that shares the data or feature. For boundary bugs (#1, #2, #5), verify behavior on **both sides of the boundary** (e.g. first and last playlist song; an event just inside and just outside the recency window; consecutive-day vs. skipped-day listens).
5. **Write the RCA entry in `submission.md`** (append under a Milestone 2/3 section) with all five fields:
   - Issue number and title
   - How you reproduced it (inputs, sequence, data condition, observed wrong output)
   - How you found the root cause (files read, navigation path, the moment of confidence)
   - The root cause — precise, plain English. Name the exact function/comparison and what it returns vs. what the code assumed. "The streak logic had a bug" is not acceptable; "`weekday()` returns 6 for Sunday but the code checked `== 0`, which is Monday" is the standard.
   - Your fix and side-effect check — what changed, why it fixes the cause, what you verified afterward.
   - Also note in the RCA (and in the doc's AI-usage section) how AI was used for that bug.
6. **Commit** — one commit per bug, conventional format, on the `bugfix/mixtape` branch. **Never commit on your own** — when the fix and RCA are ready, ask the developer for confirmation and only commit after they approve:

```bash
git add <changed files> submission.md
git commit -m "fix: <what was wrong and what changed>"
```

## Hard rules

- Stay on the `bugfix/mixtape` branch. Do not push, merge, rebase, or amend unless explicitly asked.
- Do not create commits autonomously. Always show what will be committed (files + message) and wait for the developer's explicit confirmation before running `git commit`.
- One bug = one commit. Include the bug's `submission.md` RCA entry with its fix commit (or a directly following `docs:` commit).
- Don't modify `seed_data.py`, `models.py`, `routes/`, or tests to "fix" a bug unless the root cause genuinely lives there — the bugs are in `services/`. Changing a test to pass is never a fix.
- Don't fix bugs that weren't asked about; if you notice one of the other five while working, note it in `submission.md` instead.
- Remove all debug prints/temporary code before committing.
- Preserve the existing code style: docstrings on service functions, `ValueError` for not-found/invalid input, `to_dict()` serialization.
