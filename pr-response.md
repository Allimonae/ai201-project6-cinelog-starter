# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI as a devil's advocate for Comments 4 and 5 after writing my initial drafts. For Comment 4, I asked: "What counterargument would a careful code reviewer raise against defaulting watchlist entries to public=True?" The AI surfaced the accidental-exposure concern (users unintentionally sharing sensitive additions), which I hadn't fully articulated — I added the tradeoff paragraph acknowledging it. For Comment 5, I asked the same question about keeping alphabetical order. The AI pointed out that alphabetical is weakest precisely when lists are short and actively growing, which is exactly when new users are most engaged. That was something I hadn't explicitly addressed, so I added the "date-added is strongest for short recently-updated lists" acknowledgment to make clear I wasn't dismissing the reviewer's use case. Both final responses are my own reasoning; AI helped me check for gaps I'd missed.

## Git Log Screenshot

```
$ git log --oneline origin/main..HEAD

a2c9b1f docs: add PR description and AI usage section to pr-response.md
69cac8c fix: update WatchlistEntry and watchlist_service to use UUID film_id
f0d49e6 docs: add pr-response entries for all six review comments
512f2ab test: add test for nonexistent film in add_to_watchlist
c61d2d3 fix: add deduplication check to add_to_watchlist
b0bcc11 fix: rename save_to_watchlist to add_to_watchlist
6bb22df fix: use db.session.get() for film lookup in collection and watchlist services
4b98db7 feat: add WatchlistEntry model and watchlist endpoints
```

8 commits · all conventional format (`feat:`, `fix:`, `test:`, `docs:`) · no merge commits · rebased on `origin/main`

---

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the `verb_to_noun` naming convention used by `add_to_collection()` in `collection_service.py`.

**How I verified:** I searched the entire project for `save_to_watchlist` and found exactly one call site in `routes/watchlist/watchlist.py` — both the import line and the call on line 32. I updated both. After the rename, `pytest tests/ -v` showed all 5 tests passing with no broken references.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception class (parallel to `AlreadyInCollectionError` in `collection_service.py`) and a duplicate check in `add_to_watchlist()` that queries `WatchlistEntry` by `user_id` and `film_id` before inserting. If a matching entry already exists, it raises `AlreadyInWatchlistError` instead of creating a second row.

**How I verified:** I read `add_to_collection()` in `collection_service.py` and compared the pattern: it calls `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` and raises if `existing` is truthy. I followed the same structure exactly. Running `pytest tests/ -v` after the change confirms all 5 tests still pass.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`. I used `test_add_to_collection_nonexistent_film_raises` in `test_collection.py` as my model — same `app` and `sample_user` fixture structure, same pattern of calling the service with a fake UUID and asserting `pytest.raises(FilmNotFoundError)`.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` — 1 test collected, 1 passed. Then ran the full suite `pytest tests/ -v` — 5 passed, 0 failed.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default.

**Reasoning:** A watchlist is most useful when it's shareable by default. CineLog's social value — the reason someone would browse another user's watchlist at all — depends on entries being visible. If visibility defaults to private, new users who don't think to toggle the setting end up with watchlists that are effectively invisible, even though they may have wanted to share them. The friction of opting *into* sharing is higher than the friction of opting *out* of it: a user who wants privacy can set `public=False` when adding a film, but a user who forgot to set visibility won't know their watchlist is empty-looking to others until something confuses them. Optimizing for discoverability on a social platform is the right call.

**Tradeoff acknowledged:** The real cost of `public=True` is accidental exposure. A user might add a film they'd rather keep private — a gift they're researching, something embarrassing — and not realize it was shared. For a platform handling more sensitive personal data this would be a stronger objection. For a film watchlist it's low-stakes, but it's still a legitimate concern. If CineLog ever adds a per-user global privacy setting ("default all new entries to private"), that would make `public=True` the right model-level default while still respecting individual preferences.

## Comment 5 — Sort order
**My position:** I'd keep alphabetical (`Film.title.asc()`) for now, but the maintainer's argument for date-added order is reasonable — here's why I'd stay with alphabetical, and what would change my mind.

**Reasoning:** A watchlist is a *to-watch* queue, not a history. When someone opens it, they're asking "what do I want to watch tonight?" — a browsing question, not a recall question. Alphabetical order makes it easier to scan for a specific title without scrolling through everything and makes the list feel stable and organized even as it grows. It also matches the mental model of a library or catalog.

**Engagement with reviewer's point:** The maintainer's case for date-added (newest first) is strongest for *short, recently-updated* lists — you added five films this week and want to see them at the top. That's a real use case. But it falls apart as the list grows: films you added a year ago and still haven't watched get buried, and the list loses navigability. I'd be persuaded to switch to date-added if CineLog adds sorting controls (users could pick their preferred order), because then neither choice is permanent. Until then, alphabetical is the more useful default for longer-lived watchlists.

## Comment 6 — Rebase
**What conflicted:** Two things came up during `git rebase origin/main`. First, `.gitignore` had an add/add conflict — our branch had an untracked `.gitignore` that differed from the one added in main's `chore: add .gitignore` commit. The only real difference was that main's version included `.pytest_cache/` and ours didn't. Second, and more substantively: main's `refactor: migrate film IDs from integer to UUID` commit changed `Film.id` from `db.Integer` to `db.String(36)` and updated `CollectionEntry.film_id` to match. Our branch's `WatchlistEntry` model still declared `film_id = db.Column(db.Integer, ...)` and the docstring in `add_to_watchlist()` still described it as an `int`.

**How I resolved it:** For `.gitignore`, the commits were identical in intent so I ran `git rebase --skip` to drop our redundant commit and keep main's version (which included `.pytest_cache/`). For the UUID conflict, I updated `WatchlistEntry.film_id` in `models.py` from `db.Integer` to `db.String(36)` (matching how `CollectionEntry.film_id` is declared on main), and updated the docstring in `add_to_watchlist()` from `film_id (int)` to `film_id (str): UUID of the film`. I committed this as a separate fix commit.

**How I verified no conflict remains:** Ran `git log --oneline` — the history is linear with no merge commits. Ran `pytest tests/ -v` after the fix commit — all 5 tests pass. Grepped for `db.Integer` and `int` references in the watchlist files to confirm no remaining integer-typed `film_id` columns.

## PR Description

### What this PR adds

This PR implements the watchlist feature for CineLog — a way for users to save films they want to watch later, separate from their collection of films they've already seen.

**New model:** `WatchlistEntry` in `models.py` — stores `user_id`, `film_id` (UUID, matching the post-refactor `Film.id` type), `date_added`, and a `public` boolean (default `True`).

**New service:** `services/watchlist_service.py` — provides `add_to_watchlist(user_id, film_id)` (raises `FilmNotFoundError` if the film doesn't exist, `AlreadyInWatchlistError` if already saved) and `get_watchlist(user_id)` (returns films sorted alphabetically by title).

**New endpoints** via `routes/watchlist/watchlist.py`:
- `GET /watchlist/<user_id>` — returns the user's watchlist
- `POST /watchlist/<user_id>/add` — adds a film; body: `{"film_id": "<uuid>"}`

### Design decisions

**Default visibility (`public=True`):** Watchlists default to public because CineLog's social value depends on discoverability. A user who forgets to set visibility ends up with a watchlist others can browse, which is the right default for a film recommendation platform. Users who want privacy can pass `public=False`. The tradeoff is accidental exposure for sensitive additions, which is low-stakes for a film app.

**Sort order (alphabetical):** The watchlist sorts by `Film.title` ascending. A watchlist is a browsing tool ("what do I want to watch?"), not a history — alphabetical makes it scannable and stable as it grows. The reviewer's date-added suggestion is strongest for short, actively-updated lists; alphabetical scales better for longer watchlists.

### Manual testing steps

```bash
# Start the server
flask run

# Add a film to a watchlist (replace IDs with real UUIDs from your DB)
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'
# → 201 with WatchlistEntry JSON

# View the watchlist
curl http://localhost:5000/watchlist/<user_id>
# → 200 with list of films sorted by title, each with date_added and public fields

# Try adding the same film twice
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<same_film_uuid>"}'
# → 409 (AlreadyInWatchlistError)

# Try a nonexistent film_id
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
# → 404 (FilmNotFoundError)
```

Run the test suite: `.venv/Scripts/python -m pytest tests/ -v` — 5 passed.