# Kitchen board

A single-page wall dashboard (index.html, no build step) meant to run
full-screen on an old iPad in the kitchen: clock, today's calendar for two
people, weather, tides, next buses, and a bin/garbage-day banner.

It's a dumb frontend on purpose — all the actual syncing (calendars,
weather, tides, buses, bin day) happens in a Cloudflare Worker that isn't
part of this repo. The page just polls that Worker's `/data` and `/buses`
endpoints and renders whatever comes back.

## Setup

1. **Deploy the Worker.** Use `.env.example` as the list of values it
   needs (calendar IDs, weather/tide/bus API keys, and a `KITCHEN_KEY`
   shared secret) and the "Worker contract" below for the exact response
   shape it must return.
2. **Point the board at it.** Edit `CONFIG.workerUrl` near the top of the
   `<script>` in `index.html` to your Worker's URL. There's no env-var
   support here — it's a static file, so this is a manual edit.
3. **Host it statically** (GitHub Pages, Cloudflare Pages, etc).
4. **Hand the iPad its key, once.** Open the page as
   `https://.../?key=YOUR_KITCHEN_KEY`. The key is saved into that
   browser's `localStorage` and immediately stripped from the address bar
   — it's never written back to the repo or left in history. Without a
   key, the board shows a plain "no access key yet" screen instead of
   guessing.

## `.env.example` vs `.env`

`.env.example` is the checked-in *template* — every value it lists is
either blank or a harmless placeholder, and it's safe for this repo to
stay public with it committed. It exists purely so anyone setting this up
(including future-you) knows what the Worker needs without having to
reverse-engineer it from code.

`.env` (or `.dev.vars` for `wrangler dev`) is the *filled-in* copy with
real values — real calendar IDs, real API keys, the real `KITCHEN_KEY`.
That file is what you actually load into the Worker; it is never
committed, is listed in `.gitignore`, and shouldn't leave your machine
except as Worker secrets. If a real `.env` ever shows up in `git status`
here, that's a bug — stop and don't commit it.

## Keeping names off the public repo

This repo (and its GitHub Pages site) is public, but the board shows
first names. Those never live in committed source: `index.html` only
ever has generic placeholders (`CONFIG.people = { a: 'A', b: 'B' }`), and
the real display names are returned by the Worker's `/data` response
(`people.a` / `people.b`, filled in from `CALENDAR_A_NAME` /
`CALENDAR_B_NAME` in your real `.env`). Since every request to `/data`
requires the `X-Kitchen-Key` header, only someone who already has that
key ever sees the real names — anyone else just gets the "no access key
yet" screen, and the public source/repo never contains them at all.

## Previewing without a live Worker

Open the page with `?demo=1` to render it against built-in demo data
(fake events, weather, tides, buses) instead of fetching anything. Useful
for checking layout/design changes. The real board never falls back to
this data on its own — if the Worker is unreachable or misconfigured, the
board shows genuinely empty state ("Not synced yet.") rather than stale
or made-up entries.

## Worker contract

`GET /data` (headers: `X-Kitchen-Key: <KITCHEN_KEY>`) must return JSON
shaped like:

```jsonc
{
  // Real display names, keyed the same as `cal` below. Optional — falls
  // back to index.html's generic CONFIG.people ('A' / 'B') if omitted.
  "people": { "a": "Person A", "b": "Person B" },
  "weather": {
    "temp": 14, "condition": "Light rain", "high": 17, "low": 11,
    "hourly": [{ "time": "2026-01-01T14:00:00.000Z", "temp": 14, "pop": 70 }]
  },
  "events": [
    { "start": "2026-01-01T08:30:00.000Z", "end": "2026-01-01T09:30:00.000Z",
      "title": "Example event", "cal": "a" }
    // cal: "a" | "b", matching the "people" keys above
    // "allDay": true instead of start/end for all-day events
  ],
  "tides": {
    "station": "Point Atkinson",
    "extremes": [{ "time": "2026-01-01T03:42:00.000Z", "height": 1.2, "type": "low" }]
  },
  "bins": { "date": "2026-01-02T07:00:00.000Z", "streams": ["Green bin", "Garbage"] }
}
```

`GET /buses` (same header) must return:

```jsonc
[{ "route": "4", "dest": "UBC", "mins": [3, 14, 26] }]
```

See `DEMO_DATA` / `DEMO_BUSES` in `index.html` for a fuller worked
example of both shapes.

## Notes

- Bus polling only runs during `CONFIG.busHours` and is capped at
  `CONFIG.busEvery` to stay well under TransLink's request quota.
- The board dims (`body.night`) during `CONFIG.quietHours` and does a
  once-a-day reload at 4am to clear any stuck state.
