# 04: Indexing: Offline Build and Runtime Retrieval

## Principle: structured first, lexical second, semantic last

Vector search is the least repeatable retriever. Small changes in phrasing shift the top-k. So the order is:

1. **Structured (manifest/facet) lookup.** Exact and deterministic. It answers "which rules apply to an `llm_application` with `tool_use`?"
2. **Lexical (BM25).** Deterministic given the same index. Best for IDs, CVEs, product names, and config keys such as `s3:PutBucketPolicy`.
3. **Semantic (vector).** Use it for fuzzy recall such as related precedent and long-report passages. Its results are *suggestions* the pipeline confirms, not decisions.
4. **Graph edges.** Rule → related rules → rulings → controls → compensating controls. Deterministic expansion of what the first three found.

## Offline: the knowledge snapshot

Built in CI on every merge to `knowledge/`.

```
snapshot/<sha256>/
├── manifest.json            # every object: id, type, version, content_hash, path, facets, status, dates
├── facets.sqlite            # (object_id, facet, value) for structured filtering
├── lexical/                 # BM25 index over title + body + glossary synonyms
├── vectors/                 # embeddings (record embedding model + version in manifest!)
├── graph.json               # typed edges: applies_to, related, overrides, supersedes, maps_to_control
├── reference/               # tables converted to sqlite/parquet for lookup tool
└── build-report.json        # validation results, contradiction check, coverage stats
```

**Build steps**

1. **Validate.** Every file passes its schema, every ID is unique, and every reference (`related_rules`, `rules`, `controls`, `supersedes`) resolves.
2. **Lint.** `review_by` must not be in the past for `active` objects. No `active` ruling may reference a `deprecated` rule.
3. **Contradiction check.** No two `active` rulings may have the same structured `fact_pattern` on the same rule with different decisions. Flag near-duplicates by embedding similarity for human review.
4. **Chunk.** Rules and rulings are **one chunk each**, because they were sized for it in doc 03. Context docs are split by heading. Reference tables are not chunked; they become lookup tables.
5. **Embed.** Embed only `title + intent + fact_pattern_text`, not the whole body. That keeps vectors about *what situation this is*, not boilerplate.
6. **Hash and publish.** The snapshot ID is the hash of its contents. It is immutable, and runs pin it.

**Embedding model changes** are a full re-index plus a full eval run, treated like a major version bump.

## Evidence-side indexing (per run or cached)

Evidence is usually small enough to pass directly: an IAM policy, a tool manifest, scanner JSON. Only index when it isn't.

| Evidence kind | Strategy |
|---|---|
| Structured API/MCP responses | **No indexing.** Normalize into an `EvidenceRecord` and pass the relevant fields. Keep raw output in the run cache. |
| Long reports (pen test, SOC 2, design docs) | Chunk by heading path, **300 to 800 tokens**, about 10% overlap. Store `doc_id, heading_path, page/offset, hash`. Use a per-run BM25 + vector index. |
| Large codebases | Don't embed everything. Use symbol and grep tooling (MCP code-search) driven by the rule's `evidence_required`. Embed only for "find where X happens" fallback. |
| Warehouse/reference tables | A query tool with a fixed, parameterized query catalog. No free-form SQL from the LLM. |

Cache evidence per **(source, query, subject, TTL)** so a re-run within the TTL sees identical evidence. That cache is what makes "same input → same output" achievable when the live sources keep moving.

## Runtime: how the running agent retrieves during a review

```
1. PIN        snapshot_sha, model_id, prompt_versions, retrieval params → run manifest
2. ROUTE      subject → facets
              • T0 sources first (CMDB says asset_type=llm_application)
              • LLM fills gaps from a CLOSED vocab (facets.yaml), with evidence citations
3. SELECT     facets → playbook → candidate rules
              • SQL over facets.sqlite: applies_when ∧ ¬not_applicable_when ∧ status=active
              • graph expand: related_rules (1 hop, flagged "related", not mandatory)
              • ORDER BY category, rule_id   ← stable order matters
4. PLAN       checklist = selected rules × their evidence_required
5. PER CHECK
   a. fetch evidence for that rule only (cached)
   b. precedent lookup:
        exact:    rulings WHERE rules∋rule_id AND fact_pattern ⊆ subject facets   → APPLY
        semantic: top-k=3 by vector on fact_pattern_text, threshold τ             → SURFACE
        tie-break: score desc, decided_on desc, id asc                            ← deterministic
   c. assemble context in a FIXED ORDER:
        [system prompt] [static org context] [rule body] [applied rulings]
        [surfaced rulings] [evidence, delimited + trust-tagged] [output schema]
   d. judge → finding
6. LOG        every retrieved object id + hash + score into the run manifest
```

**Why per-check retrieval instead of one big context:** it keeps every judgment's context small and focused (a few thousand tokens), makes each finding independently replayable, parallelizes cleanly, and stops one noisy document from contaminating unrelated checks.

### Determinism knobs, all recorded in the manifest

- Snapshot SHA, embedding model and version
- `k`, similarity threshold `τ`, BM25 parameters
- Tie-break order
- Context assembly order and truncation policy (truncate evidence, **never** truncate rules)
- Model ID, temperature, max tokens, prompt version

## Open questions

- Vector store: is managed (pgvector, OpenSearch, a hosted vector DB) acceptable, or does it need to be embedded? At a few thousand knowledge objects, **SQLite + FTS5 + a flat vector file** is enough, ships inside the snapshot, and is trivially reproducible. Don't over-engineer this early.
- How long can evidence be cached and still count as "current" for a review? That's a policy question per source.
- Do reviews ever need cross-subject retrieval, like "have we reviewed this vendor before"? If yes, past *run outputs* need their own index, separate from knowledge.
