# indby-guest

[![CI](https://github.com/vanshsuri07/indby-guest-cli/actions/workflows/ci.yml/badge.svg)](https://github.com/vanshsuri07/indby-guest-cli/actions/workflows/ci.yml)

A small TypeScript CLI that acts as a **guest agent** for [IndbyAgent](#about-indbyagent) invites: it reads an invite, RSVPs idempotently, and ranks date-poll options against your `.ics` calendar, skipping dates that clash.

No runtime dependencies except [`ical.js`](https://github.com/kewisch/ical.js).

## Install

Requires Node.js 22 (the version this was developed and tested on).

```sh
npm ci
npm run build
node dist/cli.js --help
```

## Usage

```
indby-guest read  <invite-url> [--json] [--timeout s]
indby-guest rsvp  <invite-url> --status yes|no|maybe [--party n] [--note t] [--dietary a,b] [--idempotency-key k] [--json]
indby-guest rank  <invite-url> --ics file.ics [--poll id] [--duration min] [--calendar-tz IANA] [--note t] [--dry-run] [--json]
```

Exit codes: `0` success, `1` runtime/server error, `2` usage error.

### Example

Output below is real, produced against a local stub serving `test/fixtures/dates-invite.json`:

```
$ indby-guest read http://localhost:3999/i/TESTTOKEN_abc123def456_27ch
Maya's 1st Birthday
When:    Date TBD — vote on the dates
Where:   Carnival
Guest:   leo (up to 1 person)
RSVP:    pending
Polls:
  [dates] Which date works best for you? (id: 231ad161-...)
    Options:
      1. Fri, Oct 16, 2026, 9:11 PM GMT+5:30 (Asia/Calcutta)
      2. Thu, Oct 22, 2026, 9:11 PM GMT+5:30 (Asia/Calcutta)
      3. Thu, Oct 29, 2026, 9:12 PM GMT+5:30 (Asia/Calcutta)
...
⚠ Server action URLs use http://localhost:3000, not http://localhost:3999 (rsvp, rank_poll, message_host)

$ indby-guest rank <url> --ics test/fixtures/me.ics --dry-run
Dropped:  Fri, Oct 16, 2026, 9:11 PM GMT+5:30 (Asia/Calcutta) — clashes with event 'Doctor Appointment' (Fri, Oct 16, 8:30 PM – 10:30 PM)
Ranking:  1. Thu, Oct 22, ...
          2. Thu, Oct 29, ...
```

## Design notes

- **Idempotent writes.** `rsvp` and `rank` send an `Idempotency-Key` derived from the token, the request body and the current server state, and skip the POST entirely ("Already set") if the server already holds what you asked for. `yes -> no -> yes` works because the state is part of the key.
- **Tokens stay on the origin you typed.** Invite JSON contains action URLs pointing at the server's configured origin. The CLI warns if that differs from yours but always POSTs to the origin you gave it.
- **Self-correcting errors.** On a server 400, the CLI prints the server's message plus the relevant piece of the invite's `schema`.
- **Calendar clash rules** (defaults and edge cases) are documented in [CONTRACT.md](CONTRACT.md).

## Tests

```sh
npm run test:unit         # 148 tests, no network beyond localhost stubs
npm run test:integration  # needs a running IndbyAgent; skips cleanly if unreachable
```

Integration tests read `INDBY_BASE_URL` (default `http://localhost:3000`).

## Contract findings

See [CONTRACT.md](CONTRACT.md). It separates behaviour verified by this repo's tests from claims about the server that have **not** been reproduced yet.

## About IndbyAgent

IndbyAgent is the invite service this CLI talks to. It lives outside this repository; this repo does not vendor or require it except for the optional integration tests.

## License

MIT
