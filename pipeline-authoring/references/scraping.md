# Scraping: picking a pace, finding an endpoint, shaping the script

Read this when a pipeline declares a `source:` and you must decide *how
often its script sleeps*, *what URL it points at*, and *what may live inside
the script at all*. The pace is a number in your script — nothing in the
engine or the YAML holds it.

## Choosing a pace

Blocking is about REQUEST RATE, not bytes — a 304 counts like a full
download to a rate limiter. Pick the pace from what the site itself says,
in this order:

1. **HTTP freshness headers** on the source answer (the engine obeys
   validators; your pace should match the origin's `cache-control: max-age`
   when it sends one — big outlets do, usually 30–60s for news sitemaps).
2. **The feed's own declared rhythm**: RSS `<ttl>` (minutes) or WordPress's
   `<sy:updatePeriod>`/`<sy:updateFrequency>`. Boilerplate-prone (a weekly
   paper often claims "hourly") but errs toward harmless over-checking.
3. **Nothing declared** → pick a human-plausible reader pace: 1–5 min for a
   busy site, 15–60 min for a small/local one. Sub-minute sustained polling
   of one origin from one IP reads as a bot.

Never poll faster than the origin's CDN cache anyway — you'd get the same
cached bytes and burn goodwill for nothing.

## Backing off on failure

The engine never retries and never backs off for you: a failed fetch answers
`status` 0 (transport) or the HTTP status with no body, and the turn is
yours to finish. Decide in the script:

- **Transient (5xx, 0)** — stay quiet, lengthen the next sleep, try again.
- **429 / 403** — you are being told to stop. Multiply the pace, don't
  hammer; a 403 that persists means the endpoint choice is wrong, not the
  pace.
- **Structural (404, moved feed)** — `error` out and let it dead-letter. A
  parked pipeline is a worklist item an operator sees; a quiet loop against a
  dead URL is invisible forever.

## Finding the right endpoint for a news source

Never scrape the homepage first. Probe in this order (one curl each):

1. `robots.txt` → `Sitemap:` lines. A **news sitemap** (Google News
   requirement, near-universal for professional outlets) updates within
   ~seconds-minutes of publish and is structured XML: url + publication_date
   + title. Best source, tolerated by design.
2. Common paths when robots hides them: `/sitemap/news.xml`,
   `/sitemaps/news.xml`, `/sitemap_news.xml`, `/feed/news/sitemap.xml`.
3. **RSS/Atom**: `/feed/` (WordPress default), `/rss`. Small, parse-friendly,
   ETag-capable. The standard for small sites.
4. Homepage HTML: last resort — heavy, JS-rendered, bot-walled (DataDome,
   Cloudflare). Expect 403s and captchas.

A source that only yields under a headless browser is a signal to stop and
ask whether the site should be a source at all.

## Shape of the fetching pipeline

Keep the fetch pipeline dumb and the parsing narrow:

- One pipeline per origin, declaring that origin's single `source.http`.
- Emit rows as close to the source's own shape as is useful, plus lineage
  (`source_url`, `fetched_sha`) — cleaning and joining belong to downstream
  `depends_on` stages that inherit the fetcher's rhythm.
- Keep the fetch→transform chain of one site in one lane, so the site's
  stages never contend with each other.

## Shape of the script: one function, one thing

**A script is a single function.** Read top to bottom it is one flat
routine — imports, constants, the turn loop, the parse, the emit — and
nothing else. This is a rule of the skill, not a style preference: reject a
script that breaks it, in review and in authoring.

- **No nested definitions.** A `def`, `class`, `lambda` bound to a name, or
  closure declared inside another `def` is not allowed. Nesting hides a
  second unit of work inside the first and makes the turn's control flow
  unreadable at the exact moment it matters (a stuck run holding a lane).
- **No helper cabinet either.** A pile of top-level helpers is the same
  script wearing a disguise. If a step deserves its own name, it deserves
  its own pipeline.
- **One thing per script.** A script fetches, *or* it transforms, *or* it
  publishes. The word "and" appearing in the honest one-line description of
  what a script does is the split signal — and the split is a new pipeline
  folder with `depends_on`, never a new function.
- **Reach outward before reaching inward.** A step that feels too big to
  inline belongs in the stdlib, in a declared `plugins:` verb, or in a
  downstream stage. Those are the three exits; a private helper is not one.

The scraper this file is about is therefore exactly one script's worth of
work — fetch, and emit rows shaped like the source:

```python
# main.py — fetch SIC's news sitemap, emit article rows. That is all it does.
import json, sys, time
CHECK_EVERY_SECONDS = 120

for line in sys.stdin:
    frame = json.loads(line)
    ...          # go / row / run / source, handled inline, one level deep
```

Deduping, enriching, translating, alerting: each is another pipeline reading
this one's table. That split is what buys per-stage replay, per-stage
dead-lettering, and a run capture that says which step actually failed —
all of which a script with private helpers silently gives up.
