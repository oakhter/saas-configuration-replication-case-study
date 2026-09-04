# Schema-Driven SaaS Configuration Replication

A case study in turning an under-documented SaaS configuration surface into a safe, deterministic workflow for comparing, replicating, verifying, and restoring account settings.

> This repository is intentionally sanitized. Product names, proprietary API parameters, real account data, credentials, internal screenshots, and production configuration mappings are excluded or replaced with synthetic examples.

---

## Problem

Recreating a customer or test environment for troubleshooting often meant manually comparing a large number of account settings across multiple areas of a SaaS product.

At first glance, the automation problem looked simple:

```text
GET settings from Account A
→
PUT settings into Account B
```

In practice, that was not enough.

Before replication could be automated safely, I needed to understand:

- which UI settings were actually exposed through the API;
- which API parameter controlled each setting;
- which values were accepted;
- what those values meant in the UI;
- which settings depended on other settings;
- which features depended on account-level modules outside the settings API;
- which fields should not be automated;
- how to confirm that a successful API response actually changed the setting;
- how to restore the target environment if the replication needed to be reversed.

The API existed.

The usable configuration model did not.

---

## My Role

My main contribution was the investigation and modeling required to make the automation trustworthy.

I:

- systematically tested configuration fields in sandbox environments;
- mapped UI settings to their API parameters;
- validated accepted values and their UI meanings;
- identified dependency relationships between settings;
- separated API-manageable dependencies from external/manual prerequisites;
- identified fields that should be excluded from automation;
- structured that knowledge into a machine-readable schema;
- designed the replication, verification, logging, and Restore workflow;
- added safeguards around account identity and rollback;
- identified opportunities to reduce unnecessary API requests;
- used live testing to uncover and model unexpected API response shapes.

The final implementation was built around this configuration model rather than embedding product-specific rules directly into application logic.

---

## Discovery and Validation

The first phase was manual.

Each candidate setting was tested individually while its API behavior was compared against the UI.

That process produced a validated configuration registry containing:

- **188 total settings**
- **179 active settings**
- **9 deliberately excluded settings**

A simplified public example looks like:

```yaml
- field_name: Example Feature
  ui_section: Scheduling
  description: Controls the Example Feature option.
  data_type: int
  api_parameter: example_feature
  value_mapping:
    0: OFF
    1: ON
  status: active
```

The schema became an explicit allowlist.

The application only operates on fields that have been reviewed and marked:

```yaml
status: active
```

Fields that should not be automated remain:

```yaml
status: ignored
```

This avoids treating every setting returned by the API as automatically safe to modify.

---

## Building a Machine-Readable Configuration Model

A key design decision was to keep product knowledge in the schema and execution behavior in the application.

Instead of writing field-specific logic such as:

```python
if field == "child_setting":
    enable_parent_setting()
```

the dependency is represented in the schema:

```yaml
dependencies:
  - type: requires
    when_source_values:
      - 1
    api_parameter: parent_setting
    required_values:
      - 1
    on_unmet: skip
```

The execution engine only needs to understand generic concepts such as:

```text
active
ignored
requires
requires_any
warnings
```

It does not need hard-coded knowledge of every individual setting.

This made the behavior easier to reason about and extend as new settings were validated.

---

## Dependency Handling

Some settings cannot be updated unless another setting is already enabled.

For example:

```text
Parent Feature = ON
        ↓
Child Option can be changed
```

The workflow checks those relationships before mutation.

Conceptually:

```text
Check prerequisite
        ↓
Already satisfied?
   ↙           ↘
 yes            no
  ↓              ↓
continue    Can it safely
            be enabled?
                ↓
             update
                ↓
             verify
                ↓
         update dependent
```

If a prerequisite cannot be satisfied, the dependent setting is not attempted.

It is recorded as:

```text
SKIPPED_DEPENDENCY
```

---

## Multiple Valid Prerequisites

Some fields are valid when any one of several prerequisites is enabled.

These are modeled using `requires_any`.

Example:

```yaml
dependencies:
  - type: requires_any
    when_source_values:
      - 1
    conditions:
      - api_parameter: visibility_option_a
        required_values:
          - 1
      - api_parameter: visibility_option_b
        required_values:
          - 1
    on_unmet: skip
```

The workflow checks whether any valid prerequisite is already satisfied.

It does not arbitrarily enable one.

---

## External Requirements vs. Dependencies

Not every prerequisite can be controlled through the same settings API.

Some settings depend on:

- account-level modules;
- paid features;
- legacy capabilities;
- external integrations.

These are modeled separately as warnings.

Example:

```yaml
warnings:
  - type: manual_confirmation
    when_source_values:
      - 1
    module: Example Module
    action: warn_only
    message: Confirm that Example Module is enabled on the target account.
```

This distinction matters.

A dependency can control execution.

A warning cannot.

A field may therefore finish as:

```text
SUCCESS
+
manual warning
```

without being misclassified as a failed update.

---

## Strict Schema Validation

Once the schema became the source of truth, malformed YAML became a safety risk.

The application validates the full schema before performing account lookup or settings mutation.

Validation includes:

- required top-level structure;
- required field properties;
- supported data types;
- valid field statuses;
- non-empty value mappings for active fields;
- duplicate API parameters;
- valid dependency references;
- prerequisite fields being active;
- dependency values existing in their mappings;
- self-dependencies;
- dependency cycles;
- warning structure;
- supported read-normalization rules.

For example:

```text
A requires B
B requires A
```

is rejected before any account is modified.

A malformed configuration model fails closed.

---

## Account Identity Safety

The workflow does not rely on users manually typing account IDs.

Instead:

```text
Source Token
    ↓
Resolve Source Company

Target Token
    ↓
Resolve Target Company
```

If both credentials resolve to the same account:

```text
BLOCK
```

This protects against accidentally using the same environment as both source and target.

The resolved target identity is also stored with each historical run and checked again before Restore.

Access tokens are treated as secrets:

- they are redacted from logs;
- they are not stored in run metadata;
- they are not stored in snapshots;
- network errors are sanitized before being shown to the user.

---

## Reducing API Calls

The initial implementation used one PUT request per changed setting.

That was safe, but unnecessarily expensive.

The final execution strategy separates changes into two categories.

### Bulk-Safe Settings

Independent settings are combined into one request:

```text
Setting A
Setting B
Setting C
Setting D
    ↓
ONE PUT
    ↓
ONE GET verification
```

### Dependency-Controlled Settings

Fields participating in active dependency relationships remain individually processed.

```text
PUT prerequisite
→
verify prerequisite
→
PUT dependent
→
verify dependent
```

An otherwise independent field is also removed from the bulk request if it is acting as a prerequisite for another changed setting during that run.

This reduces request volume without hiding dependency-sensitive behavior inside a large batch.

Transient API failures use bounded retry/backoff behavior for cases such as:

```text
429
500
502
503
504
```

Permanent errors fail without unlimited retries.

---

## Verification

A successful HTTP response is not treated as proof that a setting changed correctly.

After mutation, the application reads the target configuration again.

Example:

```text
Requested:

A = 1
B = 0
C = 14
```

The API may return:

```text
HTTP 200
```

but the verification read could show:

```text
A = 1
B = 1
C = 14
```

The final result is therefore:

```text
A → SUCCESS
B → VERIFICATION_FAILED
C → SUCCESS
```

Even when several settings are updated in one request, their final statuses remain field-level.

---

## What Live Testing Changed

Live testing exposed behavior that was not obvious during the initial field-by-field mapping.

Some settings that logically represent a single scalar value were returned by the API as a one-element list:

```python
["45"]
```

while an unset state could appear as:

```python
[]
```

It would have been unsafe to flatten all lists globally because other fields may genuinely contain multiple values.

Instead, only explicitly identified scalar fields use schema-defined normalization:

```yaml
read_normalization:
  type: single_value_list
```

or:

```yaml
read_normalization:
  type: single_value_list
  empty_value: "  "
```

The result:

```text
["45"]
→
"45"
→
validate normally
```

and:

```text
[]
→
configured empty state
→
validate normally
```

But an unmodeled multi-value result such as:

```python
[0, 6]
```

still becomes:

```text
INVALID_SOURCE_VALUE
```

rather than being guessed at.

This preserved the distinction between:

```text
one scalar serialized strangely
```

and:

```text
a true multi-value field
```

---

## Invalid Values Fail at the Field Level

Unexpected source values do not crash the entire run.

Instead:

```text
unexpected value
→
INVALID_SOURCE_VALUE
→
skip that field
→
continue remaining settings
```

Invalid values are also surfaced clearly in the UI rather than disappearing inside a long log.

The application highlights:

- invalid source values;
- failed updates;
- verification failures;
- dependency skips;
- manual warnings.

This makes abnormal outcomes visible without treating every warning as a failed execution.

---

## Backup and Restore

Every replication creates its own historical run.

Conceptually:

```text
runs/
└── <run_id>/
    ├── metadata.yaml
    ├── source_snapshot.xlsx
    ├── target_before.xlsx
    ├── target_after.xlsx
    └── replication_log.xlsx
```

The key file is:

```text
target_before.xlsx
```

which represents the target environment before mutation.

Restore uses the snapshot from a specific historical run rather than a single global backup.

Restore output is stored separately:

```text
restore_<timestamp>/
├── metadata.yaml
├── target_before_restore.xlsx
├── target_after_restore.xlsx
└── restore_log.xlsx
```

This prevents a later replication from silently replacing an earlier rollback point.

---

## Restore Dependency Ordering

Restore ordering is not always the same as replication ordering.

Suppose:

```text
Child Setting requires Parent = ON
```

Replication may need:

```text
Parent ON
→
Child
```

But if the original target had:

```text
Parent = OFF
```

Restore may require:

```text
restore Child
→
Parent OFF
```

Otherwise the prerequisite could be disabled before the dependent setting is restored.

The Restore workflow therefore accounts for dependency reversal when necessary.

---

## Audit Trail

Each setting receives its own audit record.

Representative fields include:

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

Possible statuses include:

```text
SUCCESS
ALREADY_MATCHED
FAILED
SKIPPED_DEPENDENCY
INVALID_SOURCE_VALUE
VERIFICATION_FAILED
```

Bulk execution reduces API calls without reducing field-level traceability.

---

## UI Workflow

The internal tool uses a lightweight desktop interface.

The workflow is:

```text
Source Access Token
Target Access Token

Validate Accounts
Run Replication
Restore Target
New Run
```

Resolved account names and IDs are displayed as read-only information.

`New Run` resets the current session without deleting saved historical runs or Restore data.

The completion summary separates:

- successful updates;
- already-matching settings;
- failures;
- skipped dependencies;
- invalid values;
- verification failures;
- schema-ignored fields;
- manual warnings.

Warnings and invalid/error states are visually highlighted.

---

## Testing

The completed implementation reached:

**97 passing offline tests**

The test suite covers:

### Schema Validation

- required schema structure;
- invalid statuses;
- duplicate API parameters;
- malformed dependencies;
- invalid prerequisite references;
- ignored prerequisites;
- self-dependencies;
- dependency cycles;
- malformed warnings;
- read-normalization validation.

### Value Handling

- scalar value matching;
- numeric/string normalization;
- unexpected lists and dictionaries;
- explicit single-value-list normalization;
- empty-list normalization;
- invalid multi-value behavior.

### Execution

- bulk-safe updates;
- dependency-controlled updates;
- prerequisite extraction from bulk execution;
- runtime dependency failures;
- verification failures;
- invalid values remaining non-fatal.

### Identity and Security

- account identity parsing;
- same-account blocking;
- Restore identity validation;
- token redaction;
- metadata containing no secrets.

### Restore

- run-specific Restore;
- Restore without the original source token;
- dependency reversal;
- bulk-safe Restore execution;
- malformed historical run handling.

### UI

- New Run state reset;
- warning highlighting;
- invalid-value highlighting;
- summary formatting.

The workflow was also exercised through live sandbox replication, with issues discovered during live testing feeding back into the schema and runtime safeguards.

---

## Final Architecture

```text
Source Token ──→ Resolve Source Identity
                         │
                         │
Target Token ──→ Resolve Target Identity
                         │
                         ▼
                 Same-account check
                         │
                         ▼
                GET source settings
                GET target settings
                         │
                         ▼
                Schema filtering
                         │
                         ▼
                Read normalization
                         │
                         ▼
                 Value validation
                         │
                         ▼
              Desired-state comparison
                         │
                         ▼
             Dependency + warning plan
                   │             │
                   │             │
                   ▼             ▼
             Bulk-safe       Dependency-
              settings        controlled
                   │             │
               ONE PUT        Individual
               ONE GET        PUT + GET
                   │             │
                   └──────┬──────┘
                          ▼
                  Field verification
                          │
                          ▼
               Before/after snapshots
                          │
                          ▼
                    Audit log
                          │
                          ▼
               Run-specific Restore
```

---

## Design Principles

### 1. Treat configuration knowledge as data

Product-specific behavior belongs in the schema rather than being scattered throughout application code.

### 2. Validate before mutation

Malformed configuration should fail before contacting the target account.

### 3. Do not guess dependency behavior

If a prerequisite cannot be deterministically satisfied, skip the dependent setting.

### 4. Keep warnings separate from execution state

An external requirement should not automatically turn a successful setting update into a failure.

### 5. Do not trust HTTP success alone

Read the resulting state back and verify it.

### 6. Optimize ordinary changes without hiding risky ones

Independent settings can be batched.

Dependency-sensitive settings remain explicit.

### 7. Unexpected data should fail locally

One unusual source value should not terminate an entire replication.

### 8. Secrets should not become operational artifacts

Credentials do not belong in logs, snapshots, metadata, or user-facing network errors.

### 9. Retry transient failures, not permanent ones

Retries are bounded and reserved for errors that may actually succeed on another attempt.

### 10. Rollback must belong to a specific run

A single global backup is not sufficient when multiple replications can occur.

---

## What Made This Difficult

The difficult part was not making an API request.

It was determining what could safely be automated.

That required answering questions such as:

- Is this field actually writable?
- What values does it accept?
- What does each value mean?
- Does the API state actually correspond to the UI state?
- Does this setting require another setting first?
- Can that prerequisite itself be automated?
- Does the setting depend on an external module?
- What happens when the API returns an unexpected type?
- How do we know the target actually changed?
- How do we safely undo the change?

The reliability of the final workflow came primarily from answering those questions before treating the endpoint as a simple automation surface.

---

## Result

The project converted a largely manual configuration-comparison process into a controlled workflow capable of:

- resolving source and target account identity;
- comparing validated settings;
- filtering configuration through an explicit allowlist;
- excluding unsupported fields;
- applying independent changes efficiently;
- respecting prerequisite ordering;
- surfacing external requirements;
- detecting invalid API values without crashing the run;
- verifying resulting account state;
- generating before/after snapshots;
- producing field-level audit logs;
- restoring the target from a specific historical run.

The more important output was the configuration model itself.

Knowledge that previously existed through manual testing and product familiarity was converted into structured, machine-readable rules that could drive deterministic automation.

---

## Takeaway

The core problem was not:

> How do I make the API call?

It was:

> What knowledge and safeguards need to exist before that API call is safe to automate?

The final workflow became:

```text
manual investigation
        ↓
validated configuration model
        ↓
desired-state comparison
        ↓
dependency planning
        ↓
controlled mutation
        ↓
verification
        ↓
audit + rollback
```

That same pattern can apply to many operational systems where important domain knowledge exists informally and needs to be converted into repeatable automation.

---

## Skills Demonstrated

- SaaS technical operations
- REST API investigation
- Configuration analysis
- Schema design
- Manual QA and behavioral validation
- Dependency modeling
- Workflow architecture
- Failure-mode analysis
- Python
- YAML
- API validation
- Desired-state comparison
- Test design
- Bulk API optimization
- Operational safeguards
- Rollback design
- Audit logging
- Translating domain expertise into structured automation

---

## Repository Scope

This repository is a **case study**, not a distributable version of the internal tool.

It intentionally excludes:

- production source code;
- the real configuration registry;
- real API endpoints and field names;
- client or company information;
- access credentials;
- real run folders;
- production logs;
- branded product screenshots.

Any supporting examples in this repository are synthetic and exist only to demonstrate the design and reasoning behind the project.