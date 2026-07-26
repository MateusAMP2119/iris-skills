# Lanes: composers, order, surfaces

Read this when composing pipelines into a lane, deciding where a lane
boundary belongs, or debugging an apply that refuses a lane placement.

## What a lane buys you

One goroutine per lane. Members walk **serially in composer order** —
member N+1 starts only after member N reaches a terminal state. Distinct
lanes run **in parallel, with no engine cap and no cross-lane sequencing**.

What the lane deliberately is NOT:

- **Not a gate.** Order sequences, it never propagates failure. A
  dead-lettered member does not stop the walk; the next member still starts.
  Eligibility is `depends_on`'s job, a separate mechanism that carries no
  lane or walk-position field and so can never reorder a lane.
- **Not a data link.** A lane's walk carries an order and nothing else.
  Passing data between members means declared `writes`/`reads` on tables.
- **Not a clock.** A lane loop is watermark-parked: it re-passes only when
  the meta-change sequence advanced since its last pass started. A pass that
  started runs re-passes immediately (its own records advanced the
  watermark) — root pipelines re-run back to back forever, which is why
  pacing lives in the script. A pass that started nothing writes nothing,
  parks, and costs nothing until a cause lands.
- **Not preemptible.** A hung member holds its lane indefinitely (no engine
  timeout — clock doctrine). Only an operator cancel frees it. Other lanes
  are unaffected. That isolation is the real reason to split a lane.

Each pass reads the walk once at pass start, a snapshot: a graph change
mid-pass lands at the next pass, and an in-flight run is never touched.

## The composer

`pipelines/<lane>/iris-declare.yaml` — four fields, nothing else:

```yaml
lane: news                 # must equal the folder name
order: [sic_feed, sic_clean, sic_publish]
reads:                     # optional folder surface
  - table: news.raw
    fields: [url, body]
writes:
  - table: news.articles
    fields: [url, published_at, title, site]
```

A declaration is discriminated by content: carrying `run` makes it a
pipeline, carrying `order` makes it a composer.

Composer rules, each enforced at `iris declare apply`:

- `lane` must equal the folder name the composer sits in.
- Every `order` entry must name a **pipeline folder inside the lane folder**.
  Membership is containment; you cannot order a stranger.
- No repeats in `order` — a name listed twice would run twice per pass.
- A pipeline belongs to **exactly one lane**. A re-apply may not move it, and
  no second lane may claim it.

A composer apply rewrites its lane's whole order atomically. A member apply
never writes lane rows — a pipeline's position always comes from its
composer.

## The single-member trap

**A lane of fewer than two members persists no lane rows.** One member is
nominal: no lane row, so at dispatch that pipeline is unplaced and joins the
shared serial **queue lane** with every other unplaced pipeline, walked
serially in name order.

Consequences worth internalizing:

- Writing `lane: whatever` on a lone pipeline buys nothing at runtime.
- A pipeline that must not wait behind unrelated work needs a real lane —
  two or more contained members — or it shares the queue's single goroutine.
- Omitting `lane:` entirely is the same runtime placement (the queue), it
  just skips the nominal name.

## The 2+ interlock (apply-time errors, decoded)

A pipeline joins a lane inline (`lane:` field), by containment (its folder
sits inside the lane folder), or both. Inline and containment must agree —
declaring `lane: news` from inside `pipelines/sports/` is rejected.

Inline-without-containment (naming a lane you don't live in) is tolerated
**only while that lane stays single-member**. The moment an apply would grow
the lane to two:

- every outside member must move into the lane folder first, and
- the lane's **composer must already be applied** — apply the composer
  before the lane's second member.

So the order of operations for a new multi-member lane is: create
`pipelines/<lane>/`, move the member folders in, apply the composer, then
apply the members.

## Folder surface (optional)

A composer that declares `reads`/`writes` sets a **folder surface** — a
ceiling on what its members may touch:

- Every member's declared `reads`/`writes` must be a **subset** of the
  surface, table and fields both. A table or field outside it fails apply.
- Declared writes are **exclusive among siblings**: two members of the folder
  may not both claim the same write table.
- A member that declares nothing **inherits**: reads inherit the surface
  whole; writes inherit the surface **minus tables a sibling already claims**.

No surface on the composer means members are unconstrained by it (their own
declarations still stand). Use a surface when the lane is one coherent
subject area and you want the blast radius written down once.

## Plugin lifetimes and lanes

`plugins: {alias: {ref: name@version, lifetime: run|lane|resident, fresh: true}}`

- `run` (default, and the only one executing today): a tool process per call.
- `lane`: one instance shared across a lane's serial walk.
- `resident`: one instance alive across runs.
- `fresh: true` opts out of warm attachment — a shared lane or resident
  instance is replaced before this pipeline's run rather than inherited.
  State carried across runs stays visible in lineage; `fresh` is how you
  refuse it.

Lane and resident lifetimes are parsed and modelled but not yet executed —
declaring one today is a forward statement of intent, not behavior.

## When to split a lane

Split when you want **isolation or parallelism**:

- A member that can hang or run long would otherwise hold everything behind
  it — its own lane keeps the rest moving.
- Two independent sites have no reason to serialize; one lane each lets them
  fetch concurrently.

Keep in one lane when you want **serialization**:

- One site's fetch → transform → publish chain: order is the point, and a
  shared lane means no two members of it ever contend.
- Members that must not run concurrently against the same external resource.

Remember the floor: splitting into single-member lanes does not buy
parallelism — those pipelines all land in the one shared queue lane. Real
parallelism means real lanes of two or more, or accepting the queue.
