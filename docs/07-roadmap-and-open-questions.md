# 07: Roadmap and Open Questions

## Phased path to a working skeleton

### Phase 0: Contracts (week 1–2)
- [ ] Write `schemas/` for rule, ruling, playbook, evidence-record, finding, and run-manifest
- [ ] `taxonomy/facets.yaml` and `categories.yaml` (controlled vocab)
- [ ] `severity/` factors, matrix, and modifiers, plus unit tests for the severity engine
- [ ] ADR-001: Knowledge format (Markdown + frontmatter)
- [ ] ADR-002: Severity is computed, never generated

### Phase 1: Thin vertical slice (week 3–5)
Pick **one review type** (suggestion: `llm-feature-review`, because it's bounded and current) and **10–15 rules**.
- [ ] 2–3 source adapters (one MCP, one API, one file/report)
- [ ] Indexer v0: validate, build `manifest.json` and `facets.sqlite` (structured only, no vectors yet)
- [ ] Runtime v0: route → select → plan → judge (N=1) → severity → verify → report
- [ ] Run manifest plus replay mode
- [ ] 10 golden cases and 5 adversarial cases

### Phase 2: Precedent and repeatability (week 6–8)
- [ ] Rulings with structured fact-pattern matching
- [ ] Self-consistency (N=3 escalation) and the `needs_human` queue
- [ ] Lexical index (SQLite FTS5) and a vector index for surfaced precedent and long reports
- [ ] Eval harness in CI with gates and baselines
- [ ] Stability metric (K-run agreement)

### Phase 3: Learning loop (week 9–12)
- [ ] Explorer, Proposer, and inbox triage workflow
- [ ] Override capture plus analytics
- [ ] Expiry, review_by, and contradiction checks in CI
- [ ] Canary replay before rollouts

### Phase 4: Breadth
- [ ] Additional playbooks (design, vendor, cloud config, code/PR)
- [ ] Control framework mapping and compliance exports
- [ ] Prior-review retrieval per subject
- [ ] Scale-out: queueing, prompt caching, cost budgets

## Decisions to make next (candidate ADRs)

| # | Decision | Leaning |
|---|---|---|
| 1 | Knowledge format | Markdown + YAML frontmatter + JSON Schema |
| 2 | Severity generation | Computed from factors; the LLM proposes factors only |
| 3 | Retrieval order | Structured → lexical → semantic; semantic only *surfaces* |
| 4 | Agent write access | `runs/` and `inbox/` only |
| 5 | Retrieval unit | Per-check, not per-review |
| 6 | Index tech for v1 | SQLite (facets + FTS5) + flat vectors inside the snapshot |
| 7 | Repo topology | Monorepo now, split `knowledge/` later if ownership diverges |
| 8 | Repeatability target | R2 (finding-identical), with variance routed to `needs_human` |

## Big open questions for you

1. **Review types:** which ones, and which comes first?
2. **Subjects:** what is being reviewed? Apps, vendors, AI features, PRs, cloud accounts?
3. **T0 sources:** do an asset inventory and data classification registry exist, and are they reachable through MCP or an API?
4. **Existing material:** are there current policies, standards, or past review reports to seed rules and rulings from? Mining past reports for rulings is often the fastest bootstrap.
5. **Consumers:** who reads the output, and where does it go (tickets, GRC, Slack, PR comments)?
6. **Ownership:** who approves rules, rulings, and severity changes?
7. **Scale:** reviews per week, expected rule count, latency and cost budget per review.
8. **Compliance:** do findings need to map to SOC 2, ISO 27001, NIST, or OWASP LLM Top 10 for audit evidence?

## Riskiest assumptions to test early

- **Can rules really be written as single checkable assertions?** Try converting 10 real policy statements and see how many survive.
- **Will structured fact patterns cover most precedent?** Mine 20 past decisions and try to encode them.
- **Is N=3 self-consistency enough to get R2 above about 95%** on the golden set? Measure it in Phase 2 before scaling.
- **Prompt-injection resilience:** do adversarial evidence cases ever flip a `fail` to `pass`? This must be zero.
