# Key Design Decisions

This document is the technical companion to the [case-study overview](../README.md). It records the reasoning behind the workflow's most important architectural and operational choices.

The examples are synthetic. Product names, real API parameters, credentials, and proprietary configuration mappings are intentionally excluded.

## Decision summary

| Decision | Why it matters |
| --- | --- |
| Use a reviewed schema as an allowlist | API availability does not imply that a field is understood or safe to automate |
| Keep product rules in data | New field behavior can be reviewed without adding branches to the execution engine |
| Model dependencies separately from warnings | Controllable prerequisites affect execution; external requirements require human confirmation |
| Validate the entire schema before mutation | A broken rule set must fail before it can change an account |
| Resolve account identity from credentials | Prevents accidental same-account replication and protects Restore |
| Compare desired state before writing | Avoids unnecessary mutations and creates an explicit execution plan |
| Batch only independent changes | Reduces API traffic without hiding prerequisite-sensitive operations |
| Verify by reading back each result | HTTP success does not prove configuration success |
| Normalize only explicitly identified fields | Avoids silently corrupting unfamiliar response shapes |
| Isolate invalid values at field level | One unusual value should not cancel every safe change |
| Keep a Restore point per run | Preserves rollback provenance across multiple replications |
| Record field-level outcomes | Optimization must not reduce traceability |

## 1. Use a reviewed schema as an allowlist

**Context:** The settings endpoint exposed fields with uneven documentation and behavior. Some were writable, some were not fully understood, and some should not be automated.

**Decision:** Only fields explicitly reviewed and marked `active` in the configuration schema are eligible for comparison or mutation. Known but unsuitable fields remain documented as `ignored`.

```yaml
- field_name: Example Feature
  api_parameter: example_feature
  data_type: int
  value_mapping:
    0: OFF
    1: ON
  status: active
```

**Why:** Treating every field returned by the API as safe would make undocumented behavior an implicit part of the automation. An allowlist turns incomplete knowledge into a visible boundary.

**Tradeoff:** New settings require investigation and a schema update before they can be replicated. That delay is intentional.

## 2. Keep product rules in data

**Context:** Field-specific branches would couple the execution engine to every product rule:

```python
if field == "child_setting":
    enable_parent_setting()
```

**Decision:** Store value meanings, dependencies, warnings, and read normalization in the schema. Keep the engine limited to generic concepts such as `active`, `ignored`, `requires`, `requires_any`, and `warnings`.

```yaml
dependencies:
  - type: requires
    when_source_values: [1]
    api_parameter: parent_setting
    required_values: [1]
    on_unmet: skip
```

**Why:** Product knowledge becomes reviewable, testable, and extensible without accumulating field-name conditionals throughout the application.

**Tradeoff:** The schema becomes executable policy and therefore requires strict validation.

## 3. Distinguish dependencies from external warnings

**Context:** Some prerequisites are settings the workflow can inspect or change. Others are paid modules, legacy capabilities, or external integrations outside the settings API.

**Decision:** Represent API-manageable prerequisites as dependencies and non-manageable requirements as warnings.

Dependencies control execution:

```text
prerequisite satisfied → continue
prerequisite can be enabled safely → enable, verify, continue
otherwise → SKIPPED_DEPENDENCY
```

Warnings inform the operator:

```yaml
warnings:
  - type: manual_confirmation
    when_source_values: [1]
    module: Example Module
    action: warn_only
    message: Confirm that Example Module is enabled on the target account.
```

**Why:** The application should not pretend it can prove or control an external account capability. At the same time, a warning should not turn a verified update into a false failure.

### Multiple valid prerequisites

Some settings are valid when any one of several prerequisites is already enabled. These use `requires_any`:

```yaml
dependencies:
  - type: requires_any
    when_source_values: [1]
    conditions:
      - api_parameter: visibility_option_a
        required_values: [1]
      - api_parameter: visibility_option_b
        required_values: [1]
    on_unmet: skip
```

The planner accepts any already-satisfied condition. It does not arbitrarily choose and enable one because doing so could change product behavior beyond the requested replication.

## 4. Validate the complete schema before mutation

**Context:** Once the schema determines execution, malformed YAML is an operational safety issue rather than a formatting issue.

**Decision:** Validate the entire model before account lookup or settings mutation. Checks include:

- required structure and field properties;
- supported data types and statuses;
- mappings for active fields;
- duplicate API parameters;
- dependency and warning structure;
- references to missing or ignored prerequisites;
- dependency values absent from their field mappings;
- self-dependencies and dependency cycles; and
- supported read-normalization rules.

**Why:** A partial validation could allow a run to make early changes before discovering that a later rule is invalid. Full validation makes the plan fail closed.

## 5. Resolve and bind account identity

**Context:** Manually entered account IDs can be mistyped, and credentials for the same account can be mistakenly supplied as both source and target.

**Decision:** Resolve company identity from each access token and block the run if both tokens identify the same account. Bind every historical run to the resolved target identity and check it again before Restore.

```text
source token → source company ┐
                              ├→ same identity? → BLOCK
target token → target company ┘
```

Credentials are redacted from logs and excluded from metadata and snapshots. Network errors are sanitized before display.

**Why:** Identity should be proven by the authenticated service, not transcribed by the operator. Binding Restore prevents a valid snapshot from being applied to the wrong account.

## 6. Compare desired state before writing

**Context:** Copying every active setting would create unnecessary requests, obscure the actual change set, and increase the chance of side effects.

**Decision:** Read source and target state, normalize known exceptions, validate values, and compute the difference before building an execution plan.

Fields that already match are recorded as `ALREADY_MATCHED`; only required changes reach the mutation phase.

**Why:** Desired-state comparison reduces work and makes the intended outcome inspectable before execution.

## 7. Batch only independent changes

**Context:** One PUT request per field was straightforward but expensive. A single request for every change was efficient but made dependency handling and failure attribution unsafe.

**Decision:** Combine independent settings into one request and keep active dependency chains individually ordered and verified.

```text
Independent A ┐
Independent B ├→ one PUT → one GET → field-level results
Independent C ┘

Parent → PUT → verify → Child → PUT → verify
```

An otherwise independent setting is removed from the bulk request when it acts as a prerequisite for another changed field in that run.

**Why:** This preserves the observability of dependency-sensitive work while reducing ordinary request volume.

**Tradeoff:** Planning is more complex than either all-individual or all-bulk execution.

## 8. Verify with read-after-write

**Context:** An API may return a successful status while ignoring, coercing, or partially applying a setting.

**Decision:** Read the target configuration after mutation and compare each returned value with the intended value.

```text
requested: A=1, B=0, C=14
returned:  A=1, B=1, C=14

A → SUCCESS
B → VERIFICATION_FAILED
C → SUCCESS
```

**Why:** Transport success and configuration success are different claims. Only the resulting state can establish the latter.

## 9. Normalize only known response exceptions

**Context:** Live testing found settings that were logically scalar but returned as a one-element list, while an unset state could appear as an empty list.

**Decision:** Add normalization only to fields explicitly confirmed to have that response shape:

```yaml
read_normalization:
  type: single_value_list
  empty_value: "  "
```

```text
["45"] → "45" → normal validation
[]     → configured empty value → normal validation
[0, 6] → INVALID_SOURCE_VALUE
```

**Why:** Flattening lists globally would confuse unusual scalar serialization with genuine multi-value data. Unknown shapes remain visible rather than being guessed at.

## 10. Isolate invalid values at field level

**Context:** A long-running replication may encounter one source value that is new, malformed, or absent from the validated mapping.

**Decision:** Mark that field `INVALID_SOURCE_VALUE`, skip its mutation, and continue with other safe fields.

**Why:** The unknown field must not be copied, but it also should not prevent unrelated, validated work. The UI and audit log highlight the anomaly for follow-up.

The same field-level reporting is used for:

- `SUCCESS`
- `ALREADY_MATCHED`
- `FAILED`
- `SKIPPED_DEPENDENCY`
- `INVALID_SOURCE_VALUE`
- `VERIFICATION_FAILED`

## 11. Keep a Restore point for every run

**Context:** A single global backup would be overwritten by later replications and would lose the relationship between a change, its target, and its rollback state.

**Decision:** Give each run its own dedicated history directory:

```text
runs/
└── <run_id>/
    ├── metadata.yaml
    ├── source_snapshot.xlsx
    ├── target_before.xlsx
    ├── target_after.xlsx
    └── replication_log.xlsx
```

Restore output is stored separately under that run. The target identity is resolved again before applying the snapshot.

**Why:** Rollback becomes attributable and repeatable instead of depending on whichever backup happened most recently.

### Reverse dependency order when restoring

Replication and Restore do not always use the same order. If a child can be changed only while its parent is on, but the original target had the parent off:

```text
Replication: parent ON → child
Restore:     child → parent OFF
```

Restoring the parent first could remove the condition needed to restore the child. The Restore planner therefore reverses relevant dependency order when necessary.

## 12. Preserve field-level auditability

**Context:** Batching improves efficiency but can collapse several business outcomes into one transport response.

**Decision:** Record an audit result for every field regardless of how the API requests were grouped. Representative data includes:

```text
field_name
api_parameter
source_value
source_meaning
target_value_before
target_value_after
status
detail
timestamp
```

**Why:** Request-level logging answers what the client sent. Field-level logging answers what happened to the configuration, which is the operational question that matters.

## Resulting principles

The decisions above reduce to a small set of reusable principles:

1. Treat configuration knowledge as versioned, validated data.
2. Fail before mutation when the execution model is invalid.
3. Do not infer behavior for unknown values or prerequisites.
4. Separate warnings from machine-verifiable execution state.
5. Verify the resulting state rather than trusting a response code.
6. Optimize routine work without hiding risky operations.
7. Keep failures local when unrelated work can continue safely.
8. Keep credentials out of operational artifacts.
9. Retry only errors that may be transient, with a bound.
10. Bind rollback state to the run and account that produced it.
