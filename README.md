# Schema-Driven SaaS Configuration Replication

A case study in turning an under-documented settings API into a safe workflow for reproducing SaaS account configurations—and giving troubleshooting time back to the people investigating the actual issue.

> **Success metric:** Shifted up to **~1 engineer-hour per investigation** from manual configuration adjustment to diagnosis, while expanding repeatable coverage from roughly **10–20 manually selected fields to 179 validated fields**—about **9–18× broader coverage**.

*The time baseline is an experience-based estimate from real investigations, not a controlled benchmark. The coverage comparison uses the 179 active schema fields against the 10–20 fields typically aligned by hand.*

> **Repository note:** This case study is intentionally sanitized. It does not contain the production tool, proprietary configuration registry, credentials, or customer data.

## At a glance

| | |
| --- | --- |
| **Business problem** | Manual setup consumed about half of a typical two-hour investigation and still produced incomplete environment parity |
| **My contribution** | API investigation, field validation, configuration modeling, workflow architecture, safeguards, and test design |
| **Configuration mapped** | 188 settings: 179 approved for automation and 9 deliberately excluded |
| **What shipped** | A desktop workflow for compare, plan, replicate, verify, audit, and Restore |
| **Evidence** | 97 passing offline tests plus successful live sandbox replication |

## Contents

- [Business problem](#business-problem)
- [My contribution](#my-contribution)
- [Solution](#solution)
- [How it works](#how-it-works)
- [Evidence and outcomes](#evidence-and-outcomes)
- [Lessons](#lessons)
- [Repository scope](#repository-scope)
- [Technical decision record](docs/key-decisions.md)

## Business problem

Reproducing a customer or test environment for troubleshooting required manually comparing settings spread across several areas of a SaaS product.

From my direct experience, an issue investigation often took at least two hours, with roughly one hour spent adjusting just 10–20 settings believed to be relevant. That was expensive and still did not establish full configuration parity across a surface of nearly 200 mapped settings.

Missing one dependency or unexpected difference could create an inaccurate reproduction environment. The investigation could then follow a false lead—or reach engineering as an apparent product defect when configuration drift was the real cause.

The apparent automation was simple:

```text
GET settings from Account A
→ PUT settings into Account B
```

But a trustworthy workflow first needed to determine:

- which UI settings were writable through the API;
- which values were accepted and what they meant;
- which settings depended on other settings or external modules;
- which fields should not be automated;
- whether a successful response actually changed the target; and
- how to restore the target safely.

The API existed. The reliable operating rules did not.

## My contribution

My primary contribution was the investigation and configuration modeling that made the automation dependable. I:

- tested settings individually in sandbox accounts and compared API behavior with the UI;
- mapped accepted values, dependencies, external prerequisites, and exclusions;
- designed a machine-readable schema as an explicit automation allowlist;
- defined the comparison, planning, replication, verification, audit, and Restore workflows;
- added account-identity, secret-handling, validation, and rollback safeguards;
- reduced unnecessary API requests without hiding dependency-sensitive updates; and
- converted live-test findings into schema rules and regression tests.

The implementation was built around the resulting configuration model instead of scattering product-specific conditions throughout the application.

## Solution

### 1. Discover the real behavior

Each candidate setting was tested in a sandbox and checked against the product UI. This established its API parameter, accepted values, UI meaning, write behavior, and prerequisites.

The investigation produced a registry of **188 settings**: **179 active** fields approved for automation and **9 ignored** fields deliberately excluded.

### 2. Turn product knowledge into data

The validated behavior was captured in YAML. A simplified public example looks like this:

```yaml
- field_name: Example Feature
  ui_section: Scheduling
  data_type: int
  api_parameter: example_feature
  value_mapping:
    0: OFF
    1: ON
  status: active
```

The schema is an allowlist. The workflow operates only on reviewed fields marked `active`; dependencies, warnings, and exceptional read formats are defined alongside each field.

### 3. Build an execution plan before mutation

Before changing the target, the workflow validates the full schema, resolves both account identities, reads and normalizes their configurations, rejects unknown source values, calculates the desired-state difference, and plans prerequisite order and safe batching.

### 4. Verify, audit, and preserve a Restore point

Every mutation is followed by a fresh read of the target. The returned state—not the HTTP status alone—determines each field's result.

Each run stores before-and-after snapshots, field-level outcomes, resolved target identity without credentials, and a run-specific pre-change snapshot for Restore.

## How it works

```text
Resolve source and target identities
                ↓
Validate schema and block same-account runs
                ↓
Read, normalize, and compare configurations
                ↓
Plan dependencies, warnings, and safe batches
                ↓
Apply changes and read the target again
                ↓
Record field-level results and snapshots
                ↓
Restore later from this run's pre-change state
```

Independent settings are combined into a bulk request. Dependency-sensitive settings stay explicit and ordered:

```text
update prerequisite → verify prerequisite
                    → update dependent → verify dependent
```

The workflow fails narrowly and visibly:

- malformed schemas and same-account runs are blocked before mutation;
- unsupported values and unmet dependencies are skipped at field level;
- successful API responses that do not produce the requested state fail verification;
- external module requirements remain visible as manual warnings; and
- credentials are redacted and excluded from operational artifacts.

For the deeper rationale, alternatives, and tradeoffs, see the [technical decision record](docs/key-decisions.md).

## Evidence and outcomes

The completed implementation reached **97 passing offline tests** across schema validation, value handling, execution planning, verification, identity checks, secret redaction, Restore behavior, and UI state.

Live sandbox testing validated the end-to-end workflow and exposed response shapes that initial field mapping had missed. Those findings were modeled explicitly and added as regression tests rather than handled with broad assumptions.

### Successful sandbox run

![The desktop replication tool after a completed run. The summary reports 30 successful updates, 137 settings already matched, no failed updates or verification failures, and clearly identifies skipped dependencies, invalid source values, ignored fields, and manual warnings.](docs/assets/replication-success.png)

*A completed sandbox run: 30 settings updated, 137 already matched, and no failed updates or verification failures. Unsafe or unsupported fields remained visible instead of being silently forced.*

The workflow delivered four practical outcomes:

- **More time for diagnosis:** up to roughly one hour of a typical investigation could be redirected from configuration adjustment to the underlying issue.
- **Broader reproduction coverage:** 179 validated fields could be considered consistently instead of manually selecting about 10–20 likely candidates.
- **Fewer false leads:** verified configuration parity reduced the chance that hidden drift would be mistaken for a product defect and passed downstream to engineering.
- **Safer operations:** explicit exclusions, dependency planning, audit logs, and run-specific Restore made automation reversible and explainable.

The most durable result was the configuration model itself: knowledge that had lived in manual testing and individual familiarity became explicit, reviewable, machine-readable rules.

## Lessons

The difficult part was not sending an API request; it was establishing when that request was safe.

1. **Configuration knowledge is part of the system.** Field meanings, prerequisites, and exclusions need the same rigor as application code.
2. **Transport success is not business success.** A `200` response is insufficient when the target may ignore or reinterpret a value.
3. **Coverage changes troubleshooting quality.** Matching only the suspected fields is faster than matching everything manually, but it can preserve the very difference causing the issue.
4. **Rollback needs provenance.** A Restore point must belong to a specific run and target account, not a replaceable global backup.

## Skills demonstrated

- SaaS API investigation and behavioral validation
- Configuration and schema design
- Dependency and desired-state modeling
- Failure-mode analysis and operational safeguards
- Verification, rollback, and audit design
- Python, YAML, REST APIs, and automated testing
- Translating domain knowledge into deterministic automation

## Repository scope

This repository is a **case study**, not a distributable copy of the internal application.

It intentionally excludes production source code, the real configuration registry, API endpoints and field names, customer data, credentials, run artifacts, logs, and branded product screenshots. All examples are synthetic and demonstrate the engineering approach without exposing proprietary information.

## Further reading

- [Key design decisions](docs/key-decisions.md) — detailed context, rationale, alternatives, and tradeoffs behind the architecture
