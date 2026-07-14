# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used Claude Code (Claude Sonnet 4.6) throughout this project for codebase orientation and code review feedback. Here is a specific account of how it was used:

**Codebase orientation:** I gave Claude the content of `models.py`, `services/collection_service.py`, and `tests/test_collection.py` and asked it to summarize each file's responsibility and trace how `add_to_collection()` works step by step. This helped me understand the deduplication pattern (`CollectionEntry.query.filter_by(...).first()` + custom exception) before implementing it in `add_to_watchlist()`. I verified the summary against the actual code before writing anything.

**Understanding the UUID conflict:** I gave Claude the diff between the feature/watchlist branch `models.py` and the rebased main `models.py` and asked it to explain what the conflict meant. It correctly identified that `film_id` needed to change from `db.Integer` to `db.String(36)` to match the UUID refactor — I confirmed this by reading the refactor commit message and the updated `CollectionEntry.film_id` column in main's models.py.

**Stress-testing design arguments (Comments 4 and 5):** After drafting my responses, I gave Claude each draft and asked "What counterargument would a careful code reviewer raise against this position?" For Comment 4 (visibility default), it surfaced the privacy-first principle, which I already addressed in my reasoning. For Comment 5 (sort order), it asked whether alphabetical sort is more predictable for a long watchlist — I updated my response to engage with that point directly rather than ignoring it.

**Where I did not use AI:** The actual code changes were written by me. AI was used for understanding and stress-testing, not for generating the implementation.

---

## Comment 1 — Rename

**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the import and call site in `routes/watchlist/watchlist.py`. I searched for all occurrences of `save_to_watchlist` across the codebase using project-wide search before committing.

**How I verified:** Ran `grep -r "save_to_watchlist" .` after the change — zero results. Ran `pytest tests/ -v` — all tests pass. The route now imports `add_to_watchlist` and calls it correctly.

---

## Comment 2 — Deduplication

**What I did:** Added an `AlreadyInWatchlistError` exception class and a duplicate check to `add_to_watchlist()` in `services/watchlist_service.py`. The check follows the exact pattern from `add_to_collection()` in `services/collection_service.py`:

```python
existing = WatchlistEntry.query.filter_by(
    user_id=user_id, film_id=film_id
).first()
if existing:
    raise AlreadyInWatchlistError(
        f"Film '{film_id}' is already in this user's watchlist"
    )
```

I also updated `routes/watchlist/watchlist.py` to catch `AlreadyInWatchlistError` and return a 409 Conflict response, matching how `routes/collection.py` handles `AlreadyInCollectionError`.

**How I verified:** `test_add_to_watchlist_duplicate_raises` in `tests/test_watchlist.py` tests this path directly — adds the same film twice and asserts `AlreadyInWatchlistError` is raised and only one entry exists in the DB. All tests pass.

---

## Comment 3 — Missing test

**What I did:** Created `tests/test_watchlist.py`. The required test is `test_add_to_watchlist_nonexistent_film_raises`, which follows the exact fixture and assertion structure from `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`: create a user fixture, pass a fake UUID as `film_id`, and assert `FilmNotFoundError` is raised.

I also wrote additional tests covering: basic entry creation, deduplication, `remove_from_watchlist()`, sort order (newest first), and visibility defaults — see the stretch features section below.

**How I verified:** `pytest tests/test_watchlist.py -v` — all 8 watchlist tests pass. `pytest tests/ -v` — all 12 tests (collection + watchlist) pass.

---

## Comment 4 — Default visibility

**My position:** Keep `public=True` as the default.

**Reasoning:** CineLog is described as a community film tracking app — the social layer is the core product. Users add films to their watchlist to remember what they want to see, but the visible motivation is to share those intentions with their community. The parallel is Letterboxd, where watchlists are public by default because "I want to watch this" is a statement users actively want their friends to see. A private-by-default watchlist would require every user to opt in to the feature's social value, which is backwards for a community app.

The specific behavior `public=True` optimizes for is discovery: when a friend sees a film on your watchlist, they may want to watch it together, or recommend something similar. A watchlist that defaults to private is invisible to the community until the user goes out of their way to change the setting — most users never will, and the social graph loses signal.

**Tradeoff acknowledged:** Privacy-first defaults are generally better practice for new features where user expectations aren't established yet. If CineLog had a significant number of users treating it as a personal tracking tool (not a social one), defaulting to private would be the right call. The `public` parameter exists precisely so callers can override the default — users who want privacy can pass `public=False` explicitly. The default just reflects what the majority of users on a community platform will want.

---

## Comment 5 — Sort order

**My position:** Change the sort order from alphabetical (`Film.title.asc()`) to date-added descending (`WatchlistEntry.date_added.desc()`), matching the sort order of `get_collection()`.

**Reasoning:** The most important reason is consistency: `get_collection()` already sorts by `date_added desc`, and there's no stated reason for watchlist to behave differently. Inconsistency between two parallel features in the same codebase creates cognitive overhead — a developer reading both functions will wonder why they differ and whether the difference is intentional. It isn't.

The user behavior argument also favors date-added: a watchlist is a queue of things I plan to watch, and the most natural mental model is "what did I most recently decide to add?" When I save a film after seeing a trailer, I want to find it easily at the top of my list, not buried alphabetically under a title that starts with a letter near the end of the alphabet.

**Engagement with reviewer's point:** The reviewer's implicit concern with alphabetical sorting is predictability for long watchlists — if you have 200 films and you know the title, A-Z makes it scannable. That's a real advantage for lookup. But that use case is better served by a search or filter endpoint than by the default sort order of the list view. The sort order should reflect the user's default browsing pattern ("what did I recently add?"), not the edge case of searching for a specific title by memory.

---

## Comment 6 — Rebase

**What conflicted:** The conflict was in `models.py`. The main branch's refactor commit (`refactor: migrate film IDs from integer to UUID`) changed `Film.id` from `db.Integer` (auto-increment) to `db.String(36)` (UUID) and updated `CollectionEntry.film_id` accordingly. The feature/watchlist branch added `WatchlistEntry` with `film_id = db.Column(db.Integer, db.ForeignKey("film.id"), ...)` — still using the old integer type. When rebasing, git couldn't automatically merge the two versions of `models.py` because both sides modified it: main changed the Film model, the feature branch added WatchlistEntry.

**How I resolved it:** I kept main's version of `Film`, `User`, and `CollectionEntry` (with `film_id` as `db.String(36)`) unchanged, then added `WatchlistEntry` with `film_id = db.Column(db.String(36), ...)` to match the UUID convention. I also added the `film = db.relationship("Film", lazy=True)` backref that was needed for `entry.film.to_dict()` to work in `get_watchlist()`.

**How I verified no conflict remains:** `git status` shows a clean working tree with no conflict markers. `pytest tests/ -v` passes all 12 tests, including the sort-order test which exercises the `entry.film` relationship through a full DB round-trip. The UUID column type change is captured in its own commit: `fix: update WatchlistEntry film_id to String(36) to match UUID refactor on main`.

---

## Stretch Features

### remove_from_watchlist()

Added `remove_from_watchlist(user_id, film_id)` to `services/watchlist_service.py` and a `DELETE /watchlist/<user_id>/remove` endpoint to `routes/watchlist/watchlist.py`. Follows the same pattern as `remove_from_collection()`: query for the entry, raise `NotInWatchlistError` if not found, delete and commit if found. Tests: `test_remove_from_watchlist_removes_entry` and `test_remove_from_watchlist_not_present_raises` in `tests/test_watchlist.py`.

### Second test — sort order

`test_get_watchlist_returns_newest_first` in `tests/test_watchlist.py` creates two watchlist entries with explicit `date_added` timestamps (one 5 days ago, one now) and asserts the newer one appears first. I chose this case because the original code sorted alphabetically, which is inconsistent with `get_collection()` — this test would have caught that inconsistency and forced a decision. It's the kind of test that encodes an architectural choice rather than just verifying a happy path.

### Visibility toggle

Added a `public` parameter (default `True`) to `add_to_watchlist()` and exposed it through the POST endpoint: callers can include `"public": false` in the request body to create a private entry. Tests: `test_add_to_watchlist_respects_public_false` and `test_add_to_watchlist_defaults_to_public`.

---

## git log --oneline screenshot

```
60c59d0 fix: update WatchlistEntry film_id to String(36) to match UUID refactor on main
edb6939 docs: add pr-response.md with design decisions and PR description
22ec856 test: add watchlist tests for nonexistent film, deduplication, remove, sort order, and visibility
eb88500 feat: add remove_from_watchlist function and public visibility parameter to add_to_watchlist
af765d3 fix: rename save_to_watchlist to add_to_watchlist and add deduplication check
9846768 fix: replace deprecated Film.query.get with db.session.get in collection service
dbdc5c0 feat: add WatchlistEntry model and register watchlist blueprint
bbe206c refactor: migrate film IDs from integer to UUID  ← main
```

7 commits on feature/watchlist above main. No merge commits — linear history after rebase on `origin/main`.

---

## PR Description

### Watchlist Feature

This PR adds a watchlist to CineLog — a separate list from the collection where users can save films they intend to watch. The collection represents films a user has already logged; the watchlist represents films they plan to watch. The two lists are independent: a film can exist in both, in one, or in neither.

**Endpoints:**
- `GET /watchlist/<user_id>` — returns the user's watchlist sorted by date added (newest first)
- `POST /watchlist/<user_id>/add` — adds a film to the watchlist; body: `{"film_id": "<uuid>", "public": true}`
- `DELETE /watchlist/<user_id>/remove` — removes a film from the watchlist; body: `{"film_id": "<uuid>"}`

**Design decisions made:**

1. **Default visibility (`public=True`):** Watchlist entries are public by default. CineLog is a community app; the social value of a watchlist ("I want to watch this") is most useful when visible to friends. Users who want a private watchlist can pass `"public": false` explicitly.

2. **Sort order (date-added descending):** The watchlist sorts by `date_added desc`, matching `get_collection()`. Alphabetical sorting was rejected because it doesn't reflect the user's browsing pattern ("what did I recently decide to add?") and introduces inconsistency between two parallel features in the same codebase.

**How to manually test:**

1. Start the app: `python app.py`
2. Create a user (or use an existing ID from the DB)
3. Create a film via `POST /films` (or use an existing film ID)
4. Add to watchlist:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
5. View watchlist: `curl http://127.0.0.1:5000/watchlist/<user_id>`
6. Try adding the same film again — expect 409 Conflict
7. Try adding a fake `film_id` — expect 404 Not Found
8. Remove from watchlist:
   ```
   curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
9. Add two films at different times, view the list — confirm the most recently added appears first
10. Add with `"public": false` — confirm the returned entry has `"public": false`
