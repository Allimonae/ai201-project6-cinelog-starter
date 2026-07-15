# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->