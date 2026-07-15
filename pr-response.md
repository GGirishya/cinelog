# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude throughout this project for orientation and verification, not for generating the design reasoning itself:
- Asked Claude to walk through `add_to_collection()`'s deduplication pattern before writing my own version for `add_to_watchlist()` in Comment 2 — I wrote the actual dedup logic myself, following the pattern.
- Used Claude to stress-test my Comment 4 and 5 draft arguments. My first drafts were more casual/conversational; Claude pointed out that I hadn't explicitly acknowledged the tradeoff I was making or engaged directly with the maintainer's stated reasoning. I revised both responses to explicitly name what I was optimizing for and what I was giving up, and to more directly engage with the maintainer's point in Comment 5 rather than just presenting a parallel argument.
- Used Claude to debug environment issues (a Python/conda PATH conflict causing `ModuleNotFoundError`) and to work through interactive rebase mechanics (conflict resolution, fixup/autosquash) — these were tooling/mechanics questions, not design or code content generation.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, and updated the one call site in `routes/watchlist/watchlist.py`.
**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` to confirm no references remained anywhere in the codebase, and ran the full test suite to confirm nothing broke.

## Comment 2 — Deduplication
**What I did:** Added a new `AlreadyInWatchlistError` exception (defined locally in `watchlist_service.py`, matching the domain-specific exception pattern used in `collection_service.py`) and added a duplicate-check query in `add_to_watchlist()`, mirroring `add_to_collection()`'s exact pattern: check film exists → check for existing entry → raise or create.
**How I verified:** Ran the full test suite (all passing), and later added a dedicated test (`test_add_to_watchlist_duplicate_raises`) confirming the error is raised and no duplicate entry is created. Noted that `WatchlistEntry`, unlike `CollectionEntry`, has no DB-level `UniqueConstraint` — so uniqueness here is enforced only at the application layer, consistent with existing precedent in this codebase.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, mirroring the structure of `test_collection.py` — same fixtures (`app`, `sample_user`, `sample_film`), and three tests covering entry creation, deduplication, and the nonexistent-film case, modeled directly on `test_add_to_collection_nonexistent_film_raises`.
**How I verified:** All 3 new tests pass individually and alongside the 4 existing collection tests (7/7 total via `pytest tests/ -v`).

## Comment 4 — Default visibility
**My position:** I'm switching the default to `public=False`.
**Reasoning:** A watchlist and a collection carry different privacy implications even though they're structurally similar models. A collection reflects films a user has already watched — a completed, low-stakes disclosure. A watchlist reflects current intent: what someone is curious about, planning to watch, or hasn't gotten to yet. That's a more revealing signal (mood, niche interests, or something a user is deliberately holding back from view until they've actually watched it), and defaulting it to public risks exposing that without the user consciously deciding to. Given that most users don't revisit default settings after account creation, the safer default is the one that requires an active choice to become more exposed, not less.
**Tradeoff acknowledged:** This does reduce spontaneous discoverability — a big part of CineLog's value could be social, letting friends see what you're excited to watch next. Defaulting to private means that only happens when a user opts in, which adds friction. I think that friction is worth it here because the cost of over-sharing (an unintentionally public watchlist) is higher than the cost of under-sharing (a user has to flip one toggle to go public).

## Comment 5 — Sort order
**My position:** I agree with defaulting to "date added" (descending), but recommend adding an alphabetical sort as a togglable option rather than treating it as an either/or decision.
**Reasoning:** I agree with the maintainer's core point — a watchlist functions like a queue, and most interactions with it are either "what did I just add" or "what's been sitting here forever that I should prune." Date-added ordering directly supports both of those use cases; alphabetical ordering doesn't, since it strips out any sense of recency or intent.
**Engagement with reviewer's point:** Where I'd add to the maintainer's reasoning: alphabetical sort still has real utility once a watchlist grows large (searching for a specific title in a 200-film list is painful without it). Rather than picking one sort order as globally correct, I think the better long-term answer is date-added as the default (matching the maintainer's reasoning) with an explicit UI/query-param toggle for alphabetical as a secondary option. For this PR, I implemented the default change only (`get_watchlist()` now orders by `WatchlistEntry.date_added.desc()`) — the toggle is a reasonable follow-up but out of scope for the comments here.

## Comment 6 — Rebase
**What conflicted:** `.gitignore` (add/add conflict — both branches added the file independently) and `models.py` (the `WatchlistEntry` class didn't exist on the post-refactor `main`, so git flagged it as a conflict against the new UUID-based `Film`/`CollectionEntry` schema).
**How I resolved it:** For `.gitignore`, merged both versions' entries (kept `.pytest_cache/` from main alongside my existing ignores). For `models.py`, kept my `WatchlistEntry` class but updated `film_id` from `db.Column(db.Integer, ...)` to `db.Column(db.String(36), ...)` to match the new UUID `Film.id` type, consistent with how `CollectionEntry.film_id` was already refactored.
**How I verified no conflict remains:** Ran `git status` (clean tree, no unmerged paths) and the full test suite (`pytest tests/ -v`, 7/7 passing) against the new UUID schema. Also manually reviewed `routes/watchlist/watchlist.py` for any lingering integer-ID assumptions (e.g., Flask route converters) — found none, but corrected a stale docstring referencing `film_id` as `<int>`.

## PR Description
This PR adds a watchlist feature to CineLog, allowing users to save films they intend to watch later, separate from their collection of already-watched films. It introduces a `WatchlistEntry` model, an `add_to_watchlist()` / `get_watchlist()` service layer, and REST endpoints for viewing and adding to a user's watchlist.

**Design decisions:**
- **Default visibility:** Watchlists default to `public=False`. Unlike a collection (already-watched films — low-stakes disclosure), a watchlist reveals current intent and interest, which is more sensitive. Defaulting to private protects users who don't revisit settings, while still allowing an explicit opt-in to public sharing.
- **Sort order:** `get_watchlist()` defaults to date-added (descending), matching the maintainer's preference that most users want to see recently added films first, functioning like a queue. Alphabetical sort remains a reasonable secondary option for large watchlists, but is out of scope for this PR.

**Manual testing steps:**
1. Start the app: `python app.py`
2. Create a user and film via direct DB insertion or existing setup routes
3. `POST /watchlist/<user_id>/add` with body `{"film_id": "<uuid>"}` — confirm a 201 response and entry created
4. Repeat the same POST — confirm it returns an error (`AlreadyInWatchlistError`), not a duplicate entry
5. `GET /watchlist/<user_id>` — confirm entries return sorted by most recently added first
6. Run the full test suite: `pytest tests/ -v` — confirm all 7 tests pass

## git log --oneline screenshot
check `git log --oneline.pg`