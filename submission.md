# Stock Ticker Explorer — Design

Type a company name, get its ticker. Backed by the SEC's official list.
**Design only.** Sized for ~1 hour.

---

## 1. The data (measured 2026-08-27)

`https://www.sec.gov/files/company_tickers.json` — 795 KB, 10,391 rows.

| Fact | Value | So |
|---|---|---|
| Distinct `cik_str` | 8,004 | A company ≠ a row |
| CIKs with >1 ticker | 1,443 (max 32) | Tickers are a child table |
| Distinct `ticker` | 10,391 | Unique *today*. Not a key |
| Titles used by 2+ CIKs | 1 | Title is not a key |

**One company, many tickers** — Alphabet is one CIK with four:

```json
{"cik_str":1652044,"ticker":"GOOGL","title":"Alphabet Inc."}
{"cik_str":1652044,"ticker":"GOOG", "title":"Alphabet Inc."}
{"cik_str":1652044,"ticker":"GOOGN","title":"Alphabet Inc."}
{"cik_str":1652044,"ticker":"GOOGM","title":"Alphabet Inc."}
```

Return rows and "Alphabet" gives four results. So results are **companies**, not rows.

**Titles already collide** — live in the file today:

```json
{"cik_str":2041208,"title":"Tactical Resources Corp."}
{"cik_str":2037786,"title":"Tactical Resources Corp."}
```

Two different registrants, one name. **`cik_str` is the only identifier.**

---

## 2. Storage — SQLite

The SEC file is a snapshot with no history. The DB remembers what the file forgets: renames, delistings.

```sql
CREATE TABLE company (
  cik        INTEGER PRIMARY KEY,   -- the only real identity
  title      TEXT NOT NULL,         -- display text, as-is
  norm_title TEXT NOT NULL,         -- normalized, for matching
  rank       INTEGER NOT NULL,      -- row order in the SEC file
  first_seen TEXT, last_seen TEXT
);

CREATE TABLE ticker (
  ticker     TEXT NOT NULL,
  cik        INTEGER NOT NULL REFERENCES company(cik),
  is_active  INTEGER NOT NULL DEFAULT 1,
  first_seen TEXT, last_seen TEXT,
  PRIMARY KEY (ticker, cik)         -- NOT ticker alone
);

CREATE INDEX idx_ticker ON ticker(ticker);
CREATE INDEX idx_norm   ON company(norm_title);

CREATE TABLE sync_meta (            -- one row
  last_sync TEXT, etag TEXT, last_modified TEXT, source TEXT, error TEXT
);
```

Three decisions:

- **PK is `(ticker, cik)`.** Tickers are unique today but the SEC promises nothing. A `ticker`-only key
  turns a future reuse into a crash or a silent overwrite; this turns it into two returnable rows.
- **No unique constraint on `title`.** It would fail today.
- **`is_active = 0`, never `DELETE`.** A delisted ticker still finds its company, labelled — that's the
  reason to use a DB instead of a dict.

Search runs in memory (8k companies); SQLite is durability + history, reloaded into the index after each sync.

---

## 3. Sync

**Conditional GET.** A refresh sends the stored `ETag`/`If-Modified-Since`; a `304` ends it in one round
trip with no parse and no writes — so a developer can hit refresh freely and it costs nothing when the
SEC file is unchanged.

| Trigger | When |
|---|---|
| First boot | DB empty → seed it |
| Manual | `POST /admin/refresh`, run by a developer |

**No scheduler, no TTL, no background job.** The ticker list is stable enough that polling it on a timer
is machinery guarding against a problem we don't have. A developer refreshes when they want fresh data;
until then the DB is served as-is. Adding a scheduler later is one function.

**Reconcile, don't reload** — one transaction:

1. Upsert `company` on `cik` → a rename updates `title`, keeps identity and `first_seen`.
2. Upsert `ticker` on `(ticker, cik)`, `is_active = 1`, bump `last_seen`.
3. Any ticker not seen this sync → `is_active = 0`. Never deleted.

Truncate-and-reload is shorter and throws away every delisting. One transaction means a mid-sync
failure leaves the last good state, not half a table.

**Fallback ladder:** live fetch → existing SQLite → committed `data/company_tickers.json`.
A failed sync is logged into `sync_meta.error` and shown in `/health`, never fatal.
SEC returns **403 without a descriptive `User-Agent`** — exactly what breaks on a demo machine.

---

## 4. Matching

Normalize titles (matching only; raw `title` is always displayed):
uppercase → strip `/DE/`-style state annotations → strip punctuation → drop trailing `INC CORP CO LTD
PLC LLC HOLDINGS GROUP THE`. So `Apple Inc.` → `APPLE`.

| # | Tier | `matchType` | Score |
|---|---|---|---|
| 1 | Equals an active ticker | `exact_ticker` | 100 |
| 2 | Equals a normalized title | `exact_title` | 95 |
| 3 | Title **starts with** query | `title_prefix` | 80 |
| 4 | Title contains query | `title_substring` | 70 |
| 5 | Equals an **inactive** ticker | `delisted_ticker` | 60 |
| 6 | Fuzzy ≥ 70 (`rapidfuzz.token_set_ratio`) | `fuzzy_title` | `similarity − 15` |

One result per CIK, scored by its best tier. **Sort key: `score` desc, then `rank` asc, then `cik` asc.**
`rank` is the SEC file's own row order, roughly largest-first (rows 0–4 are NVDA, AAPL, GOOGL, MSFT,
AMZN), so "bank" surfaces Bank of America above a micro-cap; it's undocumented, so it affects order
only, never correctness. `cik` is the final tiebreak purely to make the order **total** — without it,
two equal-scoring companies can swap places between requests and the "Show more" button would repeat or
skip a row.

**No ticker-prefix tier**, deliberately: the list shows names, not tickers (§6). Ranking name-invisible
ticker matches above name matches would make the order inexplicable to the user.

Every query scores the full 8k set once, then slices by `offset`/`limit`. Scoring the whole set is
~milliseconds and it's what makes paging stable — computing fuzzy matches lazily "only if needed" would
mean page 2 is ranked against a different candidate set than page 1.

---

## 5. API

**`GET /search?q=&limit=5&offset=0`** — also the browse endpoint: one letter is a valid query.

```jsonc
{"query":"aple","total":1,"offset":0,"hasMore":false,"results":[{
  "cik":320193, "cikPadded":"0000320193",
  "title":"Apple Inc.",
  "tickers":["AAPL"], "inactiveTickers":[],
  "matchType":"fuzzy_title", "score":61,
  "edgarUrl":"https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK=0000320193"
}]}
```

`matchType` lets the UI show an `exact_ticker` hit as an answer and a `fuzzy_title` hit as *"Did you
mean…?"*. `limit` default **5**, cap 50. `hasMore` is what the "Show more" button reads — the UI never
has to guess whether another page exists.

**`GET /company/{cik}`** — canonical fetch by the only real key. Accepts `320193` or `0000320193`.
`404 {"error":"no company with CIK 999"}`.

**`POST /admin/refresh`** — the only way data changes. `{"changed":true,"added":3,"updated":1,"deactivated":2,"took_ms":210}`
or `{"changed":false,"reason":"304"}`.

**`GET /health`** — `{"ok":true,"companies":8004,"activeTickers":10391,"lastSync":"…","source":"sec.gov","lastSyncError":null}`.

---

## 6. UI

One input, one list. **The list shows names only — no tickers**, and only **5 at a time**. Typing
filters by name prefix, live: `N` → the top 5 companies starting with N; `NV` narrows; `NVID` narrows.

A **Show more** button appends the next 5 (`offset += 5`), and disappears when `hasMore` is false. Five
is the right default because a beginner scanning for their company reads five names and stops — a wall
of 300 is a worse answer than five plus a button. Clicking a company opens its detail, and **that** is
where the tickers appear. List finds; detail answers.

Cost, stated plainly: with tickers hidden the two `Tactical Resources Corp.` rows look identical in the
list. Both appear, distinct CIKs underneath; one click apart.

---

## 7. Edge cases

| Input | Behavior |
|---|---|
| `N` | Top 5 companies starting with N, `hasMore:true`; Show more pages through the rest |
| `aapl` / ` AAPL ` | Exact ticker — trim + uppercase first |
| `Alphabet` | **One** result, 4 tickers. Not four results |
| `BRK-A` | Exact ticker. Hyphens are ticker characters, never stripped |
| `Tactical Resources Corp.` | **Two** results, both CIKs. Never silently one |
| Delisted ticker | Found, `matchType:"delisted_ticker"` |
| Rename between syncs | Same CIK, new title, `first_seen` kept |
| empty / whitespace | `400` — not an empty 200 |
| `zzzzzzz` | `200 {"count":0}` — not found is not an error |
| `limit=9999` | Clamped to 50. `offset` past the end → `200 {"total":n,"results":[]}` |
| `/company/abc` | `400` |
| Sync fails | Serve last good data; `/search` never 500s |

Fuzzy is skipped under 3 chars — it would match everything.

---

## 8. Stack & budget

**Python 3.11 · FastAPI · sqlite3 · rapidfuzz · httpx** — ~250 lines.

```
app/  main.py (4 routes) · db.py · sync.py · search.py · static/index.html
data/company_tickers.json     seed snapshot
tests/test_sync.py · test_search.py
```

| Min | Work |
|---|---|
| 0–10 | Schema, seed from snapshot |
| 10–25 | Sync: conditional GET, reconcile txn, deactivation |
| 25–37 | Index build, normalization, tiers 1–5 |
| 37–44 | Fuzzy tier |
| 44–52 | Routes |
| 52–58 | UI + Show more |
| 58–60 | Tests |

Tests assert only what would ship a wrong answer: Alphabet → 1 result; `Tactical Resources Corp.` → 2
CIKs; ticker dropped between syncs → inactive, not gone; rename keeps CIK; `aple` → Apple; empty → 400.

---

## 9. Known limits

- Fuzzy cutoff 70 is untuned — needs a fixture of real misspellings.
- Freshness is manual: if nobody calls `/admin/refresh`, the data is as old as the last one. `/health`
  exposes `lastSync` so staleness is visible rather than assumed.
- `rank` tie-break relies on undocumented file order — order only, never correctness.
- US SEC registrants only. No LSE/TSE/private. The UI should say so.
