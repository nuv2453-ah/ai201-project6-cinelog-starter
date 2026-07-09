# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI (Claude) for three distinct roles during this project:

1. **Codebase orientation.** I pasted `models.py`, `services/collection_service.py`, and `tests/test_collection.py` and asked for a walkthrough of what each file does, the naming conventions, and how deduplication was handled in `add_to_collection()`. The output was accurate against the actual code — I verified by reading the source myself before writing my own version of the dedup check in `add_to_watchlist()`. I did not ask AI to write the dedup code for me.

2. **Devil's-advocate on Comments 4 and 5.** After drafting my positions on default visibility and sort order in my own voice, I asked for pushback like a maintainer would give: what am I overclaiming, what tradeoff am I missing, where is my argument thinnest. This surfaced two real gaps: (a) my Comment 4 draft asserted an inference about *why* `public` only exists on `WatchlistEntry` as if it were fact rather than reasoning from an omission, and (b) my Comment 5 draft asserted "watchlists are mostly used the intent-aid way" with no anchor in the codebase. I revised both — softening the Comment 4 claim into an interpretive reading, and grounding Comment 5 in the observable fact that `get_collection()` already sorts newest-first (making alphabetical on the watchlist the outlier, not the default). The positions and the reasoning are mine; AI helped me catch weaknesses before finalizing.

3. **Git mechanics.** I used AI to sanity-check the rebase plan against the maintainer's refactor, work through credential/keychain issues on push, and structure the interactive rebase into pick/fixup/reword steps. I ran every command myself and reviewed each result before continuing.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, updated the docstring, and updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call in `add_film()`).
**How I verified:** Ran `grep -rn "save_to_watchlist" .` after the change — the only remaining match was a stale compiled `.pyc` cache file, not source code, confirming no live call sites were missed. Ran the full test suite (`pytest tests/ -v`) to confirm nothing broke.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a duplicate check in `add_to_watchlist()`, following the exact pattern used in `add_to_collection()` in `services/collection_service.py`: query for an existing `WatchlistEntry` matching `user_id` and `film_id` before creating a new one, and raise if found.
**How I verified:** Ran `pytest tests/ -v` to confirm the existing suite still passes after the change.

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

## Final Commit History

After addressing all six comments, I used `git rebase -i` to clean up my commit history — squashing an inherited helper fix into the initial watchlist feature commit, and folding three iterative doc commits into a single documentation commit. The final linear history (6 commits, all conventional, one logical change each) is captured in the screenshot at `docs/git-log-screenshot.png`:

- f688547 — docs: add PR response documentation
- dc35499 — fix: update watchlist film_id to UUID after main refactor
- dc2317f — test: add test for nonexistent film_id in add_to_watchlist
- 9586477 — fix: add deduplication check to add_to_watchlist
- a9dc34d — fix: rename save_to_watchlist to add_to_watchlist
- ce85305 — feat: add watchlist model and endpoint

## PR Description

**What this PR does:** Adds a "watchlist" feature to CineLog — the counterpart to the existing collection feature. Where a collection tracks films a user has already watched, a watchlist tracks films they want to watch in the future. The feature ships with a `WatchlistEntry` model, an `add_to_watchlist(user_id, film_id)` service function with deduplication, a `get_watchlist(user_id)` service function that returns the user's saved films (sorted newest-first per the design decision below), and two REST endpoints: `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add`.

**Design decisions:**
- **Default visibility (`public` field): `False`.** Watchlists are provisional — items get added impulsively, from suggestions, or without vetting — and should default to private until the user explicitly chooses to share. Full reasoning in the Comment 4 response above.
- **Sort order: date-added, newest first.** Alphabetical is a lookup aid; date-added is an intent aid, which is how watchlists are actually used. Also aligns with `get_collection()`'s existing convention. Full reasoning in the Comment 5 response above.

**How to test manually:**
1. Start the app: `python app.py`
2. Seed the DB with at least one `User` and one `Film` via `flask shell` — note both IDs.
3. Add a film to the watchlist: `curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add -H "Content-Type: application/json" -d '{"film_id": "<film_id>"}'` — expected: 201 Created with the new entry JSON.
4. Try adding the same film again — expected: `AlreadyInWatchlistError` (proves dedup works).
5. Try a nonexistent `film_id` (e.g. `"00000000-0000-0000-0000-000000000000"`) — expected: `FilmNotFoundError` (proves validation).
6. Retrieve the watchlist: `curl http://127.0.0.1:5000/watchlist/<user_id>` — expected: JSON array of films, sorted by date_added newest-first.
7. Run the test suite: `pytest tests/ -v` — expected: 5/5 pass.
