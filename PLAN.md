# Geo-Duels Analyzer — Project Review & Finishing Plan

> Generated: 2026-02-25  
> Status: **Draft — awaiting developer sign-off before any work begins**

---

## Architecture Overview

The project is cleanly separated into two packages:

| Layer | Files | Responsibility |
|---|---|---|
| `src/` | `api_client.py`, `models.py`, `processor.py`, `cache.py`, `analytics.py` | Data fetching, parsing, aggregation |
| `ui/` | `styles.py`, `components.py`, `tab_*.py` | All Streamlit rendering |
| `app.py` | — | Entry point, sidebar, filters, tab routing |

---

## What's Fully Implemented ✅

- **API Client** — `GeoguessrClient` handles feed pagination and per-game detail fetching cleanly.
- **Data Models** — `RoundResult` and `GameResult` dataclasses with a `to_flat_rows()` method for clean DataFrame conversion.
- **Processor** — `process_game()` correctly identifies player/opponent teams, extracts ratings via multiple fallback paths, and determines win/loss from health values.
- **Analytics Engine** — All core aggregations exist: country stats, round stats, streaks, rating history, head-to-head, Moving vs NMPZ comparison.
- **Tabs: Overview, Geography, Deep Dive** — All substantially implemented with Plotly charts.
- **Filters** — Mode, map, and date range filters applied in `app.py`.

---

## Gaps & Issues to Fix 🔧

### 1. `ui/tab_data.py` is a stub

This file is far too small to be complete. The README promises a "CSV download of all round-level data", but the tab appears to be a placeholder. It needs:
- A `st.dataframe()` display of the filtered data.
- A `st.download_button()` for CSV export of `filtered_df`.

---

### 2. Sync stopping condition is fragile

In `cache.py`, the sync loop stops at a hard-coded `STOP_DATE = datetime(2023, 1, 1)`. This is inefficient and breaks for users who played before 2023.

**Fix:** Replace the `STOP_DATE` heuristic with an early-exit check — break as soon as all tokens on a feed page already exist in `existing_ids`:

```python
page_tokens = extract_duel_tokens(entries)
new_on_page = [t for t in page_tokens if t not in existing_ids]
if not new_on_page:
    break
```

---

### 3. Opponent nicknames are never stored

`processor.py` saves `opponent_id` but never extracts `opp_player.get("nick", "")`. The `GameResult` model has no `opponent_nick` field, so the Head-to-Head table in `tab_analysis.py` can only display raw UUIDs, not human-readable names.

**Fix:**
1. Add `opponent_nick: str = ""` to the `GameResult` dataclass in `src/models.py`.
2. In `processor.py`, extract `opp_player.get("nick", opponent_id)` and assign it.
3. Include `opponent_nick` in `to_flat_rows()`.
4. Display it in the Head-to-Head table in `ui/tab_analysis.py`.

---

### 4. Player nick lost on cache reload

In `app.py`, `player_nick` is only set during a live sync via `st.session_state`. If a user reopens the app with a pre-populated `games_cache.json` but hasn't synced yet, the header renders blank.

**Fix (choose one):**
- Persist the nick in a small `player_info.json` sidecar file written alongside the cache.
- Or store it as metadata in a dedicated row/column in the cache DataFrame and read it back on load.

---

### 5. Incomplete game outcome handling

In `processor.py`, `game_won` is set to `None` for abandoned games (both health > 0). This `None` flows through all win-rate calculations and streak logic inconsistently — some functions use `.dropna()`, others don't.

**Fix:** In `analytics.py`, ensure `game_win_rate()`, `streaks()`, and `head_to_head()` all explicitly filter `game_won.notna()` before computing stats. Optionally add a `Game Completed` boolean column in `to_flat_rows()` to make filtering explicit at the DataFrame level.

---

### 6. No test suite

There is no `tests/` directory. Given the complexity of `processor.py` (team index resolution, rating extraction fallback chains, round matching), unit tests are essential for safe future changes.

**Suggested coverage:**
- `process_game()` — team swap logic, rating fallback (missing `ratingAfter`, zero base rating), abandoned game handling.
- `streaks()` — edge cases: all wins, all losses, single game, empty DataFrame.
- `sync_data()` — no new games path, partial cache, pagination early exit.
- `get_stats_overview()` — empty DataFrame guard.

---

## Recommended Implementation Order

| Priority | Task | Files Affected |
|---|---|---|
| 1 | Complete `tab_data.py` | `ui/tab_data.py` |
| 2 | Fix sync loop early exit | `src/cache.py` |
| 3 | Add `opponent_nick` | `src/models.py`, `src/processor.py`, `ui/tab_analysis.py` |
| 4 | Persist `player_nick` | `src/cache.py`, `app.py` |
| 5 | Standardise `game_won` filtering | `src/analytics.py`, `src/models.py` |
| 6 | Write test suite | `tests/` (new directory) |
