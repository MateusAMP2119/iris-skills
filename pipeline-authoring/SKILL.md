---
name: pipeline-authoring
description: Author iris pipelines and lanes — the turn protocol, the fetch frame, script-owned pacing (the engine holds NO time), declaration shape, lane composers and ordering, folder surfaces, and provenance. Use when writing or reviewing a pipeline script, a source-declaring scraper, an iris-declare.yaml, a lane composer, or when deciding how to split/compose lanes and plugin lifetimes.
---

# Authoring iris pipelines and lanes

## The one law: the engine holds no time

The iris engine schedules nothing. No intervals, no caps, no backoff, no
cron. Every wait in the whole system lives in exactly one place: **your
pipeline script**, inside its turn. If a pipeline should check something
"every N seconds", that N is a `time.sleep(N)` in the script — declared
nowhere, enforced nowhere — every case its own case.

The engine's side of the deal:

- A **successful turn never parks** — producing or quiet, the engine offers
  the next turn immediately. Your script IS the loop body; pace it yourself.
- A **dead-lettered turn parks** the pipeline until an operator acts
  (replay, manual run, drain). Failure is a worklist item, never a retry loop.
- A **gated pipeline** (`depends_on`) turns only after its upstream produced.
  That's gating, not parking — no pacing needed downstream, the upstream's
  rhythm cascades.
- A **hung run holds its lane forever**. No engine timeout, by design; only
  an operator cancel frees it. Other lanes keep dispatching.

Consequence: **a script that answers instantly-quiet in a loop builds a
CPU fire.** If your turn can end quiet (usually: nothing new), sleep BEFORE
answering or before asking for the next fetch. The sleep is your pace.

## The turn protocol (what your script speaks)

JSON Lines. stdin = engine frames, stdout = your frames, stderr = free log.

Engine sends: `go` (turn opens, echo its number), `row` (input rows from your
declared reads), `run` (input complete — your cue to act), `source` (answer
to your fetch), `res` (answer to your plugin call).

You send: `row` (output, only declared tables/fields), `fetch` (fetch my
declared source NOW), `call` (declared plugin verb), then exactly one
`done`/`error` echoing the turn number.

### The fetch frame — script times, engine fetches

```
you:    {"event":"fetch"}
engine: {"event":"source","url":"...","status":200,"changed":true,"body":"..."}
```

- Only a pipeline declaring a `source:` block may send `fetch`; one at a time.
- The engine does the conditional GET (ETag/If-Modified-Since persist across
  turns), digests the body, records provenance (url, status, sha256) in the
  run capture. Your script never touches the network for its source.
- `changed:false` (304 or identical body) carries no body → answer `done`
  quiet, zero cost anywhere.
- Failure answers `status` 0 (transport) or the HTTP status, no body. YOU
  decide: stay quiet and try next turn, back off longer, or `error` out.
  The engine never retries and never backs off on your behalf.

### The canonical self-paced scraper turn

```python
# after "run" arrives:
if not first_turn:
    time.sleep(CHECK_EVERY_SECONDS)   # your pace, your case
print(json.dumps({"event": "fetch"}), flush=True)
# on the "source" answer:
#   changed → parse, emit rows, done
#   else    → done (quiet)
```

First turn of a session: fetch immediately (catch up on wake), sleep on the
later ones. Sleeping mid-turn is legal and free — the engine has no run
timeout by design; the lane simply waits.

That sketch is the whole shape of a script: **one flat function, one job, no
nested definitions**. Choosing the number in `CHECK_EVERY_SECONDS`, finding
an endpoint worth fetching at all, and the script-shape rule itself are their
own subject → **[references/scraping.md](references/scraping.md)**.

## Declaration shape (iris-declare.yaml)

```yaml
name: sic_feed             # required; must match the folder name
run: [python3, main.py]    # required; the exec vector
lane: news                 # the lane this pipeline joins
logs: {split: true, stamp: true}   # declare the recording contract
source:
  http: https://sicnoticias.pt/sitemap/news.xml
writes:
  - table: news.articles
    fields: [url, published_at, title, site]
```

Eleven fields, no more: `name, run, env, env_file, lane, logs, plugins,
source, reads, writes, depends_on`. An unknown key is rejected at apply.

- `source.http` is the ONLY source field — no interval exists in YAML, ever.
- Declare exact write fields; a row outside them dead-letters the turn.
- Downstream stages use `depends_on: [sic_feed]` + `reads:` — they inherit
  the upstream's rhythm, no pacing of their own. `depends_on` is a data gate,
  independent of lanes: it works across lanes and never reorders a walk.
- `plugins:` binds alias → `name@version` plus a lifetime (`run` is the
  default and the only one that executes today; `lane` and `resident` parse).

### Workspace layout

```
pipelines/<lane>/iris-declare.yaml       ← the lane composer
pipelines/<lane>/<pipeline>/             ← one declaration + its script
schemas/<schema>/<table>/                ← table schemas
```

Folder position is itself a declaration: the lane folder a pipeline sits in
names its lane, and must agree with any inline `lane:`.

## Lanes

A lane is the unit of parallelism and serialization: **one goroutine per
lane, members walked serially in composer order, distinct lanes in parallel
with no engine cap.** Order sequences; it never gates — a dead-lettered
member does not stop its lane's walk.

Keep one site's fetch→transform chain in one lane. Composer shape, the 2+
interlock, folder surfaces, and the single-member trap (a `lane:` with one
member is nominal — its pipeline lands in the shared serial queue, not a
lane of its own) → **[references/lanes.md](references/lanes.md)**.

## Provenance

The run capture records what was fed (source url/status/sha256 digest —
never the body) and what you wrote. Carry source lineage into your rows when
it matters downstream: a `source_url` or `fetched_sha` column costs little
and answers "where did this article come from" forever.
