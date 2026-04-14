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

- **Signal type (primary):** Pass/fail — a benchmark that fails to complete is a hard failure
- **Signal type (secondary):** Metrics — only meaningful relative to a baseline
- **Usable in:** Correctness mode (completion pass/fail only), Comparative mode (completion + metrics delta)

### 3. Mega Integration — Small Simulation Payload
End-to-end test driving the CoreAPI via Mega with a small, fast simulation workload.

- **Signal type (primary):** Simulation pass/fail
- **Signal type (secondary):** KPIs for comparative analysis
- **Milestone KPI goals:** Defined target thresholds treated as pass/fail against a known target
- **Usable in:** Correctness mode, Comparative mode, Milestone validation

### 4. Mega Integration — Large Simulation Payload
Same as above but with a large, production-representative simulation workload.

- **Signal type (primary):** Simulation pass/fail
- **Signal type (secondary):** KPIs for comparative analysis
- **Milestone KPI goals:** Same milestone threshold structure as small payload
- **Usable in:** Correctness mode, Comparative mode, Milestone validation
- **Note:** Opt-in — not run on every PR; triggered for milestone validation and on-demand

## Decision

We adopt the following rules:

1. **API tests gate every run** — cheap, self-contained correctness signals.
2. **API tests and Benchmark tests run in parallel** — independent tracks, concurrent execution.
3. **Benchmarks have two signal layers:**
   - *Completion pass/fail* — a benchmark that does not complete is a hard failure
   - *Metrics comparative* — requires a baseline run; only triggered in comparative mode
4. **Baseline runs are conditional** — only scheduled when the caller requests comparative mode.
5. **Mega integration tests have two verdict modes:**
   - *Simulation pass/fail* — usable standalone for correctness gating
   - *KPI comparative* — requires a baseline run; used in comparative mode and milestone validation
6. **Milestone KPI goals are encoded as explicit thresholds**, not derived from a baseline run.
7. **Large-payload tests are opt-in** — triggered for milestone validation and comparative analysis on demand.
8. **Test track configuration is declared in `.fleet-testing.yaml`** — the file in the repo root is the canonical definition of all tracks, their signal types, output schemas, and mode applicability. The ADR describes the design rationale; the YAML is the machine-readable contract.

## Structured Data Requirements

All test track outputs must be machine-readable structured data:

- **Pass/Fail Results:** CTRF (Common Test Report Format) JSON
- **KPI and Metrics:** OpenTelemetry Metrics (OTLP JSON)
- **Suite-Level Summary:** Versioned JSON envelope containing schema_version, mode, run_metadata, results (CTRF), metrics (OTLP), deltas (comparative mode), milestone_verdict (milestone mode)

## CLI-First Execution Requirement

Every discrete, repeatable operation must be expressible as a CLI invocation. Agents invoke CLI tools and interpret results; they do not replicate CLI logic. The `.fleet-testing.yaml` file defines which CLI commands implement each track.

## Alternatives Considered

- **Treat all tests as pass/fail:** Rejected — benchmark metrics have no intrinsic pass/fail meaning without context.
- **Single test mode (correctness only):** Rejected — comparative analysis is a core requirement.
- **Inline configuration in prose only:** Rejected — machine-readable config (`.fleet-testing.yaml`) is required for automation.

## Consequences

- Positive: Clear signal semantics prevent benchmark results from being misinterpreted.
- Positive: Milestone KPI goals give an explicit, version-independent gate.
- Positive: `.fleet-testing.yaml` makes the test contract machine-readable and versionable.
- Negative: Comparative mode requires tooling to pair baseline and candidate runs — must be built.
- Negative: Large-payload tests require GPU resources and scheduling discipline.
- Negative: `.fleet-testing.yaml` schema must be kept in sync with the ADR as the design evolves.
