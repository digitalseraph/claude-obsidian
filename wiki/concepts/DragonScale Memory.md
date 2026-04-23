---
type: concept
title: "DragonScale Memory"
complexity: advanced
domain: knowledge-management
aliases:
  - "DragonScale"
  - "DragonScale Architecture"
  - "Fractal Memory"
created: 2026-04-23
updated: 2026-04-23
tags:
  - concept
  - knowledge-management
  - memory
  - architecture
  - fractal
status: proposed
related:
  - "[[LLM Wiki Pattern]]"
  - "[[Compounding Knowledge]]"
  - "[[Hot Cache]]"
  - "[[concepts/_index]]"
sources:
---

# DragonScale Memory

A memory-layer design for LLM wiki vaults, inspired by the Heighway dragon curve. Four mechanisms (fold operator, content-addressable paths, semantic tiling, boundary-first autoresearch) give an LLM-maintained wiki a principled way to grow, compact, and stay coherent. The dragon curve is a design-justification device, not a reasoning architecture.

> **Status: proposed, v0.2 after adversarial review 2026-04-23.** Phase 0 only. No code yet. See Review History at the bottom for what was tightened between v0.1 and v0.2.

---

## Scope

DragonScale is a **memory architecture**: it governs how a wiki grows, compacts, addresses its pages, and checks for duplicates. It is **not a search, planning, or reasoning algorithm.** Agent reasoning uses existing patterns (Tree of Thoughts with BFS/DFS/beam search; Yao et al. 2023).

**Honest disclaimer**: memory-layer choices are never neutral with respect to reasoning. What the vault surfaces, and in what order, shapes what the model sees. Long-context performance is position-sensitive (Liu et al. 2023, *Lost in the Middle*), and MemGPT's premise is that paging policy affects task success (Packer et al. 2023). One of the four mechanisms below (boundary-first autoresearch) explicitly crosses into agenda control; it is included deliberately and marked as such.

---

## The Core Analogy

Four dragon-curve properties map onto memory-system patterns already validated in adjacent fields. The word is *analogue*, not *identity*.

| Dragon curve property | Memory analogue | Strength of analogy |
|---|---|---|
| Paper-folding recursion: `D_{n+1} = D_n · R · swap(reverse(D_n))` | Hierarchical rollup / materialized summary with exponential fanout | Loose. Shares exponential batch structure, not compaction semantics. |
| Turn derivable from bits of `n` (regular paperfolding sequence, OEIS A014577) | Content-addressable page addresses as organizational convention | Loose. Deterministic addressing is useful independent of the dragon. |
| Tiling / no self-intersection | Canonical-home coverage: one concept, one page | Medium. Dedup lint enforces this mechanically. |
| Boundary dim ≈ 1.523627 vs interior dim 2 | Agent attention weighted toward frontier pages | Aesthetic. The fractal dimension number does no load-bearing work. |

The curve is useful for deciding *which knobs to tighten and why*, not as a math proof that any given mechanism is optimal.

---

## Mechanism 1 — Fold Operator

After a batch of ingests, run a fold: produce a meta-page summarizing the batch, link children back, update the index. Folds stack: after enough level-`k` folds accumulate, a level-`k+1` fold produces a super-summary.

This is a **hierarchical rollup**, loosely similar to LSM-tree compaction but with important differences.

**What it shares with LSM compaction:**
- Exponential batch fanout across levels (like LevelDB's fixed level-size ratio, typically 10× per level in leveled mode)
- Periodic consolidation rather than per-write work

**What it does NOT inherit from LSM:**
- No sorted-key semantics (pages have semantic, not key-ordered, identity)
- No SSTable/memtable distinction, no tombstones, no Bloom filters
- No write-amplification arithmetic; no read-path acceleration
- **Folds are additive**: children remain in place. LSM compaction rewrites and deletes. A DragonScale fold is closer to a materialized view than a compaction.

**Trigger options:**
- `2^k` entry count (k=4 ⇒ every 16 log entries). Simple to implement; straightforward level math; ignores page size and novelty.
- **Adaptive trigger (preferred for production)**: token budget (e.g., fold when unfolded batch exceeds N tokens), novelty score (average embedding distance from existing summaries), or staleness age (last fold > T days). Phase 1 will implement entry-count for MVP; adaptive triggers are a follow-up.

**Invariants:**
- Idempotent on the same range (re-running is a no-op).
- Reversible (children stay; a fold is additive).
- Level-bounded: with entry-count trigger `2^k`, fold depth is at most `⌈log₂(N)⌉` above leaf pages. Derived, not empirical.

---

## Mechanism 2 — Content-Addressable Paths

Every page gets an `address` field in frontmatter encoding its fold-tree position plus a content hash:

```
address: 0b10110/a7f3c2
```

**What this gives:**
- Deterministic lineage: a page's fold ancestry is reconstructable from its address.
- Organizational convention: tooling and lint can treat pages by address without consulting content.

**What this does NOT give (corrected from v0.1):**
- **It does not directly improve Anthropic prompt cache hit rate.** Per Anthropic's docs, cache hits require byte-identical content up to the breakpoint. The address becomes part of the cached content only if the vault text actually includes it inside a cached block; even then, the hash suffix *rotates on content edits*, which would destabilize any prefix that relies on it. What actually helps cache hit rate: placing stable content (system prompts, tool definitions, pinned vault sections) before the breakpoint and keeping it byte-identical across requests. That principle is independent of path aesthetics.
- Address anchoring is kept in the spec as an organizational and lineage-tracking convention, not as a cache mechanism.

**Stability policy (open design question):** when a page's content changes, either (a) the bit-prefix stays and only the hash suffix rotates (preserves lineage, creates aliasing), or (b) the whole address rotates (simpler, loses lineage). Deferred to Phase 2 implementation.

---

## Mechanism 3 — Semantic Tiling Lint

The tiling property says the same concept should live in one canonical page. Enforce it with an embedding-based dedup check in `wiki-lint`.

**Procedure (calibrated, not a guess):**
1. Compute embeddings for every page. Default model: local `nomic-embed-text` via ollama (no API cost, local compute).
2. Compute pairwise cosine similarities for all page pairs.
3. **Calibration** (one-time, before first use): label 50-100 in-vault page pairs as duplicate/near/distinct; find the thresholds that optimize target precision for each band.
4. **Default bands** (used before calibration, then refined):
   - `≥ 0.90` — near-duplicate, lint error
   - `0.80 – 0.90` — review bucket, lint warning
   - `< 0.80` — distinct, no flag
5. Never auto-merge. Output a review list.

**Why not a fixed 0.85?** v0.1 used 0.85 with no justification. Published thresholds in the embeddings literature span a wide range (Sentence Transformers' `community_detection` defaults to 0.75; Quora-duplicate calibrations land around 0.77–0.83; sparse-model defaults differ again). Thresholds are model-, corpus-, and objective-dependent, so calibration is required.

---

## Mechanism 4 — Boundary-First Autoresearch

Boundary pages (high out-degree relative to in-degree, recency-weighted) are the vault's frontier. Extend `autoresearch` with a score:

```
boundary_score(page) = (out_degree - in_degree) * recency_weight
```

When `/autoresearch` is invoked without a topic, it reads the top-K boundary pages and generates prompts from them.

**Honest labeling**: this mechanism is **agenda control**, not pure memory. It shapes what the agent researches next. It is included in DragonScale because it is a direct consequence of the dragon-curve boundary analogy, and because it pairs naturally with folds (freshly folded pages have low out-degree; frontier pages are pre-fold). But the "memory only, not reasoning" framing does not cover it. Users who want a strict memory-layer subset should omit this mechanism.

---

## Operational Policies (required before implementation)

Adversarial review flagged these gaps in v0.1. Each must be decided before the corresponding phase ships.

| Policy | Phase 0 position | Decision point |
|---|---|---|
| **Retention / GC** | No automatic deletion. Pages are permanent. | Revisit if vault exceeds ~5000 pages. |
| **Tombstones** | None. Deleted pages are removed via git revert. | Revisit if delete events become common. |
| **Versioning** | Relied on git history, not in-vault versioning. | Address-hash rotation policy doubles as a coarse version signal. |
| **Conflict resolution for contradictory folds** | Meta-page must quote both sources with explicit "conflict" callout. No automatic resolution. | Phase 1 spec required. |
| **Concurrency / atomicity** | Single-writer assumption (one Claude session at a time). PostToolUse auto-commit serializes. | Multi-writer case deferred. |
| **Provenance for meta-pages** | Every fold page must include frontmatter listing children and fold level. | Phase 1 must enforce. |
| **Access control** | Out of scope. This is a single-user vault. | Revisit only if shared. |

---

## Mapping to Claude-Obsidian

| Mechanism | New | Extends |
|---|---|---|
| Fold operator | new skill `wiki-fold` | reads `log.md`, writes `wiki/folds/`, updates `index.md` |
| Address anchors | new frontmatter field | assigned on create by `wiki-ingest`; validated by `wiki-lint` |
| Semantic tiling | new lint check | `wiki-lint` with calibrated bands |
| Boundary-first | new scoring in `autoresearch` | optional mechanism, agenda-control flagged |

The existing hot → index → domain → page hierarchy already implements self-similarity across scales. That's the one dragon-curve property this vault had before DragonScale.

---

## Why This Over Alternatives

| Pattern | What it gives | What DragonScale adds |
|---|---|---|
| MemGPT virtual context (two-tier paging) | Main context ↔ external context swap | More than two levels; explicit fold triggers; dedup lint |
| Pure LSM compaction | Exponential write-path throughput | Semantic-layer mechanisms (tiling, boundary); additive rollups over destructive merges |
| Ad-hoc `/save` | Human-triggered filing | Rule-based fold cadence |
| Vector-only RAG | Retrieval | Canonical-home structure; lineage addresses |

DragonScale composes patterns validated in adjacent systems: LSM *batching* (databases), MemGPT *paging* (agents), Anthropic *cache ordering* (prompt engineering), and embedding *dedup* (knowledge graphs).

---

## Known Limitations (v0.2)

- **Unvalidated at scale.** All four mechanisms are theoretical; none tested on a multi-thousand-page vault.
- **Fold cadence is a knob, not a theorem.** `k=4` is a starting guess. Adaptive triggers are likely better.
- **Address stability is unsolved.** Hash rotation on edits is a known issue; deferred.
- **Boundary-first crosses scope.** Included with a warning, not quietly.
- **Calibration load.** Tiling requires a one-time labeling pass; without it, only defaults apply.

---

## Primary Sources

Verified against primary sources on 2026-04-23. Claims are tagged **[sourced]** when directly citable, **[derived]** when derivable from sourced material, or **[conjecture]** when based on reasoning without a specific source.

**Dragon curve math [sourced]**
- Boundary dimension `2·log₂(λ)` where `λ³ − λ² − 2 = 0`, giving 1.523627086: [Dragon curve, Wikipedia](https://en.wikipedia.org/wiki/Dragon_curve)
- Paper-folding construction and OEIS A014577: [Regular paperfolding sequence, Wikipedia](https://en.wikipedia.org/wiki/Regular_paperfolding_sequence); [OEIS A014577](https://oeis.org/A014577)
- Tiling and rep-tiles: [Wolfram Demonstrations: Tiling Dragons and Rep-tiles of Order Two](https://demonstrations.wolfram.com/TilingDragonsAndRepTilesOfOrderTwo/)

**LSM trees [sourced]**
- Level size ratios and compaction semantics: [RocksDB Compaction wiki](https://github.com/facebook/rocksdb/wiki/Compaction), [RocksDB Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide), [How to Grow an LSM-tree? (2025)](https://arxiv.org/abs/2504.17178)
- LevelDB 10× level ratio: referenced in the arXiv paper above. Treat as *typical*, not required.

**LLM memory architectures [sourced]**
- OS-inspired paging: [MemGPT: Towards LLMs as Operating Systems (Packer et al. 2023)](https://arxiv.org/abs/2310.08560)
- Position sensitivity: [Lost in the Middle (Liu et al. 2023)](https://direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00638/119630/Lost-in-the-Middle-How-Language-Models-Use-Long)
- Note-based agentic memory: [A-Mem (2025)](https://arxiv.org/abs/2502.12110)

**Prompt caching [sourced]**
- Byte-identical prefix requirement, breakpoint mechanics, TTL options: [Anthropic Prompt Caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

**Embedding thresholds [sourced]**
- Sentence Transformers defaults and calibration examples: [Sentence Transformers util](https://sbert.net/docs/package_reference/util.html), [SBERT evaluation docs](https://sbert.net/docs/package_reference/sentence_transformer/evaluation.html)

**Reasoning search (out of scope, cited only to justify the scope boundary) [sourced]**
- [Tree of Thoughts (Yao et al. 2023)](https://arxiv.org/abs/2305.10601)

**Items marked [conjecture] in this doc:**
- `k=4`/`k=5` starting value for fold cadence (needs empirical tuning)
- `~30s` full-vault embedding-pass time (needs measurement)
- `boundary_score` formula exact weighting (a plausible starting form; not validated against retrieval metrics)

**Items marked [derived]:**
- `⌈log₂(N)⌉` fold-depth bound (trivially derivable from the entry-count trigger)
- Default tiling bands `{≥0.90, 0.80-0.90, <0.80}` before calibration (interpolated from cited ranges in Sentence Transformers examples; not optimal by construction)

---

## Review History

**v0.1 (2026-04-23, initial draft)** — written after a verification pass against Wikipedia, arXiv, and Anthropic docs. Four mechanisms proposed.

**v0.2 (2026-04-23, post-adversarial review)** — after `codex exec` adversarial review. All 7 critiques accepted:

1. *LSM "structurally identical"* → weakened to "loosely analogous to hierarchical rollup"; non-inherited properties listed explicitly.
2. *Prompt cache address benefit* → removed strong claim; narrowed to organizational convention.
3. *0.85 threshold* → replaced with calibration procedure and banded defaults.
4. *2^k cadence* → justified as implementation convenience; adaptive trigger flagged as preferred for production.
5. *Scope boundary contradiction* → acknowledged; boundary-first explicitly labeled as agenda control.
6. *Missing production mechanisms* → added Operational Policies section (retention, versioning, conflict resolution, concurrency, provenance).
7. *Unverified claims* → tagged every claim as [sourced], [derived], or [conjecture].

---

## Connections

See [[LLM Wiki Pattern]] for the broader pattern this extends.
See [[Compounding Knowledge]] for why persistent state is the precondition for DragonScale.
See [[Hot Cache]] for the existing 500-word session context, which is a level-0 manual fold.
See [[Andrej Karpathy]] for the intellectual lineage.
