# newsletter

An agent pipeline for generating a weekly newsletter, where the human keeps the two jobs
that actually matter: **curating sources** and **selecting stories**.

## Design thesis

Fan out for I/O, fan in for judgment.

The conventional agent shape parallelizes per-item "understanding" — one model call per
candidate story. That is structurally wrong for editorial work: *"is this the most
interesting thing this week"* is a comparative judgment, and a worker looking at one item
in isolation cannot make it. You get a pile of items each rated 7/10.

Instead: a cheap, model-free wide funnel → a single comparative judgment over the whole
candidate set → expensive deep enrichment on the survivors only.

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

## Commitments

- **No LLM in the ingest path.** Ingest is boring, cheap, deterministic. Understanding is
  deferred to the one stage that needs it.
- **The human gate is the product.** Source curation and story selection are never
  automated.
- **Lint, don't judge.** Models are reliable at set-membership and mechanical checks,
  unreliable at "is this good." Only the former is automated; behavioral signal answers
  the rest for free.
- **The callback graph is built from shipped issues only**, never the raw firehose. Its
  one job is *"three weeks ago this same person said the opposite"* — the thing that makes
  a newsletter feel written by someone who's been paying attention.

## Status

Pre-implementation. The design lives in [`docs/SPEC.md`](docs/SPEC.md) — data model,
pure-function surface, lint rules, observability, build order, and known risks.

**Step 1 is not code.** The voice pack (5–10 paragraphs of your own best writing, a
banned-phrase list, a structure template) has to exist before anything downstream is
worth building.

## Parameters

| | |
|---|---|
| Cadence | Weekly |
| Sources | 20–50, hand-curated |
| Editor | Single human, two approval gates |
| Store | One SQLite file with vector search |
