# PR Response Doc - CineLog Watchlist Feature

## AI Usage
I used AI assistance to orient around `models.py`, `services/collection_service.py`, and `tests/test_collection.py`, then verified each summary against the actual code before editing. I also used GitHub CLI output to read the six PR review comments and used AI to organize them into code changes, tests, and written design decisions. After drafting the default-visibility and sort-order responses, I used AI as a devil's advocate by asking what a careful reviewer would push back on. That surfaced two useful gaps: public watchlists can still reveal personal taste or intent, and alphabetical sorting becomes useful for large lists. I revised Comments 4 and 5 to acknowledge those tradeoffs directly.

## Comment 1 - Rename
**What I did:**
Renamed the watchlist service function in `services/watchlist_service.py` from `save_to_watchlist()` to `add_to_watchlist()` so it follows the same `verb_to_noun` convention as `add_to_collection()`. I checked the only call site in `routes/watchlist/watchlist.py` and updated both the import and the POST handler call.

**How I verified:**
I used a project-wide tracked-file search with `git grep -n "save_to_watchlist"` to confirm there were no remaining code references to the old name. I also searched `git grep -n "add_to_watchlist"` to confirm the service definition, route import, route call site, and tests all use the new name. Ran `.\.venv\Scripts\python -m pytest tests` and confirmed all tests pass.

## Comment 2 - Deduplication
**What I did:**
I used `add_to_collection()` in `services/collection_service.py` as the model. That function checks `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` before creating a new row and raises `AlreadyInCollectionError` if it finds one. I added the equivalent `AlreadyInWatchlistError` and the same pre-insert query against `WatchlistEntry` in `add_to_watchlist()`. I also added a database-level unique constraint named `unique_user_film_watchlist` for `(user_id, film_id)` and updated the watchlist route to return `409` for duplicate adds.

**How I verified:**
I added `test_add_to_watchlist_duplicate_raises()` to add the same film twice for the same user, assert `AlreadyInWatchlistError`, and confirm only one `WatchlistEntry` row remains. I also added `test_add_to_watchlist_allows_same_film_for_different_users()` to confirm deduplication is scoped to one user's watchlist, not the film globally. Ran `.\.venv\Scripts\python -m pytest tests` and confirmed all tests pass.

## Comment 3 - Missing test
**What I did:**
Created `tests/test_watchlist.py` and used `test_add_to_collection_nonexistent_film_raises()` from `tests/test_collection.py` as the direct model. I copied the same fixture style (`app`, `sample_user`, `sample_film` where needed), used the same fake UUID value, entered `with app.app_context():`, and asserted `pytest.raises(FilmNotFoundError)` around `add_to_watchlist(user_id=sample_user, film_id=fake_film_id)`.

**How I verified:**
I ran the focused test `.\.venv\Scripts\python -m pytest tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises` and confirmed it passed. I then ran `.\.venv\Scripts\python -m pytest tests` and confirmed the new test passes with the full suite.

## Comment 4 - Default visibility
**My position:**
Keep the current default of `public=True` for watchlist entries.

**Reasoning:**
CineLog is framed as a community film tracking app, not a private personal database. The behavior I am optimizing for is social discovery: a user saves films they want to watch, friends can see those intentions, and the watchlist becomes a conversation starter or recommendation surface. A public default makes the new feature line up with that product direction without requiring users to discover and change a setting before sharing anything.

I also see watchlist visibility as different from ratings or watched-history metadata. A rating can reveal a stronger personal opinion, and a watched log can reveal past behavior. A watchlist usually represents future interest, so it is still personal, but it is a less sensitive default-sharing surface than reviews, ratings, or private notes.

**Tradeoff acknowledged:**
The tradeoff is user expectation and privacy. Some users treat "save for later" as private by default, and a public watchlist can still reveal taste, identity, mood, or intent. A private-by-default design would be more conservative and would reduce surprise. I am keeping `public=True` because it better supports CineLog's community behavior, but I would want the UI to make the visibility clear and eventually let users override it per entry or at the account level.

## Comment 5 - Sort order
**My position:**
Use date-added descending order for watchlists.

**Reasoning:**
I implemented the maintainer's preference: `get_watchlist()` now sorts by `WatchlistEntry.date_added.desc()`. The behavior I am optimizing for is returning to recently saved films. When a user adds something to a watchlist, the most likely immediate follow-up is "what did I just save?" or "what have I been meaning to watch lately?" Newest-first keeps those recent intentions at the top.

This also matches the existing collection behavior in `get_collection()`, which returns newest entries first. Keeping collection and watchlist ordering consistent makes the app easier to reason about: both user-specific film lists prioritize recent activity rather than catalog browsing.

**Engagement with reviewer's point:**
I agree with the reviewer that most users will care more about recent additions than alphabetical order in the default view. Alphabetical sorting is useful once a watchlist becomes large and the user is trying to find a known title, but that is a browsing/search problem rather than the best default ordering. I changed the implementation from `Film.title.asc()` to `WatchlistEntry.date_added.desc()` and would treat alphabetical sorting as a future optional sort mode if the API grows query parameters.

## Comment 6 - Rebase
**What conflicted:**
The review conflict was between the watchlist branch's pre-refactor assumptions and updated `main`: the original watchlist work still treated film IDs as integers, while `main` had migrated `Film.id` and film foreign keys to UUID strings. The conflict area was the model/service boundary: `WatchlistEntry.film_id`, watchlist route body documentation, and watchlist service arguments needed to line up with the UUID-based `Film` model from `main`.

**How I resolved it:**
I fetched the updated branch with `git fetch origin` and rebased with `git rebase origin/main`. On the final run Git reported `Current branch feature/watchlist is up to date` because the branch had already been rebuilt on top of `origin/main`. The resolution in the rebased branch keeps `Film.id` and `CollectionEntry.film_id` from `main` as `db.String(36)` UUID fields and adds `WatchlistEntry.film_id` as `db.String(36)` as well. I also updated the watchlist route docs to show `{ "film_id": "<uuid>" }` and kept `add_to_watchlist()` using the UUID film ID passed through to `db.session.get(Film, film_id)`.

**How I verified no conflict remains:**
I checked the branch history with `git log --oneline --merges origin/main..HEAD`; it returned no commits, so the feature branch has no merge commits after the rebase. I searched tracked files for stale watchlist/integer-ID references including `film_id (int)`, `db.Integer, db.ForeignKey("film.id")`, and `autoincrement=True`; none remain in the rebased watchlist code. I then ran `.\.venv\Scripts\python -m pytest tests` and confirmed all tests pass.

**Commit history screenshot:**
![Commit history screenshot](<Screenshot 2026-07-12 185138.png>)

## Stretch - remove_from_watchlist()
**What I did:**
Added `remove_from_watchlist(user_id, film_id)` and `NotInWatchlistError`, following the same pattern as `remove_from_collection()`. I also exposed it through `DELETE /watchlist/<user_id>/remove` for parity with the collection route.

**Why:**
Watchlists need the same lifecycle as collections: users can add a film by ID, list saved films, and remove an entry later. Returning a domain-specific exception keeps service behavior predictable and easy for routes to translate into HTTP responses.

**How I verified:**
Added tests for successful removal and for removing a film that is not on the user's watchlist. Ran `.\.venv\Scripts\python -m pytest tests` and confirmed 11 tests pass.

## Stretch - Additional test
**What I tested:**
Added `test_add_to_watchlist_allows_same_film_for_different_users()`.

**Why I chose it:**
The duplicate rule should prevent one user from saving the same film twice, but it should not prevent different users from saving the same film. This edge case verifies the uniqueness rule is correctly scoped to `(user_id, film_id)`.

**How I verified:**
Ran `.\.venv\Scripts\python -m pytest tests` and confirmed 11 tests pass.

## PR Description
This PR adds a watchlist feature so users can save films they want to watch later and retrieve that list through the watchlist service and route. It follows the existing collection service conventions: `add_to_watchlist()` validates the film, rejects duplicates, persists one entry per `(user_id, film_id)`, and `get_watchlist()` returns film dictionaries with watchlist metadata. The stretch `remove_from_watchlist()` path lets users remove saved films later.

Design decision 1 - default visibility: watchlist entries remain `public=True` by default because CineLog is a community film tracking app and public watchlists support sharing, discovery, and recommendations. I documented the privacy tradeoff in the response doc: public watchlists can reveal taste or intent, so the UI should make that visibility clear and future settings should allow users to override it.

Design decision 2 - sort order: watchlists return newest-first by `WatchlistEntry.date_added.desc()`. I implemented the maintainer's preference because recent saves are the most likely items a user wants to revisit, and this matches the collection service's newest-first behavior. Alphabetical sorting is still useful for large lists, but it is better as a future optional sort mode than as the default.

Manual testing steps:
1. Start the app with `python app.py`.
2. Create or seed a user and a film in a local app context.
3. Send `POST /watchlist/<user_id>/add` with `{ "film_id": "<uuid>" }` and confirm a `201` response.
4. Send `GET /watchlist/<user_id>` and confirm the film appears with `date_added` and `public` metadata.
5. Repeat the same `POST /watchlist/<user_id>/add` request and confirm it returns `409` instead of creating a duplicate.
6. Send `DELETE /watchlist/<user_id>/remove` with the same film body and confirm it returns `200`.
7. Send `POST /watchlist/<user_id>/add` with an unknown film UUID and confirm it returns `404`.

Validation: `.\.venv\Scripts\python -m pytest tests` passes with 11 tests.
