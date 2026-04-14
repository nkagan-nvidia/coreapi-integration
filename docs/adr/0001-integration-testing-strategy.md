# ADR-0001: Integration Testing Strategy for CoreAPI

- **Status:** Proposed
- **Date:** 2026-04-14
- **Authors:** nkagan-nvidia

## Context

`coreapi-integration` houses the integration test suite for the CoreAPI service.
Testing must support two distinct use modes:

- **Correctness mode** — validate behavior at a single software version (pass/fail)
- **Comparative mode** — evaluate changes between two software versions
  (metrics delta, regression detection)

These modes have different signal semantics. Not all test tracks produce
standalone pass/fail signals; some require a baseline to be meaningful.

## Test Tracks

### 1. API Tests
Validate correctness of the CoreAPI surface — request/response contracts,
error handling, and edge cases.

- **Signal type:** Pass/fail (self-contained, no baseline required)
- **Usable in:** Correctness mode, Comparative mode

### 2. Benchmark Tests
Measure throughput, latency, and resource utilization of the CoreAPI service.

- **Signal type (primary):** Pass/fail — a benchmark that fails to complete
  is a hard failure regardless of mode
- **Signal type (secondary):** Metrics — only meaningful relative to a baseline;
  raw numbers alone cannot determine regression or improvement
- **Usable in:** Correctness mode (completion pass/fail only),
  Comparative mode (completion + metrics delta)

### 3. Mega Integration — Small Simulation Payload
End-to-end test driving the CoreAPI via Mega with a small, fast simulation workload.

- **Signal type (primary):** Simulation pass/fail (did the simulation complete correctly?)
- **Signal type (secondary):** KPIs available for comparative analysis
- **Milestone KPI goals:** Defined target thresholds that must be met at key
  milestones (treated as pass/fail against a known target, not a baseline run)
- **Usable in:** Correctness mode (pass/fail), Comparative mode (KPI delta),
  Milestone validation (KPI vs. target)

### 4. Mega Integration — Large Simulation Payload
Same as above but with a large, production-representative simulation workload.
Intended for milestone validation and deep comparative analysis.

- **Signal type (primary):** Simulation pass/fail
- **Signal type (secondary):** KPIs for comparative analysis
- **Milestone KPI goals:** Same milestone threshold structure as small payload
- **Usable in:** Correctness mode, Comparative mode, Milestone validation
- **Note:** Wall-clock cost is significant; not suitable for every CI run

## Decision

We adopt the following rules:

1. **API tests gate every run** — they are cheap and provide self-contained
   correctness signals.
2. **API tests and Benchmark tests run in parallel** — they are independent
   and can execute concurrently as the "candidate" suite.
3. **Benchmarks have two signal layers:**
   - *Completion pass/fail* — a benchmark that does not complete is a hard
     failure, usable standalone
   - *Metrics comparative* — requires a baseline run; only triggered when
     comparative results are requested
4. **Baseline runs are conditional** — a baseline is only scheduled when the
   caller requests comparative mode. Correctness-only runs do not require one.
5. **Mega integration tests have two verdict modes:**
   - *Simulation pass/fail* — usable standalone for correctness gating
   - *KPI comparative* — requires a baseline run; used in comparative mode
     and milestone validation
6. **Milestone KPI goals are encoded as explicit thresholds**, not derived from
   a baseline run, so milestone validation can run without a prior version.
7. **Large-payload tests are opt-in** — not run on every PR; triggered for
   milestone validation and comparative analysis on demand.
8. **Test track configuration is declared in `.fleet-testing.yaml`** — the file
   in the repo root is the canonical definition of all tracks, their signal
   types, output schemas, and mode applicability. The ADR describes the design
   rationale; the YAML is the machine-readable contract.

## Execution Model

- **Correctness run:** `[API + Benchmark]` in parallel → pass/fail verdict, no baseline needed
- **Comparative run:** `[baseline: API + Benchmark]` || `[candidate: API + Benchmark]` → metrics delta
- **Milestone run:** candidate suite + KPI threshold check

## Structured Data Requirements

All test track inputs and outputs, and the suite-level summary, must be
machine-readable structured data. Log output alone is not sufficient.

### Schema Selection

Different tracks have different signal types, so a hybrid schema approach is adopted:

#### Pass/Fail Results — CTRF (Common Test Report Format)
All test tracks emit a **CTRF JSON** result for their pass/fail signal.
CTRF is JSON-native, CI/CD-integrated (GitHub Actions, Jenkins, GitLab),
and provides a unified structure across tracks.

- API tests: full CTRF output (one entry per test case)
- Benchmark tests: CTRF entry recording completion pass/fail
- Mega integration tests: CTRF entry recording simulation pass/fail

#### KPI and Metrics — OpenTelemetry Metrics (OTLP JSON)
Benchmark and Mega integration tracks additionally emit **OTLP-format JSON**
for all numeric KPIs (throughput, latency, accuracy, resource utilization).

OTLP natively models histograms, counters, and summaries — well-suited to
performance metrics and comparative analysis across versions.

#### Suite-Level Summary — Versioned JSON Envelope
The suite produces a single top-level JSON document containing:
- `schema_version` — envelope version string
- `mode` — `"correctness"` | `"comparative"` | `"milestone"`
- `run_metadata` — software version(s), timestamp, environment
- `results` — CTRF array for all tracks
- `metrics` — OTLP exports for tracks that produce KPIs
- `deltas` — computed metric deltas (comparative mode only)
- `milestone_verdict` — KPI vs. threshold results (milestone mode only)

### Why Not JUnit XML
JUnit XML has excellent CI/CD support but no native model for numeric KPIs
or comparative results. Extensions are vendor-specific and fragmented.
CTRF provides the same CI/CD integration in a modern JSON format.

## CLI-First Execution Requirement

Test execution must be driven by CLI tools wherever possible. Agent involvement
should be reserved for tasks that require judgment — result interpretation,
triage, comparative analysis, and escalation — not mechanical orchestration.

### Principle

Every discrete, repeatable operation in the test pipeline must be expressible
as a CLI invocation with structured input and structured output. If no CLI
exists for an operation, that gap is a task to be filed, not a reason to
use an agent. The `.fleet-testing.yaml` file defines which CLI commands
implement each track.

### Expected CLI Surface

The following operations must be covered by CLI tools (existing or to be built):

| Operation | CLI Status |
|-----------|------------|
| Run API test track | TBD — identify or build |
| Run Benchmark track | TBD — identify or build |
| Run Mega integration (small payload) | TBD — identify or build |
| Run Mega integration (large payload) | TBD — identify or build |
| Emit CTRF result for a track | TBD — identify or build |
| Emit OTLP metrics for a track | TBD — identify or build |
| Assemble suite-level JSON envelope | TBD — build |
| Compute metrics delta between two runs | TBD — build |
| Evaluate milestone KPI thresholds | TBD — build |
| Invoke baseline + candidate in parallel | TBD — build (orchestration wrapper) |

### Gap Process

When a CLI tool does not exist for a required operation:
1. Document the gap in this table with status `Missing`
2. File a fleet task to build the CLI tool
3. Do not substitute agent execution as a workaround

### What Agents May Do

Agents may invoke CLI tools, interpret their structured output, and act on
results. Agents must not replicate logic that belongs in a CLI tool.

## Alternatives Considered

- **Rely solely on Mega's own test suite**: Rejected — Mega tests do not cover our
  configuration, environment, or integration patterns.
- **Mock GPU/CUDA calls**: Rejected — too much risk of divergence; failures in
  production would not be caught.
- **No pinning (always track Mega HEAD)**: Rejected — upstream breakage would
  block unrelated work.
- **JUnit XML for all output**: Rejected — no native model for KPI metrics or
  comparative results.
- **Treat all tests as pass/fail:** Rejected — benchmark metrics have no
  intrinsic pass/fail meaning without context; forcing a threshold produces
  false signals on hardware variance.
- **Single test mode (correctness only):** Rejected — comparative analysis
  is a core requirement for evaluating CoreAPI changes across versions.
- **Inline configuration in prose only:** Rejected — machine-readable config
  (`.fleet-testing.yaml`) is required for automation.

## Consequences

- Positive: Clear signal semantics prevent benchmark results from being
  misinterpreted as standalone correctness signals.
- Positive: Milestone KPI goals give an explicit, version-independent gate
  for release readiness.
- Positive: CTRF + OTLP hybrid keeps pass/fail and metrics concerns separate
  and uses purpose-built schemas for each.
- Positive: CLI-first execution makes pipelines auditable, reproducible, and
  agent-independent.
- Positive: `.fleet-testing.yaml` makes the test contract machine-readable
  and versionable.
- Negative: Comparative mode requires tooling to pair baseline and candidate
  runs and compute deltas — this must be built.
- Negative: Large-payload tests require GPU resources and scheduling discipline
  to avoid blocking CI.
- Negative: CTRF is newer than JUnit XML; ecosystem tooling is still maturing.
- Negative: `.fleet-testing.yaml` schema must be kept in sync with the ADR
  as the design evolves.

## Open Questions

- **Statistical significance for metric deltas**: Variance tolerance thresholds
  for benchmark and KPI comparisons are not yet defined. To be specified before
  the first comparative run.
- **Large-payload promotion criteria**: Conditions under which large-payload
  tests graduate from opt-in to gated are not yet defined. To be specified at
  milestone planning.
