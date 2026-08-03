# Newsletter Pipeline — Specification

**Status:** design, pre-implementation
**Cadence:** weekly
**Scale:** 20–50 hand-curated sources
**Editor:** single human (owner), two approval gates

---

## 1. Design thesis

The pipeline inverts the conventional agent shape: **fan out for I/O, fan in for judgment.**

The common pattern parallelizes per-item "understanding" — one model call per candidate
story. That is structurally wrong for editorial work. "Is this the most interesting thing
this week" is a *comparative* judgment, and a worker looking at one item in isolation
cannot make it. Per-item extraction produces a pile of items each rated 7/10.

So: a cheap, model-free wide funnel; a single comparative judgment over the whole
candidate set; expensive deep enrichment on the survivors only.

Three further commitments:

1. **No LLM in the ingest path.** Ingest must be boring, cheap, and deterministic.
   Understanding is deferred to the one stage that needs it.
2. **The human gate is the product.** Source curation and story selection are the moat.
   Neither is automated.
3. **Lint, don't judge.** Models are reliable at set-membership and mechanical checks,
   unreliable at "is this good." Only the former is automated.

---

## 2. Pipeline

```
sources.yaml ──┐                                    [hand-curated: the moat]
               ▼
  1. INGEST          poll → normalize → hash-dedup        no LLM
               ▼
  2. COVERAGE FILTER embed vs. last N issues              no LLM
               ▼
  3. TRIAGE          ALL survivors in ONE call → ranked   fan IN
               ▼
  4. ENRICH          top ~10 only, parallel subagents     fan OUT
               ▼
  5. ◆ GATE A ◆      approve / reorder / cut / steer      blocking
               ▼
  6. DRAFT           whole issue, one shot, voice pack
               ▼
  7. LINT            mechanical + containment             ⟲ max 2 retries
               ▼
  8. ◆ GATE B ◆      diff-shaped review → send            blocking
               ▼
  9. LEARN           CTR/replies/unsubs → priors into (3)
                     ↑
        memory: voice pack · coverage.db · entity graph
```

### Stage detail

**1. Ingest.** Poll every active source. Normalize to `RawItem`. Canonicalize URLs
(strip tracking params, resolve redirects, normalize scheme/host case). Content-hash
dedup. Zero model calls. Source fetch failures are logged and non-fatal — one dead feed
must never block a run.

**2. Coverage filter.** Embed `title + lede`. Compare against `shipped_stories`
embeddings from the last N issues (N=8 default) and the entity set. Drop exact and
near-duplicates. Deterministic, testable, cheap.

**3. Triage.** One long-context call over the entire surviving candidate set. At this
cadence and source count the set is order-of-hundreds and fits comfortably — **no
sharding, no pre-ranking.** Returns ranked clusters with a one-line `why` and a
confidence. The rubric and performance priors are prompt-cached (identical every run).

**4. Enrich.** Parallel read-only subagents across the top ~10 clusters only. Each:
read the primary source, verify the central claim, extract a usable quote, find the
strongest counter-take. Expensive and deep, but on 10 items rather than 400.

**5. Gate A — selection.** The high-leverage gate. Ranked cards showing headline, why,
source trust, and memory's callback suggestions. **The cut list is visible below the
fold** — promoting from it is the highest-value action the editor takes and the primary
calibration signal for triage. Editor reorders, cuts, promotes, and may attach one-line
steers ("lead skeptical on this"). Target: under 5 minutes.

**6. Draft.** The entire issue in one call. Pacing and transitions only work if the model
sees the whole arc; section-by-section generation is why most automated newsletters read
like a listicle. Input: approved stories + enrichment + steers + voice pack + callbacks.

**7. Lint.** See §5. Violations loop back to stage 6 with specifics, max 2 retries, then
escalate to the editor.

**8. Gate B — final read.** Built **diff-shaped, not read-shaped**: lint results,
callbacks used, and anything changed since Gate A, plus one send button. A full re-read
every week becomes a chore and chores get abandoned. Skippable per-issue once the voice
pack is trusted. Send remains a discrete, explicitly authorized action — never automatic.

**9. Learn.** Per-story metrics written back, feeding ranking priors into stage 3.
Monthly source audit: which feeds produced zero selected items over the last 8 issues.

---

## 3. Data model

Single SQLite file with vector search. No vector database at this volume.

```
sources          id, url, kind, beat, trust, active, added_at
raw_items        id, source_id, url_canonical, content_hash, title,
                 published_at, fetched_at, body, embedding
candidates       run_id, raw_item_id, cluster_id, rank, why, confidence
issues           id, number, subject, sent_at
shipped_stories  issue_id, slot, raw_item_id, headline, embedding
entities         id, name, kind
claims           id, entity_id, issue_id, claim, stance
metrics          issue_id, slot, opens, clicks, replies, unsubs
```

**Scoping decision:** `entities` and `claims` populate from `shipped_stories` only —
never from the raw firehose. This is what keeps the callback graph tractable instead of
a boil-the-ocean knowledge graph. Its sole job is enabling *"three weeks ago this same
person said the opposite"*, which is the thing that makes a newsletter feel written by
someone who has been paying attention.

---

## 4. Memory layer

| Artifact | Contents | Maintenance |
|---|---|---|
| `voice/exemplars.md` | 5–10 paragraphs of the editor's own best writing | Hand-curated, versioned |
| `voice/banned.md` | Banned phrases, tics, structural anti-patterns | Hand-curated, grows over time |
| `voice/template.md` | Issue structure: sections, lengths, sign-off | Hand-curated, rarely changes |
| `coverage.db` | Everything ever shipped, embedded | Automatic |
| entity/claim graph | Callback fuel | Automatic, shipped-only |

The voice pack is prompt-cached — byte-identical across runs. It is the single thing
preventing issue #40 from sounding like generic model output, and it is **pure human
work that must exist before any code is written.**

---

## 5. Lint rules

Split strictly by what models are reliable at.

**Mechanical (deterministic, no model):**
- Every factual claim maps to a source URL present in the enrichment record
- Every link resolves 200 (with a redirect allowance)
- No banned phrases from `voice/banned.md`
- Structural conformance to `voice/template.md` (section count, length bounds)

**Containment (narrow model check, high precision):**
- Does the draft assert anything not present in the enrichment payload?

This is a set-membership question, not a taste question, which is why it is trustworthy.
It is the only thing standing between the editor and a confidently fabricated quote going
out over their name.

**Explicitly not automated:** "is this interesting", "is this well written", "is this the
right lead." LLM-as-judge is unreliable on exactly these, and behavioral signal (§7)
answers them better for free.

---

## 6. Pure function signatures

The TDD surface. Coverage concentrates here.

```ts
canonicalizeUrl(raw: string): string
contentHash(title: string, body: string): string
isDuplicate(item: RawItem, history: ShippedStory[], threshold: number): DupeVerdict
clusterCandidates(items: RawItem[], threshold: number): Cluster[]
rankFinal(triage: TriageRank[], priors: SlotPerformance[]): OrderedStory[]
lintDraft(draft: Draft, enrichment: Enrichment): Violation[]
```

### Fixture requirement

**Mandatory, and this is the highest-risk area in the system.** RSS in the wild is a
swamp: RSS 2.0 vs Atom, multiple date formats, CDATA-wrapped HTML, relative URLs, feeds
that put the full body in `<description>` and feeds that put a teaser.

Capture real responses from the actual source list into `tests/fixtures/feeds/` and
snapshot-test normalization through the production path. Synthetic feed XML will pass
while production silently drops a third of the sources.

---

## 7. Observability

### Startup banner

Printed at run start: run ID, config path, source count (active/total), cadence,
last issue number and date, coverage window (N issues), model IDs for triage / enrich /
draft / lint, voice pack version hash, dry-run flag.

### Per-run metrics

| Field | Type | Description |
|---|---|---|
| `run_id` | string | ULID, one per pipeline run |
| `sources_polled` | int | Sources attempted |
| `sources_failed` | int | Fetch failures (non-fatal) |
| `raw_items` | int | Items retrieved pre-dedup |
| `dupes_hash` | int | Dropped by content hash |
| `dupes_semantic` | int | Dropped by coverage filter |
| `candidates` | int | Items entering triage |
| `triage_ms` | int | Duration of the single triage call |
| `triage_tokens` | int | Tokens consumed by triage |
| `clusters` | int | Clusters returned |
| `enriched` | int | Clusters enriched |
| `enrich_failed` | int | Enrichment subagent failures |
| `gate_a_wait_ms` | int | Editor time at Gate A |
| `promoted_from_cut` | int | **Primary triage calibration signal** |
| `draft_tokens` | int | Tokens consumed by drafting |
| `lint_violations` | int | Violations found |
| `lint_retries` | int | Redraft cycles (max 2) |
| `gate_b_wait_ms` | int | Editor time at Gate B |
| `sent` | bool | Whether the issue shipped |

`promoted_from_cut` is the number to watch. Regular promotions mean triage ranking is
miscalibrated — fixable via the rubric. Zero for eight consecutive issues means triage is
trustworthy and Gate A can be shortened.

### Per-story detail

| Field | Type | Description |
|---|---|---|
| `issue` | int | Issue number |
| `slot` | int | Final position in the issue |
| `source_id` | string | Originating source |
| `cluster_size` | int | Items merged into this story |
| `triage_rank` | int | Rank assigned by triage |
| `final_rank` | int | Rank after editor action |
| `editor_action` | enum | `kept` / `moved` / `cut` / `promoted` |
| `steer` | string? | Editor's one-line direction, if any |
| `callbacks_used` | int | Prior-issue references included |

### Stage-progress events

This is a minutes-long batch run, not a long-lived daemon, so a 60s periodic heartbeat
does not apply. The equivalent is a `stage` event emitted on entry and exit of each of
the nine stages, carrying `{run_id, stage, status, elapsed_ms}` — enough to see where a
run is or where it died.

### JSONL format

All events: `{ts, event, run_id, ...fields}`

| `event` | Emitted when | Key fields |
|---|---|---|
| `run_start` | Pipeline begins | config, source counts, model IDs, voice hash |
| `stage` | Stage entry/exit | stage, status, elapsed_ms |
| `source_fetch` | Per source | source_id, status, items, error? |
| `triage_done` | Triage returns | candidates, clusters, ms, tokens |
| `enrich_done` | Per cluster | cluster_id, status, quote_found, counter_found |
| `gate_a` | Editor submits | wait_ms, kept, cut, promoted_from_cut, steers |
| `lint_result` | Each lint pass | pass, violations[], retry_index |
| `gate_b` | Editor submits | wait_ms, skipped, sent |
| `issue_sent` | Send completes | issue, subject, story_count, recipients |
| `metrics_sync` | Writeback | issue, opens, clicks, replies, unsubs |
| `run_end` | Pipeline ends | status, total_ms, sent |

---

## 8. Build order (TDD: RED → GREEN → wire → verify)

| # | Step | Notes |
|---|---|---|
| 1 | **Voice pack** | No code. Exemplars, banned list, template. Everything downstream is worthless without it. |
| 2 | **Walking skeleton** | `sources.yaml` → ingest → dedup → triage → ranked JSON to file. Ship issue #1 by hand from that output. |
| 3 | Gate A review page | Reads the run JSON. Static app, not generated per-run. |
| 4 | Enrich subagents | Parallel, read-only, top-10 only. |
| 5 | Draft + lint loop | Voice pack wired in; containment check is the priority rule. |
| 6 | Gate B, render, send | Diff-shaped. Fixed email template. |
| 7 | Metrics writeback | Close the loop into triage priors. |

Steps 1–2 determine whether the idea works at all — they test the source list, which is
the actual moat. Everything after step 2 is machinery. Do not build UI before issue #1
has shipped by hand.

---

## 9. Known risks

| Risk | Mitigation |
|---|---|
| **Gate A becomes a chore** | Primary failure mode. Target <5 min; track `gate_a_wait_ms` and treat regression as a bug. |
| Voice drift over months | Voice pack + banned list + tone lint; refresh exemplars from best recent issues. |
| Source list goes stale | Monthly audit on zero-selection sources over 8 issues. |
| Hallucinated quotes | Containment lint; enrichment payload is the only permitted claim source. |
| Feed parsing silently drops sources | Real-response fixtures; alert when `sources_failed > 0`. |
| Triage miscalibration | `promoted_from_cut` tracked per issue; rubric is tunable. |

---

## 10. Open questions

- Email delivery provider (determines the metrics writeback integration in §7)
- Whether Gate A and Gate B share one review app or are separate surfaces
- Coverage window `N` — 8 issues is a starting guess, tune once history exists
- Embedding model choice for the coverage filter
