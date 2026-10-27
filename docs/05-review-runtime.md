# 05: Review Runtime: Repeatable Output

## Definition of "repeatable"

Be explicit, because there are levels:

| Level | Meaning | Achievable? |
|---|---|---|
| **R1 byte-identical** | Same report text | Only with frozen evidence, pinned model, temperature 0, and luck. Don't target this. |
| **R2 finding-identical** | Same set of (rule, outcome, final severity) | **Target this.** |
| **R3 coverage-identical** | Same checks evaluated, even if some outcomes differ | Guaranteed by the planner design |
| **R4 explainable variance** | When outcomes differ, it's flagged `needs_human` instead of silently flipping | Guaranteed by self-consistency |

## The loop

```
intake ──▶ normalize ──▶ route ──▶ select ──▶ plan
                                                │
         ┌──────────── for each check (parallel) ┘
         ▼
   gather evidence ─▶ precedent lookup ─▶ judge ×N ─▶ agree? ──no──▶ needs_human
                                                        │yes
                                                        ▼
                                            severity engine (code)
         ┌──────────────────────────────────────────────┘
         ▼
   explorer (optional, quarantined) ─▶ verifier ─▶ reporter ─▶ proposer ─▶ manifest
```

### Stage notes

**Intake / normalize.** A review request is a typed object: `{subject_id, review_type, scope, requested_by, inputs[]}`. Free-text requests get normalized into this shape first, and a human confirms the scope for high-stakes reviews.

**Judge.** This is the core LLM call. Its constraints:

- It answers **one rule** at a time.
- Output is **schema-constrained** (structured output / tool call), with no free prose outside the `rationale` field.
- Valid outcomes are `pass | fail | not_applicable | needs_human`. **`needs_human` is a first-class, good outcome.** It's what stops the model from guessing.
- Every factor value and the outcome must cite evidence IDs. An uncited claim fails the verifier.
- Low temperature.

**Self-consistency (×N).** For each check, run the judge N times (N=3 is a sane default), optionally with shuffled evidence order.

- All N agree → accept, `confidence = 1.0`.
- Majority agrees → accept the majority, record `confidence`, flag it for sampling QA.
- No majority, or a pass/fail split → `needs_human`.

This turns run-to-run variance into a visible signal instead of a silent flip. To control cost, run N=1 first and escalate to N=3 only for `fail`, high-severity, or low-margin results.

**Severity engine.** A pure function. Same factors produce the same severity, every time. See doc 03.

**Explorer.** A separate, open-ended prompt: "Given the evidence, what security concerns are *not* covered by the checklist?" Its output goes in a separate report section labelled **"Exploratory, not scored"**, and **it feeds the Proposer**. This is how you discover missing rules without hurting repeatability.

**Verifier**, all deterministic checks:

- Every planned check has exactly one outcome.
- Every citation resolves to a real `EvidenceRecord`, rule, or ruling in the pinned snapshot.
- Quoted evidence actually appears in the cited record (substring match).
- Severity equals `engine(factors)`, so nobody hand-edited it.
- No `pass` rests only on T3 evidence.
- If verification fails, re-run that check once. If it fails again, the outcome becomes `needs_human`.

**Reporter.** A fixed template driven by findings. The LLM may write an executive summary, but only *from the findings object*, never from raw evidence, so the summary can't introduce claims that aren't in the findings.

**Run manifest**, everything needed to replay the run:

```yaml
run_id: RUN-2026-10-27-0012
subject: { id: app-checkout-assistant, review_type: llm-feature-review }
snapshot: 9f3e…c21
model: { id: <pinned>, temperature: 0.2, n_consistency: 3 }
prompts: { router: v3, checker: v7, explorer: v2 }
retrieval: { k: 3, tau: 0.78, tiebreak: [score_desc, decided_on_desc, id_asc] }
evidence_cache: runs/RUN-…/evidence/   # every tool response, hashed
plan: [LLM-001, LLM-004, AUTHZ-003, …]
retrieved: { LLM-004: { rulings_applied: [], rulings_surfaced: [RUL-2026-0017@77ab…] } }
findings: findings.json
timing, token usage, cost
```

**Replay mode** re-runs the judge against `evidence_cache` with no live calls. That's how you debug a disputed finding and how golden eval cases get created.

## Human-in-the-loop points

1. **Scope confirmation** for high-stakes reviews.
2. **`needs_human` queue.** A reviewer adjudicates, and that decision is logged.
3. **Final sign-off** on reports above a severity threshold.
4. **Override capture.** Whenever a human changes an outcome or severity, record *which factor they disagreed with*. That's your highest-value drift and improvement signal (doc 06).

## Scale considerations

- Checks are independent, so fan them out. Use a work queue keyed by `(run_id, rule_id)`.
- Cache static prefixes (system prompt + org context + rule body) with prompt caching. They're identical across runs and subjects.
- Rate-limit per source adapter, since MCP servers and APIs are usually the bottleneck rather than the LLM.
- Budget per run: max checks, max tokens, and max tool calls per check. Exceeding the budget yields `needs_human`, never a silent truncation.

## Open questions

- What's the acceptable `needs_human` rate? Early on, 20–30% is healthy because it means the system is honest. Over time, rulings should drive it down.
- Batch reviews (nightly sweeps across many assets) or on-demand (per PR or design doc)? Both work with this design, but they need different queueing and cost models.
- Who reviews the Explorer output, and how often?
