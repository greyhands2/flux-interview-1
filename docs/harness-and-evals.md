# Campaign Performance Analyst — harness and evaluations

## Runtime harness

The harness is the small runtime layer that makes each model or data call bounded, typed, observable, and replaceable. It owns workflow state, deadlines, budgets, validation, error mapping, and trace events. It does not delegate control flow to the model.

### Interfaces

Define narrow interfaces after inspecting repository conventions:

- **Model client:** accepts an explicit task, bounded messages/context, output schema, deadline, and token budget; returns structured output, provider/model metadata, usage, and a typed error. Keep extraction, drafting, and optional review as distinct task kinds with distinct prompts/schemas.
- **Analytics source:** exposes only approved read operations with typed metric IDs, campaign identifiers, date windows, and allowlisted dimensions. Return typed data rows and provenance/freshness metadata. It has no write method and accepts no arbitrary SQL or expressions.
- **Clock/deadline and trace sink:** make timeouts, deterministic evaluations, and trace capture controllable in local tests/evals.
- **Human-review destination:** an optional interface for a configured queue. Until one exists, return a local review-ready bundle instead of claiming successful routing.

Model and source adapters should be independently replaceable. Prefer the standard library and small dependencies unless repository inspection or operational needs show a clear benefit from a dependency.

### Context by step

- **Field extraction:** original question, allowed metric names, known campaign names/IDs, supported date conventions, user time zone if available, and a strict output schema. No raw analytics data.
- **Clarification:** unresolved fields and allowed choices only. Do not ask for details already confidently resolved.
- **Query planning/retrieval:** no model context. Deterministic planner produces typed parameters; source adapter receives only those parameters plus authorized scope.
- **Calculation/checks:** no model context. Operate on validated typed rows, definition version, and derived facts.
- **Drafting:** sanitized question, approved definition excerpt, calculation outputs, relevant evidence snippets/IDs and limitations. No credentials, hidden source data, or unrelated rows.
- **Model-based review, if enabled:** candidate answer, same evidence/facts, required answer contract, and explicit instruction to identify specific unsupported claims or omissions. It cannot mark its own result as proof of correctness.

### Validation and budgets

Validate every model response against a strict schema and semantic constraints (known metric/entity IDs, valid date formats, bounded output length, citations drawn from supplied evidence IDs). Treat markdown and prose as untrusted. Reject unexpected fields where practical.

The run budget includes:

- One extraction call and, only where useful, one drafting call; optional reviewer use must have a defined budget and should be omitted initially if it does not add measured value.
- At most one answer revision after a failed check.
- At most one transient retry for a read operation, with backoff and only inside the run deadline.
- Configurable total deadline, per-step timeouts, tool-call count, date-span/row bounds, prompt size, output tokens, and cost ceiling.
- No background agent loops, self-directed tool selection, or model-issued query plans.

If a budget expires, return a typed limitation such as timeout, data unavailable, or budget exceeded. Do not return a partial draft as a supported conclusion unless the contract explicitly identifies it as incomplete.

Map failures into stable categories: invalid/unsupported request, needs clarification, permission denied, source unavailable, stale/incomplete data, malformed model output, validation failure, budget/deadline exceeded, and review required. Avoid retrying semantic failures or permission errors.

### Trace fields

Record a run ID; start/end and step timings; workflow version; current step/status; question fields after redaction; clarification count; metric ID/definition version; source adapter and approved query shape; evidence IDs/count and freshness; calculation inputs, formula version, outputs and units; model/provider/version/task and prompt-template version; token usage and estimated cost where provided; retry/revision counts; validation/check names and outcomes; final answer status or limitation; and review-bundle reference.

Keep secrets out of traces. Apply data minimization and retention rules to question text, evidence, and reviewer notes. Log hashes/IDs or redacted summaries rather than full raw analytics records unless an approved audit policy requires otherwise.

## Deterministic and model-based checks

### Deterministic checks (release gates)

- Required answer fields and status are present.
- Every cited evidence ID exists in the run and matches the stated window, metric, and campaign.
- Values, units, signs, deltas, and percentages match deterministic calculations within the defined display-rounding policy.
- A zero denominator is shown as undefined/unavailable rather than infinity or a fabricated value.
- Unsupported/missing/stale values cannot be presented as facts.
- Required metric-definition and comparison-window caveats are present.
- Unsupported metrics or breakdowns cannot trigger data access.
- Revision count never exceeds one; source and model retries stay within their configured bounds.
- Input/output size, date span, row count, tool calls, and deadlines remain within limits.

### Model-based judgments (advisory)

A separate bounded reviewer may assess whether prose is understandable, distinguishes observation from hypothesis, answers the question, and appears to make unsupported causal claims or omit an important limitation. Keep findings specific and trace them.

Model review can miss errors, agree with a flawed draft, or be biased by the same context and model family. It cannot certify calculations, citations, source quality, privacy compliance, or causal truth. Use human audits and deterministic evidence checks for those concerns. Do not present a reviewer pass as a correctness guarantee.

## Repeatable offline evaluation set

Keep a version-controlled set of synthetic fixtures, approved-definition snapshots, questions, expected structured outcomes, and expected check results. Run regressions with a fake model client that returns scripted outputs and a fixture source; tests must make zero live model or analytics calls. Include these minimum cases:

| Case | Input/setup | Expected behavior and checks |
|---|---|---|
| Ambiguous metric/campaign/period | Ask “How did the campaign do?” with multiple plausible campaigns and no period | Ask a focused clarification before retrieval; no source call until required fields resolve |
| Unsupported metric | Ask for a metric absent from the approved registry (for example, view-through revenue when not configured) | Decline with supported alternatives; no retrieval |
| Missing data | One comparison period has absent or stale conversion data | Return a limitation; do not infer a delta from missing values |
| Incorrect calculation | Script a draft with a wrong CPA or percentage change despite correct fixture rows | Deterministic numeric check fails; one revision can correct it, otherwise limitation/review |
| Invalid citation | Draft cites an unknown evidence ID or evidence from another campaign/window | Citation validator fails; revision once maximum |
| Unsupported causal claim | Metrics show CPA rose while impressions fell, but no causal evidence exists; draft says impressions caused the rise | Reviewer flags it where possible; deterministic policy rejects causal language unless supported by an approved attribution method; otherwise route/limit |
| Malformed model output | Extraction or drafting returns invalid JSON, unknown IDs, or missing fields | Schema/semantic validation fails safely; no unvalidated data access or answer |
| Retry limit | Script repeated transient source errors and a failed answer check | At most one transient read retry and one answer revision; then a clear limitation or review outcome |

Add cases as real metric definitions and source adapters are approved. Pin fixture versions and expected outcomes so results are reproducible.

## Metrics for evaluation and operation

- **Quality:** field extraction exact match; clarification precision and recall; metric calculation accuracy against independent golden values; citation validity and coverage; limitation correctness; unsupported-claim rate from blinded human review; reviewer agreement and user-rated usefulness.
- **Latency:** end-to-end median/p95, each step’s median/p95, source wait, model wait, and timeout rate.
- **Cost:** model calls, input/output tokens, estimated cost per run and per successful answer, source-query volume, and revision/retry rates.
- **Reliability and experience:** source success/freshness, malformed output frequency, check failure/recovery rates, human-review rate, clarification completion/abandonment, and answer usefulness.

Report the denominator and slices (metric, source, question class, and failure class); averages can conceal unsupported or high-risk segments. Establish targets from a pilot baseline instead of inventing them in advance.

## Limits of the evaluation set

Eight hand-built synthetic cases are a starting regression suite, not evidence of broad real-world correctness. They cannot represent all wording, campaign naming collisions, time-zone boundaries, attribution changes, source outages, permissions, data drift, or adversarial prompts. Scripted models also do not measure live model variability, latency, or cost. Expand the suite with privacy-reviewed, de-identified examples and analyst adjudication; sample real pilot answers for ongoing human audit; run live-model evaluations separately from offline regressions and never make those live calls a prerequisite for deterministic regression runs.
