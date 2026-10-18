# ELOBOARD relay — canonical source

This repository owns the `spawn-note-elo-relay` Cloudflare Worker source, regression tests and Wrangler configuration. `hanuri2000-svg/spwn-note/worker` is a generated mirror, not a second implementation.

## Change and release

1. Edit `src/index.js` and extend `tests/dashboard.test.mjs` here.
2. Run `npm test`, then commit to this repository.
3. In `spwn-note/worker`, run `npm run sync:upstream`, `npm test`, and commit the mirror and lock together. The sync records the exact canonical commit and file hashes.
4. Deploy from this repository with `npm install && npm run deploy` using the existing Cloudflare account. Alternatively the mirror's `npm run deploy` checks canonical main over HTTPS and refuses stale/offline/mismatched deployments before testing and invoking Wrangler. Direct `wrangler deploy` bypasses npm guards; do not use it.
5. Check `/health` (expected `1.5.4`) and `/api/elo/dashboard?player=NAME` after deployment. For rollback, restore the previous canonical commit, sync the mirror, test and deploy.

CI runs regression tests on pushes and pull requests. The mirror also checks for drift daily and on changes. Only this repository should be connected to Cloudflare automatic builds; build command: `npm ci && npm test`, deploy command: `npm run deploy` (use `npm install` until a lockfile is committed).

## Compatibility and data

Worker name, entrypoint, compatibility date/flags, CORS origin, API routes and existing response fields are preserved. Detailed statistics are additive. Dashboard queries all official categories and pages through up to 5,000 matches (including an empty sentinel page for exactly 5,000); overflow or invalid upstream payload returns an explicit error instead of partial totals. Overall totals, `totalGames` and detail groups are calculated from the same complete, deduplicated official win/loss match set. Profile fields inside `player` remain unchanged for API compatibility; they can have a different upstream category scope. Supplied race statistics still come from the dedicated upstream race endpoint, with match-derived fallback when a race row is missing. Preview remains sponsored-only. No browser storage, KV, D1, durable objects, secrets or user records are changed.

The source configuration does not prove the deployed account/version. Verify Cloudflare build settings and deployment history with account access; the public `/health` endpoint only reports the running application's version.
