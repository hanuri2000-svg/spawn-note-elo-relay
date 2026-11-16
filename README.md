# ELOBOARD relay — canonical source

This repository owns the `spawn-note-elo-relay` Cloudflare Worker source, regression tests and Wrangler configuration. `hanuri2000-svg/spwn-note/worker` is a generated mirror, not a second implementation.

## Change and release

1. Edit `src/index.js` / `src/public-eloboard.js` and extend the relevant tests here.
2. Run `npm test`, then commit to this repository.
3. In `spwn-note/worker`, run `npm run sync:upstream`, `npm test`, and commit the mirror and lock together. The sync records the exact canonical commit and file hashes.
4. Deploy from this repository with `npm install && npm run deploy` using the existing Cloudflare account. Alternatively the mirror's `npm run deploy` checks canonical main over HTTPS and refuses stale/offline/mismatched deployments before testing and invoking Wrangler. Direct `wrangler deploy` bypasses npm guards; do not use it.
5. Check `/health` (expected `1.7.0`) and the four `/api/elo/public/*` routes below after deployment. The legacy routes remain available but depend on upstream JSON endpoints that have returned HTTP 403. For rollback, restore the previous canonical commit, sync the mirror, test and deploy.

CI runs regression tests on pushes and pull requests. The mirror also checks for drift daily and on changes. Only this repository should be connected to Cloudflare automatic builds; build command: `npm ci && npm test`, deploy command: `npm run deploy` (use `npm install` until a lockfile is committed).

## Compatibility and data

### Public HTML statistics (1.7.0)

The new routes read the ordinary `https://eloboard.com/players/ID` pages. They do not call the upstream JSON API, retry rejected requests, execute upstream scripts, or import matches into a user's records. Profile data uses the main race selected by the public profile. Previously saved local records, ladder entries, aliases, notes and account settings are outside this Worker's storage.

Version 1.6.1 uses `redirect: "manual"` and explicitly rejects every HTTP 3xx response without following its `Location`. Cloudflare Workers [does not implement the Fetch standard's `redirect: "error"` mode](https://developers.cloudflare.com/workers/runtime-apis/request/); that mode threw before sending a source request in 1.6.0. The redirect refusal policy is unchanged, and runtime compatibility plus refusal of redirects are covered by regression tests.

Version 1.6.2 allows up to 30 seconds for each single public HTML request, including its body. This accommodates slower server-rendered source pages without adding retries. Clients should allow at least 100 seconds for summary (three pages), 35 seconds for rival (one page), and 220 seconds for recent (at most seven pages); the three routes remain independent.

Version 1.7.0 adds coverage-aware activity summaries for registered players and refreshes successful public snapshots once per Korean calendar day on demand. Activity and recent share one collection of at most 50 matches per player; activity also needs a client deadline of at least 220 seconds for a cold collection. There is no scheduled task or background polling requirement.

| GET route | Result |
| --- | --- |
| `/api/elo/public/summary?playerId=443` | `player`, `overall`, `races`, `types`, `maps`, `opponents` |
| `/api/elo/public/rival?playerId=443&opponentId=513` | `player`, `opponent`, `headToHead` with `from`/`to`, `maps`, `types` |
| `/api/elo/public/recent?playerId=443&limit=50` | `playerId`, `requested`, `complete`, `warning`, `matches` |
| `/api/elo/public/activity?playerId=443` | `playerId`, `asOfDate`, `coverage`, `today`, `currentMonth`, `streak`, `months`; supports any valid positive player ID and never returns a match list |

Every response includes `schemaVersion: 1`, `source: "eloboard"`, `fetchedAt` (ISO UTC), `sourceUrl`, and `stale`. Aggregate rows contain numeric `wins`, `losses`, `games`, `winRate`; race rows add `race`/`label`, type rows add `key`/`label`, map rows add `id`/`label`, and opponent rows add `id`/`name`/`race`. Player ELO is a number or explicitly `null` when the public page marks it unavailable. Unknown races remain empty strings.

Summary reads the profile, maps and rivals pages. It preserves the displayed aggregate totals rather than reconstructing them from a small recent list. Race/map groups can exclude unclassified source records and therefore need not sum to the overall total. `opponents` contains the publicly supplied initial list (currently up to 40); this is not an exhaustive search index. A direct rival lookup does not require the opponent to appear in that list. If the page does not establish a selected pair and its totals, the adapter returns an explicit error instead of invented zero statistics.

Recent matches are currently restricted to player 443 and a limit from 1 to 50. Each row contains `id`, `date`, `opponentId`, `opponent`, `race`, `result` (`W`/`L`), `map`, `type`, and signed numeric `eloChange` (or `null`). This list follows the site's own order, including ties within a date. The collector requests at most seven public HTML pages, using the visible inclusive `to` date filter. The whole previous boundary-day suffix must equal the next page's prefix, and repeated match objects must remain unchanged. Duplicate IDs, changed records, ignored filters, invalid dates and inconsistent profile totals fail validation. When the public daily window saturates or the request bound is reached, `complete: false` and a warning describe the partial result. Statistics never use that partial list as lifetime totals.

Activity reports `coverage: {from, to, complete, allMatches}`. `from`/`to` are the oldest/newest included match dates, or `null` for a verified empty history. `complete` means the latest 50 matches (or all available history) were collected; it does not mean every calendar period is covered. `allMatches` is true only when the complete collected list equals the profile's full match count. Pre-1.7 cached snapshots without that field are treated as `allMatches: false`.

`today` has `date`, `currentMonth` and each `months` entry have `month`; each contains `complete`, `wins`, `losses`, `games`, `winRate`. A period is complete only when a complete latest-match collection extends strictly before its start date, or the whole source history is known. All four numbers are `null` for incomplete periods. `months` includes the snapshot month plus months represented in the collected matches, and does not claim to list the player's full historical months. `streak: {result, count, complete}` uses `W`/`L` (or `null` for no matches); a uniform truncated prefix has `complete: false`, so its count is a lower bound. Dates are based on the actual snapshot `fetchedAt` in UTC+09:00. Future-dated match rows fail validation. A stale snapshot keeps its original `asOfDate` and `today.date`; clients must not present yesterday's values as today's totals.

For example, the verified 443 fixture contains 50 records from 2026-09-28 through 2026-11-14. At its 2026-11-16 snapshot, November is complete at 32 games (10 wins, 22 losses, 31.3%) and October is complete at 13 games (7 wins, 6 losses, 53.8%). September is incomplete and its activity totals are `null`. The newest `WWWL` sequence proves a complete current streak of three wins.

Successful results use bounded Worker memory and Cloudflare's Cache API when available. A snapshot whose `fetchedAt` is on the current Korean calendar day is reused; the first request on a new day refreshes it. Concurrent identical requests share one source sequence, and activity/recent use the same `recent/PLAYER/50` key. The existing v1 successful cache keys are preserved so a same-day 1.6 snapshot survives the upgrade without an unnecessary source fetch. Failed attempts are remembered in bounded memory and the Cache API for that Korean day; subsequent requests return the prior validated snapshot or the saved error until the next day. This is cache-based refresh control, not a global lock across unavailable or independent caches.

A failed refresh can serve a previously validated result no older than 24 hours, with `stale: true`, its original `fetchedAt`, and a warning; the HTTP response is `no-store`. The age bound is checked again after a slow failed refresh. Failures never become zero-filled successful responses, and a prior complete collection is retained instead of an incomplete refresh. Every source fetch has a 30-second deadline, a 5 MB body bound and no retries. No user journals, aliases, ladder records, settings or source data are written by these routes.

CORS preserves `ALLOWED_ORIGIN`, accepts additional comma/space-separated `ALLOWED_ORIGINS`, and permits `https://hanuri2000-svg.github.io` and `https://my-starlog.pages.dev`. An unknown origin is not reflected when an origin allowlist is configured. The existing wildcard behavior remains when `ALLOWED_ORIGIN` is unset or `*`.

`tests/public-eloboard.test.mjs` uses seven compressed, unmodified public HTML responses from diagnostic [run 43843443022](https://github.com/hanuri2000-svg/star-match-manager/actions/runs/43843443022), collected on 2026-11-16. `tests/fixtures/eloboard/provenance.json` records each decompressed SHA-256 and byte count. Tests cover actual totals and 50-record pagination, boundary/duplicate corruption, stale cache behavior, unavailable sources, ID bounds, and CORS without live network access.

### Legacy JSON API compatibility

The additive `/api/elo/profile?player=NAME` route returns only `{player}` using one player lookup, so the home screen does not wait for full match history. Each upstream fetch (including its body) has an 8-second abort deadline and at most two attempts. The complete dashboard remains a separate request; clients must allow for all history pages (60 seconds in the integrated app), and must not treat a history failure as a failed score lookup. During a rolling deployment, an older Worker's exact `404 {error:"not found"}` response permits the integrated client to share its existing full-dashboard request instead.

Worker name, entrypoint, compatibility date/flags, existing API routes and existing response fields are preserved. Detailed statistics are additive. Dashboard queries all official categories and pages through up to 5,000 matches (including an empty sentinel page for exactly 5,000); overflow or invalid upstream payload returns an explicit error instead of partial totals. Overall totals, `totalGames` and detail groups are calculated from the same complete, deduplicated official win/loss match set. Profile fields inside `player` remain unchanged for API compatibility; they can have a different upstream category scope. Supplied race statistics still come from the dedicated upstream race endpoint, with match-derived fallback when a race row is missing. Preview remains sponsored-only. No browser storage, KV, D1, durable objects, secrets or user records are changed.

The source configuration does not prove the deployed account/version. Verify Cloudflare build settings and deployment history with account access; the public `/health` endpoint only reports the running application's version.
