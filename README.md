# Schema-Driven SaaS Configuration Replication

A case study in turning an under-documented SaaS settings API into a safe, repeatable workflow for comparing, replicating, verifying, and restoring account configurations.

> **Note:** This repository is intentionally sanitized. It contains the engineering case study, not the production tool or proprietary configuration data.

## At a glance

| | |
| --- | --- |
| **Challenge** | Replace a slow, error-prone manual account comparison process without treating every API field as safe to copy |
| **My contribution** | API investigation, field validation, configuration modeling, workflow design, safeguards, and test design |
| **Configuration mapped** | 188 settings: 179 approved for automation and 9 deliberately excluded |
| **Validation** | 97 passing offline tests plus live sandbox replication |
| **Core outcome** | A deterministic workflow with dependency planning, read-after-write verification, field-level audit logs, and run-specific Restore |

## Contents

- [The problem](#the-problem)
- [My role](#my-role)
- [The approach](#the-approach)
- [How the workflow operates](#how-the-workflow-operates)
- [Safety and failure handling](#safety-and-failure-handling)
- [Results](#results)
- [What I learned](#what-i-learned)
- [Repository scope](#repository-scope)
- [Key design decisions](docs/key-decisions.md)

## The problem

Recreating a customer or test environment for troubleshooting required manually comparing settings spread across several areas of a SaaS product.

The apparent solution was simple:

```text
GET settings from Account A
→ PUT settings into Account B
```

But the API did not provide a trustworthy configuration model. Before any setting could be copied safely, the workflow needed to know:

- whether the UI setting was exposed and writable through the API;
- which values were accepted and what they meant in the UI;
- whether another setting or account-level module was required;
- whether the field should be excluded from automation;
- whether a successful response actually changed the target; and
- how the target could be restored to its previous state.

The API existed. The reliable operating rules did not.

## My role

My primary contribution was the investigation and modeling that made the automation dependable. My work included:

- testing settings individually in sandbox accounts and comparing API behavior with the UI;
- mapping accepted values, dependencies, external prerequisites, and exclusions;
- designing a machine-readable configuration schema as an explicit allowlist;
- defining the compare, plan, replicate, verify, audit, and Restore workflows;
- adding identity, secret-handling, validation, and rollback safeguards;
- improving request efficiency without obscuring dependency-sensitive updates; and
- turning live-test findings into schema rules and automated test cases.

The implementation was built around that configuration model instead of scattering product-specific behavior throughout the application.

## The approach

### 1. Discover the real API behavior

Each candidate setting was tested against a sandbox account and checked in the product UI. This established the API parameter, accepted values, UI meaning, write behavior, and prerequisites for each field.

That investigation produced a registry of:

- **188 total settings**
- **179 active settings** approved for automation
- **9 ignored settings** deliberately excluded

### 2. Encode the knowledge as data

The validated behavior was captured in a YAML schema. A simplified public example is shown below:

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

The schema acts as an allowlist: the workflow operates only on reviewed fields marked `active`. Dependencies, warnings, and exceptional read formats are modeled alongside the field rather than hard-coded into the execution engine.

### 3. Plan before changing anything

Before mutation, the workflow:

1. validates the complete schema;
2. resolves source and target identities from their credentials;
3. blocks execution if both credentials identify the same account;
4. reads and normalizes both configurations;
5. validates source values against the schema;
6. calculates only the changes needed; and
7. determines prerequisite order, warnings, and safe batching.

### 4. Verify and preserve evidence

After each mutation, the target is read again. The returned state—not the HTTP status alone—determines whether each field succeeded.

Every run records:

- target state before and after replication;
- source values used for the run;
- field-level outcomes and explanations;
- resolved account identity, without credentials; and
- a run-specific snapshot that can restore the target later.

## How the workflow operates

```text
Resolve account identities
          ↓
Validate schema and block same-account runs
          ↓
Read source and target settings
          ↓
Normalize and validate known fields
          ↓
Compare desired and current state
          ↓
Plan dependencies, warnings, and safe batches
          ↓
Apply changes and read the target again
          ↓
Record field-level results and snapshots
          ↓
Offer Restore from this run's pre-change state
```

Independent settings are combined into a bulk request. Settings involved in active dependency relationships remain explicit and ordered:

```text
update prerequisite → verify prerequisite
                    → update dependent → verify dependent
```

This reduces unnecessary API calls while keeping risky changes observable.

## Safety and failure handling

The workflow is designed to fail narrowly and explainably.

| Situation | Behavior |
| --- | --- |
| Malformed schema or dependency cycle | Stop before contacting or modifying the target |
| Source and target resolve to the same account | Block the run |
| Unsupported source value | Mark that field `INVALID_SOURCE_VALUE`; continue with other valid fields |
| Unmet prerequisite | Mark the dependent field `SKIPPED_DEPENDENCY` |
| External module may be required | Show a manual warning without misclassifying a successful update |
| API accepts a request but state does not match | Mark the field `VERIFICATION_FAILED` |
| Transient API error | Retry with bounded backoff |
| Permanent API error | Fail without unlimited retries |
| Restore requested for a different target | Block Restore after rechecking account identity |

Credentials are redacted from logs and excluded from snapshots and run metadata. User-facing network errors are sanitized as well.

## Results

The project replaced an informal, manual process with a controlled workflow capable of:

- comparing only reviewed and supported settings;
- applying independent changes efficiently;
- respecting prerequisite order for replication and Restore;
- isolating invalid values instead of terminating the whole run;
- verifying the resulting state field by field;
- producing before/after snapshots and an audit trail; and
- restoring the target from a specific historical run.

The completed implementation reached **97 passing offline tests**, covering schema validation, value handling, execution planning, verification, identity checks, secret redaction, Restore behavior, and UI state. Live sandbox runs were used to validate the end-to-end workflow and turn unexpected API responses into regression tests.

The most durable outcome was the configuration model: product knowledge that had existed through manual testing and individual familiarity became explicit, reviewable, machine-readable rules.

## What I learned

The difficult part was not sending an API request; it was establishing when that request was safe.

Three lessons shaped the final design:

1. **Configuration knowledge is part of the system.** Field meanings, prerequisites, and exclusions must be modeled and validated like code.
2. **Transport success is not business success.** A `200` response is insufficient when the target may ignore or reinterpret a value.
3. **Rollback needs provenance.** A Restore point must belong to a specific run and target account, not to a replaceable global backup.

Live testing reinforced the need for explicit handling of irregular data. For example, some logically scalar settings were returned as one-element lists. Rather than flattening every list and risking data loss, only identified fields received schema-defined normalization; unknown multi-value shapes remained invalid and visible.

For the reasoning, alternatives, and tradeoffs behind these choices, see [Key design decisions](docs/key-decisions.md).

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

It intentionally excludes production source code, the real configuration registry, API endpoints and field names, customer data, credentials, run artifacts, logs, and branded screenshots. All examples are synthetic and demonstrate the engineering approach without exposing proprietary information.

## Further reading

- [Key design decisions](docs/key-decisions.md) — detailed rationale, alternatives, and tradeoffs for the architecture
