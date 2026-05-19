# UCR Club Discovery — Design

**Date:** 2026-05-18
**Status:** Draft, pending implementation

## Goal

A discovery script that finds UC Riverside student-organization Instagram handles and merges them into the existing `pipeline/accounts.json`, which feeds the IG story scraper in `pipeline/scrape.py`.

`accounts.json` is currently hand-curated (2 entries). The goal is to grow it from UCR's official org directory rather than maintain it by hand.

## Non-goals

- Scraping Instagram for discovery. IG search is rate-limited and bans accounts that look like enumeration. Discovery stays out of IG entirely.
- Fetching IG bios or display names. The existing scraper already pulls per-story metadata; discovery only needs handles.
- Replacing the hand-curated `accounts.json`. Existing entries are authoritative and never modified by this script.
- Automating UCR SSO. The cookie is pasted manually, not minted by the script.

## Source: Highlander Link

`highlanderlink.ucr.edu` is UCR's instance of Engage (Campus Labs), the standard university org-management platform. Every page — the directory, individual org profiles, and the undocumented JSON API — redirects to UCR SSO when unauthenticated.

Recon confirmed:

- `GET /organizations` → 302 to `/Account/Login`
- `GET /organization/<slug>` → 302 to `/Account/Login`
- `GET /api/discovery/search/organizations` → 302 to `/Account/Login`

Auth is therefore required for everything. The script handles auth via a pasted session cookie (see below) rather than automating SSO.

## Approach

Single script, `pipeline/discover.py`. Mirrors the style of `pipeline/scrape.py`: top-level `main()`, `logging` not `print`, env-driven config, polite jitter between requests.

### Auth

User logs into Highlander Link in a browser once, copies the session cookie from DevTools, and pastes it into the `HIGHLANDER_LINK_COOKIE` env var. The script sets that cookie on every request. Cookies typically last 2–4 weeks before re-login is required.

Loaded via `python-dotenv` from `.env.local` (already in `requirements.txt`), or via plain `export`.

If the cookie is expired, the first request returns a 302 to `/Account/Login`. The script detects this via `response.history`, fails loud with a message that names the env var and tells the user to re-paste.

### Data flow

```
HIGHLANDER_LINK_COOKIE env
        │
        ▼
GET /api/discovery/search/organizations?top=100&skip=N  ──► paginated org list
        │   (1–3s jitter between pages)
        ▼
For each org in the list:
    GET /api/discovery/organization/bySlug/<slug>  ──► {socials, description, ...}
        │   (1–3s jitter between orgs)
        ▼
    Extract IG handle:
        1. structured `socialMedia` field where type == "Instagram"
        2. fallback: regex over description text for instagram.com/<handle>
        │
        ▼
    yield { handle, label: org.name, category: map_category(org.categories) }
        │
        ▼
Merge into pipeline/accounts.json:
    - load existing entries
    - index by normalized handle
    - existing entries WIN on conflict (preserves hand-curated labels)
    - new entries appended in alphabetical order by handle
    - atomic write: temp file + os.replace
```

### Handle normalization

`Instagram.com/Ucrvsa/`, `@ucrvsa`, and `ucrvsa` all normalize to `ucrvsa`. Steps: strip URL prefix, strip leading `@`, strip trailing `/`, lowercase. Used as the merge key and as the stored handle.

### Category mapping

Highlander Link returns category names like `"Academic"`, `"Cultural"`, `"Greek Life"`, `"Service"`. Lowercase, replace spaces with hyphens (`"greek-life"`). If an org has multiple categories, take the first. If none, default to `"club"` (matching the existing entries' shape).

### Merge semantics

- Existing entries in `accounts.json` are authoritative. If `ucrvsa` already exists with `label: "UCR VSA"`, a discovery entry with `label: "Vietnamese Student Association at UCR"` does **not** overwrite it.
- New entries are appended, then the full list is sorted alphabetically by handle for stable diffs.
- Re-running the script with no upstream changes produces a zero-line diff.

### Atomic write

Write merged content to `accounts.json.tmp`, then `os.replace()` to the real path. A mid-run crash leaves the existing file untouched.

## Error handling

| Condition | Behavior |
| --- | --- |
| Cookie expired (first request → 302 to login) | `SystemExit` with a message naming `HIGHLANDER_LINK_COOKIE` and how to refresh it |
| 429 or repeated 403 | Sleep 30s, retry once. If it happens twice in a row, abort the run (no partial write) |
| Single org fetch fails | `log.warning(slug, err)`, skip, continue. Mirrors `scrape.py` |
| KeyError / unexpected JSON shape | Log the slug + the offending key, skip. If >10% of orgs hit this, abort — the API contract has shifted |
| Network timeout / ConnectionError | 15s per-call timeout; retry once after 5s; then skip the org |

## Testing

**Unit tests** (`tests/pipeline/test_discover.py`) cover pure functions only:

- `_normalize_handle` for all the input variants above
- `_extract_handle_from_socials` given the structured socials list
- `_extract_handle_from_text` regex fallback over free-text bios
- `_map_category` for single, multiple, and empty category lists
- `_merge_accounts` — existing wins on conflict, alphabetical output, no in-place mutation

**Integration test** — one recorded fixture (`tests/pipeline/fixtures/highlander_link_org.json`) captured from a real Highlander Link response. End-to-end parse → accounts entry. Updates require explicit fixture refresh.

**Out of scope:**

- No mocking of pagination flow — that's testing `requests`, not our code
- No test for atomic write — `os.replace` is the OS contract
- No live network test in CI — would require a valid cookie

**Manual smoke test before commit:** run `discover.py` against a real cookie, eyeball the diff to `accounts.json`, confirm `ucrvsa` and `cyber_ucr` are preserved with their existing labels.

## File layout

```
pipeline/
├── discover.py              # NEW: discovery script
├── accounts.json            # merged in place (existing entries preserved)
├── config.py                # adds HIGHLANDER_LINK_COOKIE env loading
└── requirements.txt         # adds beautifulsoup4 (regex fallback over bio text)

tests/pipeline/
├── test_discover.py         # NEW: unit tests for pure functions
└── fixtures/
    └── highlander_link_org.json   # NEW: one recorded API response
```

## Open questions

None at design time. Implementation will need to confirm:

- Exact JSON shape returned by `/api/discovery/organization/bySlug/<slug>` — recon stopped at the auth wall. The category-mapping and socials-extraction code is provisional until we see real data.
- Whether the `socialMedia` field uses a typed enum (`"Instagram"`) or free-text URLs. The script handles both, but the regex fallback rate tells us which is dominant.

## Cadence

Ad-hoc. Clubs come and go on a semester scale; running discovery once a month or less is sufficient. Not scheduled in GitHub Actions (unlike `scrape.py`, which runs every 4 hours).
