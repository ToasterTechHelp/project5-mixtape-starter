# Mixtape Bug Hunt — Submission

## AI usage

I used Claude to get oriented in the codebase faster. Mostly I pasted the service
files and asked "what does this module do and what are its functions" so I didn't
have to read everything cold, and I had it trace the route → service call chains
(e.g. what happens when you rate a song) so I knew where logic actually lived. I
didn't let it guess the bugs for me — for each one I read the suspicious function
myself and confirmed by running the existing tests. The one thing I double-checked
against the AI was the `datetime.weekday()` return values (it says Monday=0,
Sunday=6), which lined up with the streak bug once I saw the `!= 6` check.

## Codebase map

**Main files**
- `app.py` — Flask app factory (`create_app`). Sets up the `db` (SQLAlchemy) and
  registers 4 blueprints under url prefixes: `/songs`, `/playlists`, `/users`,
  `/feed`.
- `models.py` — the data model. Models: `User`, `Tag`, `Song`, `ListeningEvent`,
  `Rating`, `Playlist`, `Notification`. Plus 3 association tables: `friendships`
  (user↔user), `song_tags` (song↔tag), and `playlist_entries` (playlist↔song),
  where `playlist_entries` also carries `position`, `added_by`, and `added_at`, so
  songs in a playlist have an explicit order, not just insertion order.
- `routes/` — thin HTTP layer (`songs.py`, `playlists.py`, `users.py`, `feed.py`).
  Each route parses the request, calls a service function, and `jsonify`s the
  result. No real logic here.
- `services/` — where all the business logic lives: `streak_service`,
  `feed_service`, `search_service`, `notification_service`, `playlist_service`.
- `tests/` — pytest suites for streaks, search, and playlists, run against an
  in-memory SQLite DB.
- `seed_data.py` — fills the dev database with test data.

**Pattern I noticed:** every route immediately hands off to a service function.
Routes only do request parsing + response formatting; all the actual logic is in
`services/`. So when something's broken in an endpoint, the bug is basically
always in the matching service file.

**Data flow — rating a song:** `POST /songs/<song_id>/rate` hits
`routes/songs.py:rate()`, which pulls `user_id` and `score` off the JSON body and
calls `notification_service.rate_song(user_id, song_id, score)`. That function
validates the score is 1–5, looks up the song and user, then either updates the
user's existing `Rating` row for that song or inserts a new one (there's a unique
constraint on user+song, so it's an upsert), and commits.

## Root cause analysis

### Bug #1 — My listening streak keeps resetting

- **How I reproduced it:** Ran `test_streaks.py::test_streak_increments_on_sunday`
  — it listens on Saturday then Sunday and expects the streak to go to 2, but it
  came back as 1. Same thing happens live: any day you listen where "today" is a
  Sunday, the streak drops to 1 even though you listened the day before.
- **How I found the root cause:** Started from the route
  `POST /songs/<id>/listen` → `record_listening_event` → `update_listening_streak`
  in `streak_service.py`. Read the branch that decides increment vs reset and the
  `today.weekday() != 6` in the increment condition jumped out.
- **The root cause:** `datetime.weekday()` returns 6 for Sunday. The increment
  branch was `days_since_last == 1 and today.weekday() != 6`, so on Sundays the
  "listened yesterday" case failed that condition and fell through to the `else`,
  which resets the streak to 1. Streaks silently died every Sunday.
- **The fix + side-effect check:** Removed the `and today.weekday() != 6` guard so
  the branch is just `days_since_last == 1`. Checked the other streak tests
  (starts at 1, same-day no double count, resets after a skipped day) still pass,
  so normal days and real gaps still behave correctly.

### Bug #3 — The same song keeps showing up twice in search

- **How I reproduced it:** Ran
  `test_search.py::test_search_no_duplicates_multi_tag_song` — searching for a song
  that has 3 tags returned it 3 times instead of once. A song with 1 tag or 0 tags
  came back fine, which is why it looked inconsistent.
- **How I found the root cause:** Traced `/songs/search` → `search_songs` in
  `search_service.py`. Saw the query does an `.outerjoin(song_tags, ...)` but never
  actually uses tags in the filter, which is exactly the kind of thing that
  duplicates rows.
- **The root cause:** The outer join to `song_tags` produces one row per
  (song, tag) pair. A song with N tags becomes N joined rows, and `.all()` returns
  the same `Song` object N times → duplicates. Songs with 0 or 1 tag only produce
  1 row, so they looked fine — that's the "conditional" part.
- **The fix + side-effect check:** Deleted the `.outerjoin(song_tags, ...)` line
  (and dropped the now-unused `song_tags` import). The join wasn't needed — tags
  are already loaded through the `Song.tags` relationship inside `to_dict()`. Ran
  the rest of the search tests (matching songs, single-tag, no-tag, no-match) and
  they all pass, so results are still correct and complete, just no dupes.

### Bug #5 — The last song in a playlist never shows up

- **How I reproduced it:** Ran `test_playlists.py::test_playlist_returns_all_songs`
  — a playlist seeded with 5 songs returned only 4. The missing one was always the
  last by position.
- **How I found the root cause:** Followed `/playlists/<id>/songs` →
  `get_playlist_songs` in `playlist_service.py`. The query and ordering looked
  correct, so I read the return line and saw `songs[:-1]`, which chops off the last
  element.
- **The root cause:** The list comprehension iterates over `songs[:-1]` instead of
  `songs`. That slice drops the final song in position order, so every non-empty
  playlist is missing its last track.
- **The fix + side-effect check:** Changed `songs[:-1]` to `songs`. Checked the
  ordering test (songs still come back in position order) and the empty-playlist
  test (`[]` in, `[]` out) — both still pass, so nothing else regressed.
