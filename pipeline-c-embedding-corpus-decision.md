# Pipeline C — Embedding Topology Decision: One Corpus, Filter, Then Cluster

> **Status:** Decided
> **Scope:** Pipeline C (unified corpus) — ingestion / embedding stage and the
> clustering-topology question it touches.
> **Suggested placement:** next free ADR number (ADRs 0005–0010 exist) or
> `docs/pipeline-c-embedding-strategy.md`.
> **Constants referenced are canonical as of this record** — see Appendix A.

---

## 0. TL;DR

> **DECISION**
>
> Embed the **complete day's corpus as a single file** (one canonical vector per
> post, persisted to pgvector), **then** apply platform (FB / T) and locale
> (PS / INT / IL) **filters**, **then** run HDBSCAN clustering on the filtered
> slices.
>
> This was chosen over splitting the source into per-(platform × locale) files
> *before* embedding. The two paths produce **identical embedding values and
> identical clusters when fed the same slices** — so the choice was made on
> operational and architectural grounds, not on embedding quality.

---

## 1. Context — the question we resolved

Two candidate ingestion shapes were on the table:

- **Scenario 1 — Embed unified, filter, then cluster.**
  One complete `posts.csv` → embed the whole corpus → filter by platform/locale
  → cluster each slice.

- **Scenario 2 — Split first, then embed.**
  Source pre-split into `posts_PS_FB.csv`, `posts_PS_T.csv`, `posts_IL_FB.csv`,
  … → embed each split file separately → cluster each.

The load-bearing technical question underneath both was:

> *Is the embedding of a post dependent on the other posts in its file, or
> independent per post? Would a post's vector change between a large corpus file
> and a split file?*

---

## 2. Core mechanical finding — embedding is per-post independent

> **VERDICT — Embedding independence**
>
> The embedding of a post is a **pure function of that post's own text** (plus
> the model weights and requested dimensionality). It takes **no input** from the
> other rows in the file. File composition has **zero causal influence** on the
> number. Splitting vs. unifying does not change a post's vector.

Why this holds, mechanically:

- `text-embedding-3-small` encodes each input string on its own — one post in,
  one 1536-dim vector out.
- There is **no corpus-level statistic** in the encoding: no IDF, no
  document-frequency weighting, no batch-level normalization that mixes one post
  into another.
- A transformer embedding model uses **within-sequence attention only**. Post A's
  tokens never attend to Post B's tokens, even when A and B ride in the same
  batch of a single API call. Batching is a throughput convenience server-side,
  not a semantic operation.
- The pipeline's **L2 normalization is per-vector** (each vector divided by its
  own norm), so the normalization step reintroduces no corpus dependence either.

### 2.1 The only source of non-identity (and why it is irrelevant here)

> **VERDICT — Non-determinism is call-level, not corpus-level**
>
> Bit-for-bit equality across two separate API calls is **not contractually
> guaranteed**. The same string sent twice can differ at roughly the **1e-6–1e-7
> level** per component (frequently identical). The cause is backend
> floating-point behavior — GPU reduction order, server-side batching/padding,
> mixed precision — and it is **call-to-call variance, not file-composition
> variance**. You would see the same jitter embedding the same lone post twice in
> two runs of the *same* scenario.

Practical consequence for us: any difference observed between "separate file" and
"single corpus file" is attributable to it being a *second API call*, not to the
corpus around the post. And because we **embed once and persist to pgvector**
(`model_id` / `model_version` stamped), the canonical vector is reused
downstream — the jitter never re-enters. Even where re-embedding a duplicate
would occur, the delta is **orders of magnitude below** any clustering threshold
and cannot move a cluster boundary.

---

## 3. Scenario comparison

| Dimension | Scenario 1 — embed unified, filter, cluster | Scenario 2 — split, then embed |
|---|---|---|
| Embedding value per post | Identical (per-post independent) | Identical (per-post independent) |
| Clusters on the same slice | Identical (HDBSCAN deterministic on fixed points/params) | Identical |
| **Embedding quality** | **No difference** | **No difference** |
| Embeds per post | Exactly once (canonical) | Once *per file the post lands in* — duplicates embed N times |
| Token cost | Baseline | Same, **unless** a post appears in multiple split files → wasted spend |
| Vector consistency / provenance | One canonical vector, one run, one provenance | Risk of N near-identical vectors for one post across files |
| pgvector mapping | Clean: one table, `WHERE platform=… AND locale=…` | Risk of siloed / redundant storage |
| Corpus-wide pre-clustering ops (e.g. §5.1 template detector) | **Available** — runs on full corpus before the split | **Foreclosed** at ingestion |
| Cross-lens work (`unclassified` audit) | **Available** | **Foreclosed** |
| Clustering-granularity flexibility | Choose at clustering time (incl. not splitting) | Hard-coded at ingestion |
| Auditability | Higher (single embed run, single model-identity stamp) | Lower (multiple runs, drift surface) |

---

## 4. Why we chose Scenario 1

Because embedding quality is identical, the decision rests entirely on the
right-hand consequences above:

1. **Embed-once is cost-safe and consistent.** One canonical vector per post; no
   duplicate-embedding spend; no two-slightly-different-vectors-for-one-post
   problem.
2. **It maps cleanly onto pgvector** and onto the existing
   `model_id`/`model_version` provenance stamping.
3. **It preserves optionality.** Splitting *before* embedding bakes the
   segmentation into ingestion and forecloses anything that needs the whole
   corpus. Embedding once keeps the door open to:
   - the **§5.1 template detector** running corpus-wide *before* the
     platform/locale filter, and
   - the **`unclassified` cross-lens** diagnostic.
4. **It is more auditable** — one embed run, one model-identity stamp — which fits
   the audit-first / verify-model-identity discipline.

> **VERDICT — The embed-where question carries no quality cost**
>
> Picking Scenario 1 over Scenario 2 wins on cost-safety, dedup, provenance,
> pgvector fit, and optionality, and loses **nothing** on embedding or clustering
> quality.

---

## 5. The decision splits into two independent questions — keep them separate

This is the most important framing to preserve in the docsite, because it is easy
to conflate.

| # | Question | What we decided | Quality consequence |
|---|---|---|---|
| Q1 | **Where do we embed?** | Embed the **complete corpus once**, then filter | **None** — vectors identical either way |
| Q2 | **What do we cluster — segmented slices or the unified corpus?** | Filter by platform **and** locale, then cluster each slice | **Real** — see §6 |

Q1 is settled with no trade-offs. Q2 is the actual quality lever and is
**orthogonal** to Q1. The chosen path answers Q2 with *segmented clustering*
(per platform × locale). That is a legitimate choice, but it is **Pipeline B's
segmentation philosophy applied on new axes**, not Pipeline C's native
unified-corpus clustering — and it brings the quality implications in §6.

---

## 6. Quality implications of the chosen clustering topology

Filtering by platform **and** locale before HDBSCAN produces **smaller point sets
per slice**, which interacts with the **absolute** clustering floors
(`min_cluster_size=35`, `min_samples=10`). Watch for:

- **Sub-floor micro-story dropout.** A topic that clears the density floor in the
  unified corpus can fall below it once sliced into a thin PS/FB-only subset, and
  vanish as noise.
- **Structural splitting of cross-platform running stories.** A story that spans
  FB and T is split at the platform boundary *before clustering ever sees it*,
  rather than being fused and then analyzed.
- **Reportability shifts.** A cluster that would be reportable in the unified
  corpus may not reach reportable shape inside a single thin slice.

> **VERDICT — Embedding cost: zero; clustering topology: a real lever**
>
> The chosen path imposes **no embedding-quality penalty**. Any quality movement
> comes from the *decision to segment before clustering*, not from embedding once.
> This lever should be validated on real data before it is treated as settled
> (see §8).

---

## 7. Things to always consider / watch list

- **Embedding is deterministic for every practical purpose.** Treat a post's
  vector as a fixed function of its text. Do **not** attribute any clustering
  change to "which file it was in."
- **Sub-noise jitter is call-level, never corpus-level.** If two vectors for the
  "same" post differ, the cause is a second API call, not the corpus.
- **One canonical vector per post.** Guard against the same post being embedded in
  more than one file/run; dedup before/at embedding, not after.
- **Absolute floors bite harder on thin slices.** Any change to slice granularity
  (adding an axis, narrowing locale) shrinks point sets and interacts with
  `min_cluster_size` / `min_samples`. Re-check floors when granularity changes.
- **Cross-platform / cross-locale stories.** Segmented clustering cannot fuse
  across a boundary it split on. Decide deliberately which boundaries are
  analytically meaningful to split on.
- **Preserve corpus-wide pre-clustering hooks.** Run the §5.1 template detector
  (and any future corpus-global filter) on the full corpus *before* the
  platform/locale filter, while the unified vector set still exists.
- **Model identity is canonical.** `text-embedding-3-small`, not BGE-M3. Verify
  model identity before any cost or architecture decision; stamp
  `model_id`/`model_version` on persisted vectors.
- **No UMAP in production.** HDBSCAN runs directly on L2-normalized 1536-dim
  vectors; UMAP exists only in the offline evaluation harness.

---

## 8. Open / follow-up items

- **Validate Q2 (segmentation lever) on evidence, not intuition.** Recommended
  read-only diagnostic: embed one day once, then cluster the **same day three
  ways** — fully unified / locale-only split / locale × platform split — and
  compare on cluster count, noise %, and known failure modes (platform-split
  running stories, sub-floor micro-stories). Decide the topology on the result.
- **Confirm template-detector placement.** If §5.1 ships, fix its position as
  corpus-wide, *before* the platform/locale filter.
- **No change required to the embedding stage itself** beyond the standing
  embed-once + persist-to-pgvector behavior already in place.

---

## Appendix A — Canonical constants referenced

- **Embedding model:** `text-embedding-3-small` (OpenAI), 1536-dim. *Not* BGE-M3.
- **Vector handling:** L2-normalized per-vector; HDBSCAN direct (no UMAP in
  production).
- **HDBSCAN (healthy-day targets):** `min_cluster_size=35`, `min_samples=10`,
  `cluster_selection_epsilon=0.0`. Production env-var pins may differ.
- **LLM:** `gpt-4.1-mini`.
- **Storage:** pgvector / PostgreSQL; vectors stamped with
  `model_id` / `model_version`.
- **Domains:** seven canonical domains + `unclassified` sentinel (cross-lens,
  first-class).
- **Terminology:** Pipeline B = segmented legacy; Pipeline C = unified corpus.

---

*This record is a faithful summary of the working session that produced the
decision. The embedding-independence finding is a property of the model and the
endpoint, not a measurement taken during the session; the ~1e-6–1e-7 jitter
figure describes known endpoint behavior, not a value benchmarked here.*
