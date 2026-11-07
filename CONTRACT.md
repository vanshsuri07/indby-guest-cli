# Contract findings

What this CLI assumes about the IndbyAgent invite contract, and **how much of it has actually been reproduced**.

IndbyAgent (the server) is a separate project and is not part of this repo. The `src/lib/*.ts` / `supabase/` line references in the original notes pointed into the server's source, which was read, not run. Those claims are kept below but labelled as such.

Two statuses are used:

- **Verified (CLI side)**: a test in this repo exercises the CLI behaviour. These tests run against fixtures and an in-process stub server, so they prove what the CLI *does with* a given response, not what the real server *sends*.
- **Server claim, not reproduced**: taken from reading server source. Reproducing it needs a running IndbyAgent (`npm run test:integration` against `INDBY_BASE_URL`). In the last run of this repo's checks the 6 integration tests were skipped because no server was reachable, so none of these have been reproduced yet.

## Verified (CLI side)

| # | Finding | CLI behaviour | Test |
|---|---------|---------------|------|
| 1 | Actions are `"METHOD URL"` strings | Split on first space; default GET if no space | `parseAction` |
| 4 | Any field may be absent | Missing `guest` or `actions.rsvp` is rejected; optional fields are omitted from request bodies | `fetchInvite`, `rsvp command` #9 |
| 6 | `party.timezone` may be absent/invalid | Falls back to UTC | `formatTime` |
| 8, 21 | Action URLs use the server's configured origin, not the one you typed | One-line cross-origin warning; **POSTs always go to the origin the user typed**, so a guest token is never sent to a different host | `formatInvite`, `getCrossOriginWarnings`, `rsvp command` #8 |
| 9 | Token shape is 20-40 base64url chars | CLI reimplements the regex; bad tokens and `/j/`, `/m/`, `/v/` links are rejected before any request | `looksLikeToken`, `normalizeInviteUrl` |
| 12, 14 | `hint` and `party.date = "tbd"` | Both are displayed | `formatInvite` |
| 13 | Errors are `{error}` / `{error, issues}` | Printed verbatim; 404, 500 and network failures get distinct messages | `printServerError`, `rsvp command` #12-16 |
| 15 | Server accepts `needs_human` | CLI only allows `--status yes\|no\|maybe`, exit 2 otherwise | `rsvp command` #7 |
| 16, 20 | Replayed idempotency keys are accepted but ignored | CLI sends `Idempotency-Key`; derives it from token + body + current state; warns when the server's stored state differs from what was sent | `deriveIdempotencyKey`, `rsvp command` #1-5, `rank command` #13-18 |
| 17 | Over-max `party_size` is a 400 | Server message printed, exit 1 | `rsvp command` #6 |
| 19 | Rank body: 1-8 entries, optional note, `by` | CLI truncates to 8 and sends `ranking` + `by` | `ranking library` #4, `rank command` #13 |
| 23 | Empty ranking is invalid | If every option clashes, nothing is sent, exit 1 | `rank command` #10 |
| 24 | Clash rules | Touching boundaries do not clash; all-day events block the whole day; cancelled/transparent events ignored; recurrence expanded in a bounded window | `ranking library`, `ics library` |
| n/a | `rsvp_by` is a bare `YYYY-MM-DD` | Displayed without timezone shifting (regression-tested across 6 US zones) | `rsvp_by is a bare calendar date`, `formatDateOnly` |

## Server claims, not reproduced

- **2, 3**: `schema` is descriptive prose, and always lists `rank_poll`/`answer_poll`/`message_host` even when `actions` omits them. (Prose schema and the action set are visible in `test/fixtures/dates-invite.json`; the "always lists" part is not.) The CLI trusts `actions`, not `schema`.
- **5, 7**: Which fields are always present; `rsvp.status` defaults to `"pending"`.
- **10, 11**: Poll JSON shape; `polls_after_yes` count.
- **16, 20 (server half)**: Postgres unique-violation `23505` on a replayed key is swallowed and returns success without updating the row.
- **17 (server half)**: `party_size` defaults to 1 for `yes`, 0 otherwise.
- **18**: `/rsvp` accepts form posts (303 redirect) and JSON (returns invite JSON).
- **19, 22 (server half)**: Strict `RankBody` validation; non-dates polls and closed polls return 400.
- **`rsvp_by` bare-date contract**: the integration test for this exists but was skipped.

When a server is available, any of these can be promoted to "verified" by running the integration suite and noting the server commit.
