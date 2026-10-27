# 08: Prior Art: Uber

Uber has published two systems that overlap with this design. Neither is a direct template, but both confirm several choices here and suggest a few changes.

> Sources: Uber's blog (uber.com) wasn't reachable from the session that wrote this, so the uReview notes come from search summaries of the post. The ADR notes come from the public GitHub repo and README. Re-check the details against the originals.

## 1. uReview: AI code review at scale

**What it is:** Uber's GenAI code review system for internal PRs. Its core, "Commenter," is a modular, multi-stage pipeline.

**Notable design points**
- **Eligibility filter first.** Config files, generated code, and experimental directories are excluded *before* any LLM call.
- **Prompt chaining as separate stages:** generate comments → filter → validate → deduplicate. Each stage can be changed and evaluated independently.
- **Specialized assistants per issue class,** each with its own prompts, context, and evals.
- **Context assembly per file:** surrounding functions, class definitions, and imports, not the whole repo.
- **Success measured by usefulness to humans:** around 75% of comments rated useful and around 65% addressed, versus a reported 51% for human reviewers. These are secondary-source numbers.

**How it maps to this design**

| uReview | Here | Takeaway |
|---|---|---|
| Eligibility filter | Router/Planner (`not_applicable_when`) | **Add an explicit scoping stage** that drops low-value evidence before planning. It's cheap, and it removes noise. |
| Generate → filter → validate → dedupe | Judge → Verifier | **Add a dedupe/merge stage.** Several rules can fire on the same root cause, so findings should group by root cause in the report. |
| Specialized assistants | Rule categories and playbooks | Same idea. Version prompts and evals **per category** so changes in `llm/` can't regress `authn/`. |
| Usefulness / addressed rate | Override analytics (doc 06) | **Add "addressed rate" and "useful" ratings** as first-class metrics alongside recall and precision. |

## 2. ADR: Agentic AI Detection and Response (open source, `uber/ADR`)

**What it is:** a security layer that watches AI agents (Claude Code, Cursor, Codex, support bots) at runtime and detects unsafe sessions. It's Apache-2.0 licensed, and there's an MLSys 2026 paper.

**Notable design points**
- **Two-tier detection.** Tier 1 is a cheap, high-recall triage model that routes to `BENIGN_EXIT` or `ESCALATE`. Tier 2 is a stronger agentic reasoner that runs only on escalated sessions and reviews the source of the components involved.
- **Context through MCP "context providers"** backed by YAML: a threat intelligence server (threat repository YAMLs), a policy store server (`policy_store.yaml`), and a source code analyzer server (a source code registry). These are listed in a `context_providers_registry.json`.
- **Trust boundary between tiers.** Tier 1's free-form text is *not* copied into Tier 2's trusted prompt slot. Only validated, bounded fields cross over.
- **Fail-safe direction.** Missing or malformed structured output fails toward `ESCALATE`.
- **Run manifest** with SHA-256 digests of the inputs the detector actually read, plus pinned dependencies and pinned results directories.
- **Benchmark (ADR-Bench)** with about 300 labelled tasks, 134 MCP servers, and 17 attack techniques. It reports precision, recall, F1, and a confusion matrix.
- **Output is binary** (malicious or benign) plus route and risk fields. There's **no severity scale**.

**How it maps to this design**

| ADR | Here | Takeaway |
|---|---|---|
| Two-tier triage → deep reasoning | N=1 → N=3 escalation (doc 05) | **Use different models per tier,** not just more samples. A cheap model screens every check, and a stronger model plus self-consistency handles `fail`, high-severity, and ambiguous results. That's the main cost lever at scale. |
| MCP context providers (policy, threat intel, source) backed by YAML | `knowledge/` snapshot plus lookup tools | **Serve the knowledge snapshot as an MCP server** (`get_rule`, `find_rulings`, `lookup_reference`, `get_severity_factors`). Same knowledge, one interface, and swappable between agents. |
| Tier-1 text never enters Tier-2's trusted slot | Trust levels T0–T3 | Apply the same rule *between internal stages*. Pass only schema-validated fields from one LLM stage to the next, never prose. |
| Fail toward `ESCALATE` | Verifier failure → `needs_human` | Same principle. Keep it. |
| Run manifest with input digests | Run manifest (doc 05) | Same principle. Also digest the *exact bytes* the judge read, not just the record IDs. |
| ADR-Bench + ground-truth labels | Golden + adversarial evals (doc 06) | Consider **seeding the adversarial set from ADR-Bench and AgentDojo** for prompt-injection cases. |
| No severity model | Severity engine (doc 03) | A gap in ADR that this design covers. Keep severity computed from factors. |

## Net changes to fold in

1. Add a **scoping/eligibility stage** before planning (from uReview).
2. Add a **dedupe / root-cause grouping stage** after judging (from uReview).
3. Make escalation **two-tier by model** (cheap triage → strong reasoner), not just by sample count (from ADR).
4. Expose the knowledge snapshot through an **MCP knowledge server** (from ADR).
5. Allow **only validated structured fields** between LLM stages (from ADR).
6. Add **usefulness and addressed-rate** metrics (from uReview).
7. Seed adversarial evals from **ADR-Bench / AgentDojo** (from ADR).

## Links

- uReview post: https://www.uber.com/blog/ureview/
- ADR repo: https://github.com/uber/ADR
- ADR detection README: https://github.com/uber/ADR/blob/main/Detection/README.md
- Coverage: https://techstrong.ai/articles/uber-open-sources-tool-to-catch-agentic-blunders-legacy-security-misses/
