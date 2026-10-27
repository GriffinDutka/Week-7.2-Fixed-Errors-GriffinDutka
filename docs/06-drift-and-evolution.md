# 06: Drift and Evolution: Building on Context Without Rot

## Types of drift

| Drift | What happens | Primary defense |
|---|---|---|
| **Knowledge drift** | Rules or rulings become stale, contradictory, or quietly edited | Append-only rulings, `review_by` / `expires`, contradiction check in CI, CODEOWNERS |
| **Context accretion drift** | The agent "remembers" things, and errors compound over time | **The agent never writes canonical knowledge.** All of its proposals go through the inbox, then a PR. |
| **Model drift** | A provider updates the model and outcomes shift | Pin model IDs, and treat any model change as a release gated by evals |
| **Prompt drift** | Small prompt tweaks shift behavior in unrelated checks | Versioned prompts plus a full eval run on every change |
| **Retrieval drift** | Re-embedding or new docs change what gets retrieved | Snapshot pinning, retrieval logged in the manifest, retrieval-level evals |
| **Evidence drift** | Live sources change shape or meaning | Adapter contract tests, schema validation on `EvidenceRecord`, TTL caching |
| **Calibration drift** | The share of `high` severities creeps up or down over months | Monitor the outcome and severity distribution per rule over time, and alert on shifts |

## The learning loop: how the system builds on context

```
 run ──▶ findings ──▶ human adjudication ──▶ override log
  │                                             │
  └─▶ explorer notes ──┐                        │
                       ▼                        ▼
                    PROPOSER  (LLM drafts candidate rulings / rules / eval cases)
                       │
                       ▼
                 inbox/  ──▶ triage (human) ──▶ PR into knowledge/ + new golden case
                                                     │
                                                     ▼
                                     CI: validate + contradiction check + FULL EVAL
                                                     │
                                                     ▼
                                              new snapshot ──▶ future runs
```

Key properties:

- **Every lesson becomes two artifacts:** the ruling (so future runs apply it) and a golden eval case (so nothing regresses it).
- The Proposer is allowed to be creative because its output is never trusted directly.
- Rulings are **append-only**. You supersede them, never edit. History stays auditable, and you can always answer "why did we rule this way in May?"

## Golden eval set (the backbone)

Each case is a frozen subject and frozen evidence (recorded tool responses), plus expectations:

```yaml
# evals/golden/case-0042/expected.yaml
must_find:
  - { rule: LLM-004, outcome: fail, severity_in: [high, critical] }
must_not_find:
  - { rule: AUTHN-001, outcome: fail }          # precedent RUL-2026-0041 should apply
must_be_not_applicable: [DATA-007]
allowed_needs_human: [LLM-009]
```

**Metrics tracked per (snapshot × prompt versions × model):**

- **Recall** on `must_find`, the most important one. Missing a real issue is the worst failure.
- **Precision**: `must_not_find` violations
- **Severity agreement**: exact and within-one-level
- **Stability**: run every case K times (for example 5) and measure the R2 agreement rate (doc 05)
- **needs_human rate**
- **Citation validity rate**, which should be 100%. Anything else is a bug.

**Gates:** a PR to `knowledge/`, `prompts/`, `runtime/`, or the pinned model can't merge if recall drops, stability drops beyond a tolerance, or citation validity falls below 100%. Baselines are stored in `evals/baselines/`.

**Sources of cases:** every disputed finding, every new ruling, every explorer discovery that became a rule, plus a deliberate **adversarial set**: prompt injection in READMEs and design docs, misleading vendor claims, stale evidence, and contradictory sources.

## Hygiene rules

1. **Expiry.** Rulings expire (default 12 months). Rules have `review_by`. CI warns 30 days ahead and fails builds once a date is past. Precedent that's never revisited is how drift accumulates unnoticed.
2. **Supersession, not edits.** Changing a decision means a new ruling with `supersedes: RUL-…`.
3. **Contradiction check.** Two active rulings with the same structured pattern and different decisions fail the build.
4. **Coverage map.** Generate a report of rules × playbooks × golden cases, so rules with zero eval coverage are visible.
5. **Override analytics.** If humans override rule X more than about 20% of the time, the rule, its guidance, or its severity defaults are wrong. Open a ticket automatically.
6. **Distribution monitoring.** Track outcome and severity histograms per rule per month. A sudden shift without a corresponding knowledge change means something drifted underneath you: the model, the evidence, or the retrieval.
7. **Canary reviews.** Before rolling out a new snapshot, model, or prompt, re-run the last N production reviews in replay mode and diff the findings. A human reviews the diff.

## What "memory" means here

Avoid giving the agent free-form, self-written long-term memory. In this design, memory is:

- **Institutional memory:** rulings and rules. Curated, versioned, and cited.
- **Subject memory:** past run manifests and findings for the same subject, retrievable as *prior reviews*, explicitly labelled as such, and trusted at T2.
- **Run memory:** the scratchpad within a single run, discarded afterward.

Nothing moves from run memory into institutional memory without a human PR.

## Open questions

- Who staffs inbox triage, and on what cadence? This loop only works if someone empties the inbox.
- How big should the golden set be before going live? A reasonable bar is at least 3 cases per high-impact rule and at least 1 per rule overall, plus about 20 adversarial cases.
- Should prior-review findings for the same subject automatically raise the bar? For example, "this was a `fail` last quarter, so `pass` now requires T0/T1 evidence of remediation."
