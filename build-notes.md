# Pinehurst 2026 Trip Site — Build Notes

This file is a project summary. If you're reading this in your own Claude conversation, feel free to ask follow-up questions about any of it — architecture choices, why a particular approach was used, how you'd build something similar for your own trip, etc. The person who built this (Tyler) is happy for you to poke at the details.

## The short version

Two builds, not one:
1. A polished PDF itinerary, built first.
2. A live website, built a few weeks later once the PDF turned out to be the wrong format for something that keeps changing.

Total build time: the PDF took an hour or two. The website took a few hours across a couple of sessions, plus ongoing small tweaks. The whole thing started about three weeks before the trip, not over some long stretch of time.

## Phase 1: the PDF itinerary

Input was everything scattered across texts, calls, and emails: reservation confirmations, notes from a call directly with the resort, flight details collected from each traveler. All of that got dropped into a Claude conversation with instructions to turn it into a single shareable itinerary.

Claude built a day-by-day PDF, and also did the legwork of finding supporting material to make it feel like a real trip dossier rather than a bullet list: course logos, real photos of the courses and restaurants, the season-typical weather for the trip dates, and direct links to each course/restaurant so nobody had to search for where they were headed. A few rounds of feedback tightened up layout and content from there.

## Phase 2: why it became a website instead

A PDF is a snapshot — accurate the moment it's generated, and increasingly wrong after that. Tee times shift, flights get delayed, forecasts change the morning of. Once a PDF is sent out, it has no way to reflect any of that.

The fix was to rebuild the same content as a live web page instead of a static document, so the page itself could stay current without anyone re-sending anything. That also opened the door to things a PDF can't do at all: a live link to the group's scorecard, and photos landing on the site automatically as people took them, so friends not on the trip could still follow along in something close to real time.

## Architecture

- **One self-contained HTML file.** Structure, styling, and behavior all live in a single page — no separate frontend/backend split, no build step.
- **No database, no server.** It's a static file. Anything that looks "live" is live because a background process edits that static file's content directly and republishes it — the page itself doesn't call any API when someone visits it.
- **Design continuity.** The website reuses the color palette and typography from the original PDF, so it reads as the same document, just alive — same cream background, dark green, gold accents, matching typeface pairing (a serif display face for headings, a clean sans for body text).
- **Client-side JS handles anything that doesn't need outside data**: a countdown to the trip, "add to calendar" links generated per event, in-page section navigation.

## The automation ("the routines")

Three independent scheduled routines run in the background, on a recurring basis, regardless of whether anyone is actively viewing the site:

1. **Weather refresh** — periodically calls a weather service for each trip location/date and updates the forecast shown for that day.
2. **Flight status refresh** — on travel days only, checks each traveler's flight against a flight-tracking data source and updates its status (on time / delayed / landed).
3. **Photo scrapbook refresh** — periodically checks a shared Apple/iCloud photo album for new photos, works out which trip day each one belongs to based on when it was taken, and adds it to that day's section on the page.

Each routine is conservative by design: it only edits and republishes the page when there's actually something new to reflect (a changed flight status, a new photo, etc.) — otherwise it does nothing and leaves the page untouched. None of these routines expose any account credentials or API keys in the published page itself; those stay server-side, used only by the routine when it runs.

The 18Birdies scorecard is different from the above three — it's not something this site refreshes. It's just a direct link out to a round that's already live in the 18Birdies app itself.

## Hosting and sharing

The site was originally built and published inside Claude, where a published page is shareable by default only within the builder's own organization/account — fine for internal use, not for sharing with friends and family outside it.

To make it openly shareable, the same page is mirrored to GitHub Pages — a free static-hosting feature of GitHub that serves a page at a normal public URL. Every time content changes (a manual edit, or one of the automated routines above), both copies are updated so they stay in sync. The public link is the one that actually gets shared with the group.

## What this file leaves out on purpose

- Any API keys, tokens, or credentials used by the background routines — never appropriate to share, and not needed to understand how the system works.
- The specific back-and-forth of design revisions (wording tweaks, spacing, image choices) — interesting to nobody but the person who made them.
