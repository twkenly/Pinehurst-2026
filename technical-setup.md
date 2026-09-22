# Pinehurst 2026 — Technical Setup

`build-notes.md` (linked from the "how it was built" page) is the narrative version — good for understanding the shape of the project. This file is the literal one: the actual source, the actual routine prompts, and the actual schedules, so you can copy this instead of re-explaining it to Claude from scratch.

There's no way to make someone a co-owner of a Claude Code session or of a scheduled Routine — both are tied to a single account and can't be shared or transferred, the same way you can't co-own someone else's ChatGPT conversation. So the honest answer to "can I get access to the real thing" is: not the account, but everything the account produced, yes — the whole repo, every commit, and the exact text that drives the automation. That's what's below.

## 1. The actual source

The repo is public: **[github.com/twkenly/Pinehurst-2026](https://github.com/twkenly/Pinehurst-2026)**

That's not a mirror or a summary — it's the real file, with the real commit history. `index.html` is the entire site (one file, no build step, no framework). `/assets` holds every image. Click through the commits and you can watch the whole thing get built, one small change at a time, including every automated weather/flight/photo update along the way.

If you want to build something similar, cloning or forking this repo is a better starting point than a blank page — you get a working structure (the `WX` weather object, the `FLIGHTS` array, the calendar/timeline markup, the chatbot) to adapt instead of inventing it from scratch.

## 2. What you'd need to replicate it

- **A Claude Code account** (claude.ai/code) with your own GitHub account connected, and a repo with GitHub Pages turned on (Settings → Pages → deploy from your default branch).
- **AccuWeather** — used as a first-party connector inside Claude Code itself for the weather routine. No separate signup; it's available directly as an MCP tool if your plan has connectors enabled.
- **An AviationStack API key** (aviationstack.com has a free tier) — only needed if you want live flight-status tracking. Keep this key only in the routine's own runtime — never commit it to the repo, put it in a file, or paste it into a prompt that gets saved anywhere public.
- **A shared iCloud album** (Apple Photos → shared album, with a public link) if you want the auto-scrapbook.

## 3. The three routines, verbatim

These are Claude Code "Routines" — scheduled prompts that wake a persistent session on a cron schedule. All three currently wake the same session that maintains this repo. Cron times are UTC.

### Weather refresh
**Schedule:** `13 11 * * *` — every day at 11:13 UTC (7:13a ET).

**Prompt:**
```
Run the weather refresh routine now, per your standing instructions: check AccuWeather's daily forecast for Pinehurst/Southern Pines NC. Use the days-parameterized forecast tool (not the one capped at 10 days) with a days value comfortably past Oct 4 2026, so Oct 1-4 are actually included once they're within AccuWeather's real forecast horizon. For each trip day (Oct 1 = thu, Oct 2 = fri, Oct 3 = sat, Oct 4 = sun) that the response actually returns a forecast for (match by the real calendar date in the response, e.g. "2026-10-01" for thu -- never by weekday name, and never substitute today's or any other day's forecast for a trip day just because the weekday name matches), update that day's entry in the WX object in index.html (cond/t8/t12/t20), interpolating t8/t12/t20 from the real day/night highs/lows the same way the 2026-09-18 refresh did (documented in that commit) until true hourly precision is available. Leave any trip day AccuWeather doesn't cover yet completely untouched (object left alone for that day). Also keep the glance-weather hardcoded block AND the "Know Before You Go" section's Weather cell (id="info", the .grid3 .cell with k="Weather") in sync with whatever WX data you write -- update its v (temp range) and d (short summary incl. precipitation outlook) text to match.

TONE RULE for any user-facing weather copy you write or rewrite (the #wx-headline summary, each day's WXDETAIL bottomLine, the Know Before You Go Weather cell's d text, and WX.<day>.note) -- lead positive. This is a trip everyone should be excited about, not bracing for. Default framing is upbeat -- "a mix of sun and scattered showers," "the wetter stretches mostly land in the evenings," etc. Never use words like "washout," "panic," "derail," or the rhetorical pattern "nothing points to [bad outcome]" -- even phrased as a denial, naming the bad outcome plants it in the reader's head, which is exactly what to avoid. If the forecast genuinely does turn bad for a day (a real multi-day heavy-rain stretch, not just scattered showers), say so plainly and factually without alarmist language -- the ask is for honest-but-positive framing, not spin that hides a real problem.

IMPORTANT -- there are two timestamps per weather block with different purposes: GENERATED_AT (and DETAIL_GENERATED_AT for the Weather-In-Depth section) record when the numbers LAST ACTUALLY CHANGED -- only bump these when you actually change a day's cond/t8/t12/t20/t8icon/t12icon/t20icon/note/WXDETAIL values. LAST_CHECKED (and DETAIL_LAST_CHECKED) record when you last CONFIRMED the forecast, whether or not anything changed -- update these to the current UTC time on every single run, including no-op runs where AccuWeather returned the same numbers as before. A no-op run should still produce a small commit that bumps LAST_CHECKED and DETAIL_LAST_CHECKED (and only those two fields, or those two plus whatever real data actually changed). Never bump GENERATED_AT/DETAIL_GENERATED_AT without an accompanying real data change.

Additionally, once a trip day is in AccuWeather's window: also pull the hourly forecast for that specific calendar date. The hourly tool currently only returns the next ~12 hours regardless of target date, so true hourly precision for Oct 1-4 will only become available as those dates approach -- check anyway each run, since the window will open eventually. Once real hourly data for a trip day is available: (1) update that day's t8/t12/t20 with the actual hourly readings nearest 8am/12pm/8pm instead of the day/night interpolation; (2) also set t8icon/t12icon/t20icon on that day's WX entry to one of 'sun','partly','cloud','rain','storm','fog' based on the hourly condition nearest each of those times -- this makes each time slot show its own icon on the page instead of one shared day icon; (3) check for anything worth flagging against that day's plans (dinner reservations, tee times) -- notably rain/storm onset, or a sharp temp drop -- and write one short, concrete, non-alarmist sentence into that day's WX.<key>.note field, e.g. "Potential rain starting around 8pm -- bring a jacket or umbrella to dinner." If there's nothing notable, leave note as an empty string ''. This note renders as a "Weather Heads-Up" card on the day's page only when non-empty.

There is also a dedicated "Weather, In Depth" section (id="weather", right after At a Glance) with a WXDETAIL object holding per-day morning (8-11a) / afternoon (12-4p) / night (5-9p) entries -- each with t (temp range string), precip (0-100 number), rain (amount string like '~0.10"'), tag (short condition label like 'Showers'/'Spot Storm'/'Mostly Dry' -- picks its icon automatically via regex: contains "storm" for lightning icon, "rain"/"drizzle"/"shower" for rain icon, "cloud" for cloud icon, otherwise defaults to sun), and desc (one-line detail). Each day also has a verdict ('great'/'good'/'fair'/'rough') and a bottomLine sentence tying the forecast to that day's actual tee time/dinner. When you update a day's daily WX data because AccuWeather's numbers changed meaningfully, also update that day's WXDETAIL entry to stay consistent, and reassess the verdict and bottomLine if the picture meaningfully changed. Also update the #wx-headline summary sentence when you touch WXDETAIL for a real change.

Commit whenever you touch LAST_CHECKED/DETAIL_LAST_CHECKED and/or real data. Before committing, always `git fetch origin main` and check for commits you don't have -- merge cleanly and never blindly overwrite real data with stale or incorrect values.
```

### Flight status refresh
**Schedule:** `1 * 30,4,5 * *` — hourly, but only on the days the prompt actually acts on (the schedule is intentionally loose; the prompt itself is the real gate — see below).

**Prompt:**
```
Run the flight-status refresh routine now, per your standing instructions: if today (America/New_York) is not Sept 30, Oct 4, or Oct 5 2026, do nothing at all and stop immediately. Otherwise check AviationStack for legs actually in progress or recently landed/departed around now and update the `real` field on the matching entries in the FLIGHTS array in index.html, committing only if a status actually changed.
```

### Photo scrapbook refresh
**Schedule:** `1 * 29,30,1,2,3,4,5 * *` — same pattern as above, hourly with the real date check inside the prompt.

**Prompt:**
```
Run the photo-scrapbook refresh routine now, per your standing instructions: if today (America/New_York) is outside Sept 29 - Oct 5 2026, do nothing at all and stop immediately. Otherwise check the shared iCloud album for photos newer than the SCRAPBOOK_SEEN marker in index.html, assign any new ones to the right day bucket by capture time, update the scrapbook and the marker, and commit only if you actually added a photo.
```

**Why the schedule looks looser than the behavior:** all three routines fire more often than they actually act — the cron schedule gets them woken up in the right general window, and the prompt itself double-checks the real date/data before doing anything, so a scheduling mistake just produces a harmless no-op instead of a wrong update. Worth copying this pattern if you build your own.

## 4. Setting this up for your own trip

1. Fork or copy this repo — you get the working HTML/CSS/JS structure to adapt instead of starting blank.
2. Turn on GitHub Pages for your copy (Settings → Pages → deploy from your default branch).
3. Open Claude Code, connect it to your own GitHub account and your copy of the repo.
4. Ask Claude to swap in your trip's actual content (dates, locations, people, courses/restaurants) — same way this one started.
5. Recreate the three routines above under Claude Code's Routines feature: paste each prompt, adjust the dates/location/day-names for your trip, and set your own cron schedule. You'll be prompted for your own AviationStack key when you set up the flight routine — use your own, never share it, and don't let it get written into any file or commit.
6. Fire each routine once by hand to confirm it works before trusting the schedule.

## What's genuinely not shareable

- Claude Code account access, sessions, and Routines — single-account, not transferable.
- API keys — AviationStack, or anything else with a secret in it. You'll need your own for anything that needs one.

Everything else above is real, current, and copy-pasteable.
