# coreapi-integration

Integration test suite for the CoreAPI service.

## Overview

This repository houses integration tests for CoreAPI, organized into four test tracks:

| Track | Signal | Mode |
|-------|--------|------|
| API Tests | Pass/fail | Correctness, Comparative |
| Benchmark Tests | Pass/fail + metrics | Correctness (completion only), Comparative |
| Mega Integration — Small Payload | Simulation pass/fail + KPIs | Correctness, Comparative, Milestone |
| Mega Integration — Large Payload | Simulation pass/fail + KPIs | Correctness, Comparative, Milestone |

## Test Modes

- **Correctness** — validates behavior at a single software version
- **Comparative** — evaluates changes between two software versions (metrics delta)
- **Milestone** — validates KPIs against explicit threshold targets

## Output Format

All test tracks emit structured output:
- **Pass/fail results:** [CTRF JSON](https://ctrf.io/)
- **KPI metrics:** [OpenTelemetry OTLP JSON](https://opentelemetry.io/docs/specs/otel/metrics/)
- **Suite summary:** versioned JSON envelope

## Architecture Decision Records

See [`adrs/`](adrs/) for architectural decisions governing this project.
