# 01: Architecture

## Top-level shape

```
                ┌───────────────────────────── KNOWLEDGE PLANE (slow, human-owned) ─────────────────────────────┐
                │  knowledge/ repo ──PR──▶ CI: validate schemas, lint, contradiction check, evals ──▶ SNAPSHOT     │
                │                                                                        (immutable, hashed)     │
                └──────────────────────────────────────────────────────────────────────────────┬─────────────────┘
                                                                                               │ pinned by run
┌──────────── EVIDENCE PLANE (fast, untrusted) ────────────┐                                   ▼
│ MCP servers · REST/GraphQL APIs · reports (PDF/MD/HTML)  │──▶ adapters ──▶ normalized ──▶ ┌──────────────────┐
│ reference tables (CSV/warehouse) · code/repos · tickets  │    (read-only,   evidence       │  REVIEW RUNTIME  │──▶ report + findings
└──────────────────────────────────────────────────────────┘     trust-tagged) records       │ route→plan→check │──▶ run manifest
                                                                                            │ →score→verify    │──▶ inbox proposals
                                                                                            └──────────────────┘
                                                                                                     │
                ┌──────────────────────────── FEEDBACK PLANE ─────────────────────────────┐          │
                │ human adjudication of findings ─▶ override telemetry ─▶ candidate rulings ◀──────────┘
                │                                   ─▶ new golden eval cases ─▶ PR into knowledge/        │
                └──────────────────────────────────────────────────────────────────────────┘
```

Three planes, three speeds:

- **Knowledge plane** changes weekly or monthly, through review. It is the source of truth for "what is a problem and how bad is it."
- **Evidence plane** changes every run. It is *what is true about this subject right now*.
- **Feedback plane** is the only path for learning, and it always ends in a human-reviewed PR.

## Components

| Component | Responsibility | Deterministic? |
|---|---|---|
| **Source adapters** | Wrap each MCP server or API. Normalize output into `EvidenceRecord`s with source, timestamp, trust level, and hash. | Yes (I/O aside) |
| **Indexer** | Build snapshot indexes from `knowledge/`. Optionally build per-run indexes over large evidence such as long reports. | Yes |
| **Router** | Map the review subject to facets (asset type, data classes, tech stack, exposure). | Mostly. Rules first, LLM fallback with closed vocab. |
| **Planner** | Facets → playbook → rule pack → ordered checklist. | Yes, pure lookup |
| **Checker** | For each check, gather evidence, then the LLM judges `pass / fail / not_applicable / needs_human` with citations. | No. This is the only real LLM judgment step. |
| **Severity engine** | Compute severity from factors via `severity/matrix.yaml`, then apply precedent overrides. | Yes |
| **Explorer** (optional) | Open-ended "what else looks wrong" pass. Output is labelled *exploratory*, never merged into the scored findings. | No |
| **Verifier** | Schema-validate, confirm every citation resolves, confirm every check has an outcome, and enforce severity/evidence consistency. | Yes |
| **Reporter** | Render findings through a fixed template. | Yes |
| **Proposer** | Turn novel patterns and human overrides into candidate rulings in `inbox/`. | No, but it can't write to canonical knowledge |

## Source trust levels

Every evidence record carries a trust level. Prompts treat evidence as *data inside delimiters*, never as instructions.

| Level | Examples | Handling |
|---|---|---|
| `T0 authoritative` | Your CMDB, IAM API, data classification registry | Can set facets directly |
| `T1 system-generated` | Scanner output, cloud config APIs, SBOMs | Factual, but may be stale. Check timestamps. |
| `T2 human-authored internal` | Design docs, tickets, past reports | Claims, not facts. Corroborate before relying on them. |
| `T3 external / subject-controlled` | Code under review, vendor docs, README files, web | Potentially adversarial. Prompt-injection surface. |

Security of the reviewer itself:

- Tools are read-only by allowlist. No tool the reviewer can call changes state, except writing to its own run output and the inbox.
- The agent never sees credentials. Adapters hold them.
- Instructions live only in `prompts/` and `knowledge/`. Evidence text is quoted, length-capped, and tagged with its source.
- A `T3` source can never *lower* a severity or mark a check `pass` by itself. It needs a `T0` or `T1` corroboration, or the check goes to `needs_human`.

## Why not "one big agent with all the tools"?

It's tempting, and it's what makes output non-repeatable. An open-ended agent picks different tools, reads different docs, and frames the problem differently on every run. The checklist-driven pipeline above keeps the open-ended part (the Explorer) quarantined, and makes the scored part a bounded classification task.

## Open questions

- What review types exist? Examples: design review, vendor/third-party, code/PR, cloud config, AI/LLM feature review. Each one becomes a *playbook*.
- Which sources are T0? Is there an asset inventory and data classification registry you can query, or does that need to be built as reference tables first?
- Is output consumed by humans only, or also by systems such as ticketing or a GRC tool? That decides how strict the finding schema must be.
