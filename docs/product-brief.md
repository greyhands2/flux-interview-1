# Campaign Performance Analyst — product brief

## User problem

Marketing teams can see that a campaign metric changed, but explaining why often means reconciling definitions and pulling reports from multiple systems. A conversational answer can save time only if it uses approved definitions, shows where its evidence came from, distinguishes observation from explanation, and admits when the data does not support a conclusion.

## Intended users

- **Primary:** performance marketers and campaign managers investigating changes in spend, conversions, or efficiency.
- **Supporting:** marketing analytics owners who approve metric definitions, source access, and interpretation rules.
- **Reviewers:** analysts who handle unsupported or ambiguous requests and verify high-impact explanations.

The initial release is for one team with a small, explicit set of approved metrics and one configured analytics source.

## Primary workflow

1. A marketer asks a question in a simple command-line interface.
2. The system identifies the requested metric, campaign, date range, comparison period, and desired breakdown. It validates them against a registry. It asks one concise follow-up if a required field remains ambiguous.
3. It retrieves only the approved, bounded data needed for the question through read-only tools.
4. Deterministic code computes metric values, deltas, and component changes where the metric definition permits it.
5. A language model drafts a concise explanation grounded in the returned evidence. It separates observed changes from possible causes.
6. Programmatic checks verify calculations, cited evidence IDs, required caveats, and output shape. A separate model-based review can flag unsupported prose, but is a fallible additional check.
7. On a failed check, the system may revise once using structured findings. It then returns a limitation or routes the case to a human reviewer; it never loops indefinitely.

Answers show the metric definition, current and comparison windows, values and units, source/as-of details, evidence references, and whether statements are observed facts, hypotheses, or limitations.

## Initial working version

Build a small but useful single-user CLI in Go, subject to repository inspection confirming no existing language or architecture that should be followed. Use a narrow workflow implemented as ordinary typed functions and explicit state transitions. Do not adopt an orchestration framework initially: a few bounded steps do not need framework lifecycle complexity, while explicit Go control flow is easy to debug, test offline, and instrument. Keep model and data-source interfaces replaceable so a framework or service boundary can be considered when real operational needs justify it.

Include:

- A versioned, local registry of a few approved metric definitions (for example spend, conversions, and CPA) with clear numerator, denominator, currency, attribution window, and aggregation rules. Owners must approve the actual definitions before use.
- One real, read-only source integration for the first team, selected after confirming the source, credentials, access policy, and metric parity. The adapter exposes bounded typed operations, never arbitrary SQL. A deterministic fixture adapter supports local development and offline regression evaluations.
- A CLI that accepts one question, asks a clarification in the same run, and prints a structured, evidence-bearing response. No web app is needed to validate the core value.
- Deterministic date handling, metric calculations, validation, citation resolution, output formatting, and stop/retry decisions.
- Optional model use through a small interface for intent extraction and answer drafting. Validate every model response against a schema and approved registry. Handle malformed or unavailable output as a safe limitation.
- One bounded revision after check failure, then a limitation or human-review outcome. Human review may initially mean a review-ready explanation and evidence bundle saved or printed for an analyst, rather than a built-in dashboard.

Local development can use synthetic or anonymized fixtures only; never copy production personal or sensitive data into fixtures. A real source integration, approved metric registry, model credentials, and access controls require team decisions and environment configuration. No database is needed: keep run state in memory and optionally emit a local structured trace with sensitive values redacted.

## Later production work

Only add after the first team validates demand and operational requirements:

- Hosted service, authentication, role-based authorization, team UI, and managed human-review queue.
- Durable run storage, retention/deletion controls, encryption, audit search, and operational dashboards.
- More approved sources, freshness monitoring, connector health checks, and reconciliation against source-of-truth reports.
- Secret management, deployment hardening, rate limits, quotas, concurrency controls, and cost budgets by user/team.
- Privacy review, data minimization, field-level access, redaction policies, and incident response.
- Background execution, cancellation, resumable review workflows, and notifications if interactive latency proves insufficient.

These are production concerns, not prerequisites for validating the narrow workflow. A database is not justified until durable run history, team access, or review queues are actual requirements.

## Non-goals

- Replacing a BI platform, campaign-management system, or analyst judgment.
- Changing bids, budgets, campaigns, or source data.
- Running user-authored SQL or querying arbitrary tables.
- Inferring causality from a before/after comparison alone.
- Supporting every marketing platform, metric, attribution model, or custom formula at launch.
- Guaranteeing correctness through model-based review.
- Building a general autonomous agent or multiple collaborating agents.

## Reliability, auditability, privacy, cost, latency, and failure behavior

- **Reliability:** approved definitions, bounded typed tools, deterministic computations, strict validation, explicit timeouts, and finite retries. Missing, stale, partial, or inconsistent data must be surfaced as such.
- **Auditability:** retain a trace of normalized question fields, metric-definition version, source and query parameters, returned evidence IDs and timestamps, calculation inputs/results, model/version and prompt-template identifiers, check outcomes, and final status. Do not log raw credentials or unnecessary personal data.
- **Privacy:** least-privilege read-only source credentials; request only required dimensions and date range; minimize prompt context; redact sensitive fields. Local fixtures must be synthetic/anonymized.
- **Cost:** use the model only for extraction/drafting/review where language judgment helps. Bound prompt size, tool calls, output tokens, and the single revision. Cache only when source freshness and access rules allow.
- **Latency:** answer synchronously for bounded short date ranges. Set per-step and total deadlines; return a clear timeout limitation. Measure end-to-end and per-step latency before choosing hosted or asynchronous execution.
- **Failure handling:** invalid/ambiguous input gets clarification; unsupported metric/source gets a limitation; source timeout or malformed data gets a retrieval failure message; calculation/check failures get one revision then a limitation or review route. Never fabricate missing values or silently substitute a metric.

## Success measures

For a small pilot, measure:

- Share of supported questions completed without analyst intervention, alongside a sampled human audit of correctness.
- Correctness of metric extraction, period interpretation, deterministic calculations, evidence references, and limitation behavior on a fixed offline evaluation set.
- Rate of unsupported causal language and rate of answers appropriately declining to conclude.
- Median and p95 end-to-end latency; source and model step latency.
- Model and source cost per completed question; average calls and tokens per run.
- Clarification rate, abandonment after clarification, human-review rate, and user-rated usefulness.
- Source freshness/availability and data reconciliation failures.

Set numeric targets with the pilot team after establishing a baseline; do not treat user satisfaction or model-review scores as proof of factual correctness.
