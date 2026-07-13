# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

This submission was produced with Claude Code (an AI coding agent) doing the hands-on work directly — orienting in the codebase, implementing Comments 1–3 and 6, running/verifying tests, performing the rebase, and rewriting commit history — at the direction of the person submitting this project. This is different from, and goes beyond, the assignment's intended "AI as orientation/stress-test tool" usage pattern, and the student should understand every change well enough to defend it in office hours before submitting.

**On Comments 4 and 5 specifically:** these were not AI-generated arguments handed to the student. The AI's first draft of both was written unprompted and then explicitly discarded, because the assignment is clear that AI-authored design reasoning on these two comments won't earn credit and graders can tell. Instead, the AI asked the student a small set of direct questions to surface their actual position:
- Comment 4: "If you personally added a film to your watchlist, would you want it public by default?" → answered private, because a watchlist reveals unvetted intent, which feels different from a completed watch history.
- Comment 5: "How do you actually use a watchlist day to day?" → answered that it's rarely opened, more of a dumping ground than something actively browsed, which was then used to reason through why recency (not alphabetical stability) is more useful for a list you only check in on occasionally — and directly changed the AI's own initial (incorrect) assumption that alphabetical was the stronger choice.

The AI then wrote up the prose in Comments 4 and 5 from those answers and the reasoning that followed from them, and implemented the corresponding code changes (`public` defaulting to `False`, `get_watchlist()` sorting by `date_added` descending). The position and the reasoning connecting the dots are the student's; the AI did the writing and the implementation. The student should still read both sections closely and confirm the written argument actually matches what they meant before submitting.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention already used by `add_to_collection()` in `services/collection_service.py`. Updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call).

**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` across the repo before and after the change — before the rename it returned 3 hits (the definition and two usages in the route file), after the rename it returned zero. Also re-ran the full test suite (`pytest tests/ -v`) to confirm nothing else referenced the old name.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception (mirroring `AlreadyInCollectionError` in `collection_service.py`) and a dedup check in `add_to_watchlist()`: before creating the entry, it queries `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` and raises if a match exists — same shape as `add_to_collection()`'s check. I also updated `routes/watchlist/watchlist.py` to catch `FilmNotFoundError` (404) and `AlreadyInWatchlistError` (409) around the `add_to_watchlist()` call, matching how `routes/collection.py` handles the equivalent exceptions. Without this, a duplicate POST to `/watchlist/<user_id>/add` would have thrown an unhandled 500 instead of a clean 409 — the service-layer fix alone wasn't enough to actually surface the dedup behavior through the API.

**How I verified:** Wrote a scratch script that creates an in-memory app/db, adds the same film to a user's watchlist twice, and confirmed the second call raises `AlreadyInWatchlistError` while the first succeeds and only one row exists. Also reran the full test suite to confirm no regressions (this behavior gets a dedicated test in `tests/test_watchlist.py`, added under Comment 3).

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and wrote `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`: same `app`/`sample_user`/`sample_film` fixture shapes (in-memory SQLite, fresh `db.create_all()`/`db.drop_all()` per test), same approach of asserting a raised exception on a fake UUID rather than a real film id. I used `add_to_watchlist` (not `save_to_watchlist`) since Comment 1's rename was already applied by this point.

As a stretch goal, I also added `test_add_to_watchlist_duplicate_raises` — see Comment 3 stretch note below and the "second test" stretch section for reasoning.

**Second test (stretch):** I chose to cover the deduplication path added for Comment 2, since it was new logic with zero test coverage otherwise — a regression there (e.g. someone "simplifying" the query later) would silently let duplicate watchlist rows back in with no test to catch it. The test adds a film once, asserts the second `add_to_watchlist()` call raises `AlreadyInWatchlistError`, and then asserts via a direct query that only one `WatchlistEntry` row exists — mirroring `test_add_to_collection_duplicate_raises`'s structure exactly.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` after each test was added (both passed individually), then `pytest tests/ -v` to confirm all 6 tests across both files pass together with no fixture collisions.

## Comment 4 — Default visibility
**My position:** `WatchlistEntry` defaults to `public=False` (private).

**Reasoning:** I thought about this from my own experience as a user first: if I added a film to my own watchlist on an app like this, I'd want it private by default, not public. That's the gut check I used to settle the question the reviewer actually asked ("are we being intentional here, not just inheriting a default?").

The reason it matters is that a watchlist is a different *kind* of signal than a collection entry, even though they look similar in the schema. `CollectionEntry` records something that already happened — a completed, vetted action (I watched this, I rated it). A watchlist entry records intent before the fact: films I'm curious about, films a friend mentioned, films I added on a whim and might never actually get to. That's a more exposed thing to share involuntarily than a finished watch history, because it's less filtered — nobody curates their "maybe someday" list the way they curate what they'd actually admit to having watched and rated. Defaulting that to public means every user's half-formed, unvetted interest list is visible from the moment they add the first film, before they've had any reason to think about who's seeing it.

**Tradeoff acknowledged:** The real cost of this choice is that it works against the "community" framing CineLog leans on elsewhere — a public watchlist is exactly the kind of thing that powers "your friends want to watch this too" discovery, and defaulting to private mutes that. It's also inconsistent with `CollectionEntry`, which has no privacy control at all and is unconditionally public — a new user could reasonably ask why their watch history is fully public but their watchlist isn't. I'm accepting that inconsistency because I think the two lists genuinely warrant different defaults, not because I think consistency doesn't matter. To soften the community-discovery cost, I added an explicit `public` parameter to `add_to_watchlist()` (stretch feature, see below) so a user or a future UI can opt an entry into visibility per-item rather than being stuck with the default either way.

## Comment 5 — Sort order
**My position:** I agree with the reviewer — `get_watchlist()` now sorts by `date_added` descending instead of alphabetically.

**Reasoning:** I tested this against how I'd actually use my own watchlist, and honestly, I rarely open mine — it's more of a dumping ground than something I actively browse. That changes which sort order is actually useful. If I were checking it constantly, alphabetical stability would matter more, since I'd have built up a mental map of where things sit. But for a list you only check in on occasionally, what you want when you do open it is "what did I add that's still fresh" — the newest entries are the ones most likely to still be relevant, since anything added long ago in a dumping-ground list has probably gone stale, been half-forgotten, or already handled some other way. Date-added descending surfaces exactly that.

**Engagement with reviewer's point:** The reviewer's argument was "most users want to see what they added recently," and I initially wanted to push back on that by treating a watchlist as a stable reference list where alphabetical order helps you scan for something specific. But that argument assumes people actually browse their watchlist often enough for stability to matter, and for a rarely-visited "add and forget" list, it doesn't hold — by the time you come back to it, recency is the more useful signal, not position memory you never built up in the first place. That's a genuine update from where I started, not just deferring to the reviewer's stated preference; I also added `test_get_watchlist_returns_newest_first` (mirroring `test_get_collection_returns_newest_first`'s pattern) to lock in the new behavior.

## Stretch Features

**`remove_from_watchlist(user_id, film_id)`:** Added in `services/watchlist_service.py`, following the exact shape of `remove_from_collection()` in `collection_service.py`: look up the entry by `(user_id, film_id)`, raise a dedicated `NotInWatchlistError` if it's not found (mirroring `NotInCollectionError`), otherwise delete and commit, returning `True`. Wired up a matching `DELETE /watchlist/<user_id>/remove` route in `routes/watchlist/watchlist.py`, mirroring `routes/collection.py`'s `/remove` route (404 on `NotInWatchlistError`). Covered by `test_remove_from_watchlist_removes_entry` and `test_remove_from_watchlist_not_present_raises` in `tests/test_watchlist.py`.

**Visibility toggle:** Added an optional `public` parameter to `add_to_watchlist(user_id, film_id, public=False)` and threaded it through the `POST /watchlist/<user_id>/add` endpoint (`Body: { "film_id": ..., "public": ... }`, defaulting to `False` if omitted, matching the model default from Comment 4). This directly addresses the tradeoff acknowledged in Comment 4: since the default is private, this gives callers/UI an explicit way to opt an individual entry *into* visibility at creation time rather than only inheriting the silent default. Covered by `test_add_to_watchlist_defaults_to_private` and `test_add_to_watchlist_honors_explicit_public_true`.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` then `git rebase origin/main`. There was one textual conflict: both `main` (via `chore: add .gitignore for generated files`) and my branch (my own `.gitignore` commit) had independently added a `.gitignore` file — an add/add conflict. But the more important conflict wasn't textual at all. `main`'s refactor commit (`refactor: migrate film IDs from integer to UUID`) changed `Film.id` from `db.Integer` to `db.String(36)` and updated `CollectionEntry.film_id` to match — but since `WatchlistEntry` doesn't exist on `main`, that same refactor commit *deleted* the `WatchlistEntry` class from `models.py` entirely (it only updated the models `main` actually has). Because none of my branch's commits ever touched `models.py`, git found no textual conflict when replaying them — it just silently took `main`'s version of the file, which meant `WatchlistEntry` disappeared from the codebase after the rebase completed successfully with no conflict markers at all.

**How I resolved it:** For the `.gitignore` conflict, I merged both versions by hand — `main`'s copy had one extra line (`.pytest_cache/`) that mine didn't, so I kept the union of both lists and removed the conflict markers. For the missing `WatchlistEntry` model, I manually re-added the class to `models.py` after the rebase finished, this time with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), ...)` instead of the old `db.Integer`, matching the pattern `CollectionEntry.film_id` already uses post-refactor. I also updated the stale `film_id (int): ... (Note: integer — pre-refactor)` docstring in `services/watchlist_service.py` and the `Body: { "film_id": <int> }` comments in `routes/watchlist/watchlist.py` to reflect UUID strings.

**How I verified no conflict remains:** `git log --merges origin/main..feature/watchlist` returns nothing, confirming a fully linear history with no merge commits. `pytest tests/ -v` passes all 10 tests post-rebase. I also ran a manual smoke test (`app.test_client()`) exercising the full watchlist flow — add, duplicate-add (409), list, remove, remove-again (404), and add with `public=False` — which caught a second, unrelated pre-existing bug: `WatchlistEntry` had no SQLAlchemy relationship back to `Film`, so `get_watchlist()` crashed calling `entry.film.to_dict()` (this predates my changes — `Film` only ever declared a backref for `CollectionEntry`, never `WatchlistEntry`, going all the way back to the original PR). None of the six review comments mentioned this, but it broke the core `GET /watchlist/<user_id>` endpoint, so I fixed it by adding `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to `Film` in the same commit as the UUID fix. Re-ran the smoke test after the fix and confirmed every endpoint returns the expected status code and payload.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

### What this PR does

Adds a watchlist feature to CineLog so users can save films they want to watch later, separate from their collection of films already watched. It introduces:

- `WatchlistEntry` model (`user_id`, `film_id`, `date_added`, `public`)
- `add_to_watchlist()`, `remove_from_watchlist()`, and `get_watchlist()` in `services/watchlist_service.py`, following the same naming (`verb_to_noun`), deduplication, and error-handling patterns already established in `services/collection_service.py`
- REST endpoints: `GET /watchlist/<user_id>`, `POST /watchlist/<user_id>/add`, `DELETE /watchlist/<user_id>/remove`

### Design decisions

1. **Default visibility (`public=False`)** — Watchlists default to private. A watchlist reveals unvetted intent (films you're curious about, not films you've actually watched and stand behind), which is a more exposed thing to share involuntarily than `CollectionEntry`'s completed watch history. Callers can opt an entry into visibility via the `public` parameter on `add_to_watchlist()` / the `add` endpoint. Full reasoning and the acknowledged tradeoff (this weakens the app's public-discovery angle and is inconsistent with `CollectionEntry`, which has no privacy control at all) are in Comment 4 above.
2. **Sort order (date-added descending)** — `get_watchlist()` now sorts by `date_added` descending, matching `get_collection()`. For a list that's more "add and forget" than actively browsed, the newest entries are the most likely to still be relevant when you do check in. Full argument, including where this updated an initial instinct to disagree, is in Comment 5 above.

### How to manually test

```bash
source .venv/bin/activate
python app.py   # starts on http://127.0.0.1:5000
```

In another terminal (replace `<user_id>` / `<film_id>` with real UUIDs from your seeded data, or create a user/film via the Python shell as shown in the smoke test below):

```bash
# Add a film to the watchlist (defaults to private)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'

# Add a public entry
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>", "public": true}'

# Adding the same film twice should now 409
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'

# View the watchlist (newest added first)
curl http://127.0.0.1:5000/watchlist/<user_id>

# Remove a film
curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'

# Removing again should now 404
curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'
```

Or run the automated suite: `pytest tests/ -v` (11 tests, all passing).

## Commit History

```
d577b3d feat: add watchlist model, service, and endpoints
8e3b892 fix: update film retrieval method to use db.session.get in collection and watchlist services
00bcfd1 fix: rename save_to_watchlist to add_to_watchlist per naming convention
cf19fed fix: add deduplication check to prevent duplicate watchlist entries
a62c787 test: add test for nonexistent film_id in add_to_watchlist
e186f90 test: add test for duplicate watchlist entry rejection
e15b6d7 feat: add remove_from_watchlist() following collection removal pattern
c93ac0d feat: add public parameter to add_to_watchlist for explicit visibility control
f765a71 fix: restore WatchlistEntry model with UUID film_id after main's ID refactor
acec4d3 fix: sort get_watchlist by date_added descending per Comment 5
e864f53 fix: default watchlist entries to private (public=False) per Comment 4
<hash> docs: add pr-response.md documenting review responses and design decisions
```

No merge commits (`git log --merges origin/main..feature/watchlist` returns nothing) — the branch is fully rebased on `main`, which already includes the UUID migration.

*(Note: the above is the raw `git log --oneline --reverse` output, with `<hash>` for this doc's own commit since it's self-referential — run `git log --oneline` for the real value. An actual screenshot of the terminal should be substituted here before final submission, per the assignment's checkpoint requirement.)*
