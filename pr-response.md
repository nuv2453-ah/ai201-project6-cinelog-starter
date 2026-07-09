# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, updated the docstring, and updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call in `add_film()`).
**How I verified:** Ran `grep -rn "save_to_watchlist" .` after the change — the only remaining match was a stale compiled `.pyc` cache file, not source code, confirming no live call sites were missed. Ran the full test suite (`pytest tests/ -v`) to confirm nothing broke.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a duplicate check in `add_to_watchlist()`, following the exact pattern used in `add_to_collection()` in `services/collection_service.py`: query for an existing `WatchlistEntry` matching `user_id` and `film_id` before creating a new one, and raise if found.
**How I verified:** Ran `pytest tests/ -v` to confirm the existing suite still passes after the change (dedup logic doesn't yet have its own test — added in Comment 3's stretch scope if pursued).

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` from `test_collection.py`. Since no `conftest.py` exists in this repo, I duplicated the `app`, `sample_user`, and `sample_film` fixtures locally in the new test file rather than assuming shared fixtures.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` in isolation (passed), then the full suite `pytest tests/ -v` (5/5 passed).

**Note on test placeholder type:** The placeholder for a nonexistent `film_id` in this test tracks the current schema. When I first wrote this test on the pre-rebase branch, `Film.id` was still an `Integer` and I used `999999`; after the Comment 6 rebase brought in the UUID refactor, I updated the placeholder to `"00000000-0000-0000-0000-000000000000"` to match. The final test uses the UUID string, consistent with `test_collection.py`.

## Comment 4 — Default visibility
**My position:** Switching `public` to default `False`.

**Reasoning:** The case for `True` is real — a watchlist-as-discovery-feed is a legitimate feature — but a watchlist isn't a settled statement the way a completed collection is. It's provisional: half of what's on there got added because a friend mentioned it in passing, or because I was in a mood at 1am, and I haven't vetted it as something I want attached to my name yet. Making that visible by default puts the burden of opting out of exposure on the user, for data they didn't necessarily choose to make legible to others.

The `public` field only existing on `WatchlistEntry` and not on `CollectionEntry` cuts the other way from how it might first read. It's not evidence the app wants watchlists to be the social, outward-facing object — if anything, `Collection` (things you've actually watched, presumably a record you'd stand behind) doesn't need the field at all because nothing about it needs gating. Its absence there suggests visibility control was added specifically because watchlists carry a different risk profile, not because they were meant to be the public-by-default one. Given that, default-private with opt-in sharing is the safer reading of what the field is for.

**Tradeoff acknowledged:** The cost of defaulting private is real — if most users never revisit their settings, the discovery-feed value of watchlists mostly disappears, since there's nothing to browse. I think that cost is worth paying given how much of what lands on a watchlist is unvetted or impulsive; the app can still surface public watchlists for the users who do opt in.

## Comment 5 — Sort order
**My position:** Agreed on date-added over alphabetical.

**Reasoning:** "Alphabetical" isn't wrong so much as it's optimizing for the wrong task. Alphabetical is a lookup aid ("did I already add The Zone of Interest?"). Date-added is an intent aid ("what was I into last week?") — and a watchlist is mostly used the second way: people don't browse it like a catalog, they skim the top of it for what's fresh in their head. It's also worth noting `get_collection()` already sorts newest-first — date-added is the app's existing convention elsewhere in the codebase, and alphabetical on the watchlist is the actual outlier, not the default.

**Engagement with reviewer's point:** Making date-added (newest first) the default addresses the maintainer's point directly. A configurable sort is a reasonable follow-up, but it's out of scope for this PR — the fix here is correcting the default, not building a preference system around it.

## Comment 6 — Rebase
**What conflicted:** The main branch had merged commit `07ca580` (`refactor: migrate film IDs from integer to UUID`) while this PR was open. That commit changed `Film.id` from `db.Integer` (autoincrement) to `db.String(36)` (UUID), and changed `CollectionEntry.film_id` from `db.Integer` to `db.String(36)` to match. My feature branch, meanwhile, added a `WatchlistEntry` class with `film_id = db.Column(db.Integer, ...)` — the pre-refactor type. Rebasing on `origin/main` therefore surfaced a type mismatch: `WatchlistEntry.film_id` was an integer FK pointing at a UUID column.

There was also a redundant `.gitignore` — main added the same file (identical content), so git skipped my `.gitignore` commit during the rebase as an already-applied change. That's expected behavior and no fix was needed there.

**How I resolved it:** Git's auto-merge for `models.py` silently took main's version whole, which meant `WatchlistEntry` was dropped from the file entirely (main didn't have it — the class was added on my branch). The rebase reported "successfully rebased" with no conflict markers, but pytest failed at import time: `ImportError: cannot import name 'WatchlistEntry' from 'models'`. So the resolution was to re-add the `WatchlistEntry` class to `models.py`, this time with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), ...)` to match the new UUID schema. I also updated `tests/test_watchlist.py` — the placeholder `fake_film_id = 999999` needed to become a UUID string (`"00000000-0000-0000-0000-000000000000"`) to match the current `Film.id` type. Committed both changes together as `fix: update watchlist film_id to UUID after main refactor`.

**How I verified no conflict remains:** Ran `git status` (clean working tree), `git log --oneline --graph` (linear history, no merge commits on my branch), and `pytest tests/ -v` (all 5 tests pass — 4 collection tests plus the new `test_add_to_watchlist_nonexistent_film_raises`). Also visually inspected `models.py` to confirm all four model classes are present and every `film_id` field consistently uses `db.String(36)`.

## PR Description
<!-- Written at the end -->
