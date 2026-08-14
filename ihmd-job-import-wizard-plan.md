# IHMD Job Import Wizard Plan

Status: Phases 1–4 implemented; Phase 5 intentionally deferred pending production evidence and catalog-workflow approval

## Implementation Status

- Durable Active Storage-backed import sessions, staged rows, background parsing, expiry, and cleanup are implemented.
- All six workbook types parse through the header-based reconciler. The private acceptance workbook produced the expected 4,204 coded, 3,604 excluded, and 600 review rows.
- Row reconciliation, explicit clears, bulk actions, stale-snapshot checks, ordered per-job-type advisory locks, atomic commit, rollback, and retry behavior are implemented.
- Every staged workbook is submitted to MetaDefender before Roo opens it. Only a clean, signed verdict for the import's current attachment can queue parsing, and both parse and commit enforce the clean verdict independently.
- The session GraphQL API and two-step Upload and Review React wizard are implemented. Column classification, row-level conflict resolution, and the atomic commit action now live together in Review.
- Legacy V1 umbrella and type-specific import routes remain available during rollout. The Jobs page exposes separate Legacy Import and New Import buttons; the new importer runs at `/jobs/import-v2` and retains the malware-scan gate.
- The V2 importer detects ambiguous normalized type/code matches during review and rechecks them inside the serialized commit transaction. V2 import sessions are created, reviewed, and committed only through the `/jobs/import-v2` wizard.
- Material/material-list creation and membership mutation remain out of scope until the separately reviewed Phase 5.

## Summary

Replace the seven blind import flows with one durable, resumable wizard covering Flooring, Painting, Miscellaneous, Cap-ex, Stack Cleaning, and Duct Cleaning.

The workflow will be:

1. **Upload:** Upload the workbook, wait for a clean MetaDefender verdict, and then parse it.
2. **Review:** Classify unknown columns, resolve row conflicts, compare new and existing jobs, stage Add, Update, or Skip choices, and atomically commit them. Keep the staged rows visible but immutable while committing and after success.

Only fields supported by `IHMDJob` will be saved. Labour costs, individual materials, material quantities, final costs, calculated rates, scheduling fields, and other unsupported workbook values are ignored and never persisted.

## Backend and Data Model

- Add `JobImport` records containing creator/committer, state, source filename, parsing progress, mappings, summary, errors, timestamps, and state-aware expiry metadata.
- Attach each workbook using Active Storage and a unique storage key, replacing the current singleton S3 object and Redis-only state.
- Store the attachment scan status/result on `JobImport`. Attachment creation queues `VirusScanJob`; a signed MetaDefender callback must match the current workbook attachment before it may mark the import clean and queue parsing. Infected workbooks fail without parsing and are purged; scan errors fail without parsing and retain the workbook only under the failed-session retention policy.
- Treat scan orchestration as defense in depth rather than the only gate: parser entry, parser persistence, commit request, and the locked commit transaction all reject a workbook whose verdict is not clean.
- Development alone uses an explicit clean-verdict bypass for both newly uploaded and resumed pre-scan sessions so local work does not depend on a reachable MetaDefender callback. Test and production retain the real scan gate.
- Add `JobImportRow` staging records containing:
  - Sheet and Excel row number.
  - Job type, original code, and normalized code.
  - Raw workbook values.
  - Parsed `IHMDJob` attributes.
  - Existing database values and `updated_at` snapshot.
  - Operator overrides, issues, classification, and Add/Update/Skip action.
- Index staged rows for server-side filtering by import, job type, classification, issues, and action.
- Parse and commit in background jobs; add a daily cleanup job enforcing state-aware retention. Reviewing and failed sessions expire 14 days after their last saved activity, succeeded workbooks are detached for purge immediately while their immutable rows remain for 7 days, discarded sessions are destroyed immediately, and committing sessions are never purged before completion.
- Keep upload, parsing, reconciliation, and review concurrent. Serialize only the final commit transaction against overlapping conflict domains:
  - Acquire a PostgreSQL transaction advisory lock for each included STI job type, ordered by a stable lock key before acquisition to prevent deadlocks. A multi-type import holds every applicable type lock until its transaction ends.
  - Do not scope job locks by department. `ihmd_jobs` codes are globally unique per STI type, so two departments can still conflict on the same type/code.
  - Do not use one global job-import lock. Imports containing disjoint job types may commit concurrently.
  - Acquire a separate global material-catalog advisory lock only when the transaction creates a material/list or changes list membership. Merely assigning an existing list to a job does not require that lock.
  - Refresh matches, snapshots, authorization, and validation only after all required advisory locks have been acquired, then write everything in the same database transaction.
- Match nonblank job codes using the same `type + lower(btrim(code))` identity throughout reconciliation and commit. The importer reports ambiguous legacy matches but never merges or deletes them automatically. Ordered per-type commit locks prevent concurrent import sessions from creating the same normalized identity.

### Parsing and Reconciliation

- Parse supported sheets by normalized sheet/header names rather than fixed column positions. Header matching ignores case, whitespace, punctuation, and embedded line breaks.
- Accept column-order changes and known aliases such as `Cap-ex`, `Job Code`, and `Cap-ex Code`.
- Require at least one supported sheet and a nonblank job code for every candidate row.
- Exclude recognized Complete/Completed and Canceled/Cancelled rows from preview and import, while retaining excluded counts in the summary.
- Flag unknown values such as `C`, `Complted`, or `ON HOLD` as row-level status issues for correction in Review.
- Match existing jobs by STI job type plus trimmed, case-insensitive code. Store new codes trimmed while preserving their workbook capitalization.
- Require operator resolution when:
  - Multiple workbook rows have the same normalized type/code.
  - A workbook row matches multiple legacy database jobs.
  - A branch, building, status, material list, or column cannot be resolved.
- Do not automatically merge or delete existing duplicate-code records.
- For updates, apply only nonblank mapped workbook values. Blank cells preserve database values unless the operator explicitly chooses Clear in the row editor.
- Map Duct Cleaning `completion_date` to the latest Last/2nd Cleaning Date and Stack Cleaning to the latest Previously/Current FY Completed date.

### Materials and Material Lists

The job-only importer may assign an existing material list or an explicit No Material List choice, but it must not create `Material`/`MaterialList` rows or change list membership.

- Ignore individual material columns, quantities, and costs, including known columns such as Glue, LVP, Paint, Mortar, and Material Cost.
- For Flooring, Painting, Miscellaneous, and Cap-ex, treat Job Description as evidence for `material_list_id`. Stack and Duct Cleaning do not infer a material list from description.
- Assign a unique normalized match to an active material list automatically.
- Treat no match or multiple matches as a row-level material-list issue:
  - A New row must select an existing material list or explicit No Material List to resolve the issue.
  - An Existing row prepopulates its current database material list. Saving that unchanged value explicitly resolves the issue without changing the association.
  - Selecting another material list stages a replacement.
  - Selecting No Material List explicitly clears an Existing job's association and leaves a New job unassigned.
  - Leaving the issue unresolved keeps the row on Skip while allowing other resolved rows to proceed.

Phase 5 remains a separate catalog-management project:

- Auto-suggest existing materials using normalized names and aliases, including `Glue` -> `Glue (LVP)`, `Torly` -> `Torly Flooring`, and `Bulkheads` -> `Bulkhead`.
- Classify unfamiliar material headers, stage new materials/lists, show impact counts, and permit additive material-list membership changes.
- Require both job-management and material-management permission to create materials/lists or alter list membership.
- Acquire the global material-catalog advisory lock for catalog mutations.

Review Phase 5 independently for fuzzy-match thresholds, alias ownership, staged catalog validation, permission boundaries, propagation, existing-job impact, and rollback behavior. Ship it only after the job-only importer has production evidence and dedicated acceptance approval.

## GraphQL and Wizard UI

Use the session-oriented GraphQL interfaces:

- `createJobImport(file, includedJobTypes)`
- `jobImport(id)` and `jobImports`
- `jobImportRows(importId, filters, pagination)`
- `updateJobImportMapping`/`updateJobImportMappings`
- `updateJobImportRow`
- `bulkUpdateJobImportRows`
- `commitJobImport`
- `discardJobImport`

Expose typed import states, summaries, issues, mappings, row diffs, progress, and results. During rollout, legacy umbrella/per-type routes continue rendering their V1 importers while V2 uses `/jobs/import-v2`; retire the old mutations and Redis status queries only after users move over. Both versions require `allowedToManageJobs`.
Every job-import GraphQL query and mutation requires the corresponding server-side `manage_jobs` permission, matching the legacy import mutations; client visibility is only a convenience and never replaces that backend check.

The revised UI will provide:

1. **Upload:** XLSX validation, explicit malware-scan status, detected-sheet selection, parsing progress, and source summary.
2. **Review:** Server-paginated New, Existing, and Issues tabs; unknown-column classification; job-type filters; checkbox-staged row actions; populated row details; the shared table with Mantine's numbered pagination control; direct atomic commit; and read-only post-commit results. Phase 5 may later add staged catalog/list changes.

The header uses the standard Cancel action to return to Jobs without deleting the resumable staged session.
On small screens the Cancel action retains its visible X icon, the stepper shows only the current Upload or Review step, and recent sessions use the shared card pattern with explicit read-only status for committing or succeeded imports.

### Review Table and Actions

- Remove the separate Resolve step and move resolution into Review.
- Keep import-wide mapping only for unknown columns because changing a column's meaning requires reparsing every row. Each unknown column must map to a supported field or Ignored.
- Apply all pending column mappings in one mutation and queue one reparse. Keep row editing and selection actions read-only until unknown columns are classified and that reparse finishes, so reparsing cannot delete completed review work.
- Resolve statuses, branches, buildings, material lists, validation errors, and duplicate-code corrections through individual row details without reparsing.
- Continue honoring non-column mappings already stored on a resumed session, but do not create new non-column mappings through the revised UI.
- Retain New, Existing, and Issues tabs, but remove the Classification column and per-row Action dropdown.
- Replace the Issues count with an accessible warning icon, show up to three issue messages beside it followed by `and more` when needed, and retain the complete list in the tooltip.
- Render job codes as plain text and translate stored job-type class names to the same human-readable labels used by the job-type filter.
- Use the common filled edit icon for row details. Make Select/action, Code, Type, Sheet row, and Issues server-sortable.
- Provide a debounced text search across the complete filtered result set, matching code, job type, sheet/row, and issue text. Reset to page 1 and recalculate numbered pagination when the search changes.
- On small screens, keep the Review tabs in one horizontally scrollable row and render results with the shared card pattern; preserve selection, issue summaries, edit/view actions, server sorting, and numbered pagination. Keep the sortable shared table on larger screens.
- Use the checkbox itself as the staged action and remove the Add selected, Update selected, and Skip selected buttons:
  - New initializes eligible rows as selected; checking stages Add and unchecking stages Skip.
  - Existing initializes rows as unselected; checking stages Update and unchecking stages Skip.
  - Issues provides review access but cannot select unresolved rows.
- Persist each checkbox change to `JobImportRow#action`. It remains staged until the Review commit action; only that atomic commit writes production `IhmdJob` records.
- Valid new active jobs default to Add. Existing jobs, including changed ones, default to Skip. Unresolved rows stay Skip and may remain skipped without blocking the Review commit action.

### Populated Row Detail

- Remove the detail-level Action dropdown. Saving corrections does not automatically select the row for import.
- Populate every persisted form field with the proposed resulting value:
  - New rows use workbook values overlaid with saved overrides and explicit clears.
  - Existing rows use database values overlaid with workbook values, saved overrides, and explicit clears.
- Keep Workbook and Final Proposed summaries visible. Show Database only for matched Existing rows; hide it for New and ambiguous rows.
- Remove the separate Explicitly Clear checkbox. Infer a clear when the reviewer empties a text field or chooses No Material List, while preserving `clearFields` in the staged API representation.
- Highlight Existing fields that differ from the database in yellow and show the database value. Keep unresolved issue fields red with `Spreadsheet value: ...`; after the reviewer changes one, convert it to a yellow warning that retains the original spreadsheet value.
- Normalize parser fields such as branch, building, and material list to their corresponding `branch_id`, `building_id`, and `material_list_id` controls. Show non-field issues in the drawer alert.
- Persist only values changed from the workbook/database baseline as overrides. Saving a conflicted field must still count as an explicit resolution when the chosen value equals the existing database value, as with retaining an Existing job's material list.

Before commit, refresh all database matches and snapshots. Any stale row, newly introduced duplicate, validation failure, or authorization change returns the session to Review with no production changes. The final transaction creates or updates jobs only and may assign existing material lists. Phase 5 may later extend that transaction with catalog mutations while holding the material-catalog advisory lock.

## Delivery Phases

Each phase must be independently reviewable and green before the next begins.

### Phase 1: Durable Staging and Parser

- Add `JobImport`/`JobImportRow`, state transitions, workbook attachment, expiry, and cleanup.
- Integrate `JobImport#workbook` with MetaDefender and prohibit parsing or committing until the current attachment has a clean verdict.
- Build the background parser, normalized header/sheet handling, row classification, progress reporting, and deterministic summaries.
- Add the sanitized synthetic workbook fixture and parser/repository tests.
- Detect duplicate workbook codes and ambiguous normalized database matches during reconciliation.
- Make no `IHMDJob`, material, or material-list production writes from staged imports in this phase.

### Phase 2: Job Reconciliation and Commit Engine

- Implement job-only matching, mappings, diffs, Add/Update/Skip, explicit clears, stale-snapshot checks, validation, idempotent retry, and rollback.
- Implement ordered per-job-type PostgreSQL transaction advisory locks and revalidation inside the locked transaction.
- Exercise the engine through service tests and a controlled Rails runner/rehearsal interface before exposing mutations.
- Permit assignment of existing material lists or No Material List, but no catalog mutation.

### Phase 3: GraphQL Session API

- Expose the session, row pagination, mapping, row/bulk update, commit, discard, progress, and result interfaces.
- Add authorization, ownership, state-transition, pagination, polling, retry, and concurrent-commit integration tests.
- Keep the existing import mutations and routes active only until the secure wizard cutover.

### Phase 4: Wizard and Legacy Cutover

- Refactor the implemented four-step wizard into Upload and Review, with commit progress and results retained in Review.
- Move column classification and row-level resolution into Review; remove Resolve and its obsolete UI helpers.
- Implement the revised table selection, issue icon, populated detail form, conditional Database comparison, conflict highlighting, and material-list behavior.
- Keep existing staged sessions and stored mappings readable; no ActiveRecord data migration is required.
- Add frontend mapping, form-baseline, explicit-resolution, filtering, selection, interruption/resume, and failure-state tests.
- Run the old and new flows side by side for acceptance, then redirect legacy routes and retire the Redis/singleton-upload implementation after successful cutover.
- Disable every legacy umbrella/type-specific backend import mutation at cutover so callers cannot bypass workbook scanning through an older GraphQL operation.

### Phase 5: Material Catalog Reconciliation

- Add fuzzy/alias suggestions, staged material/list creation, additive list membership, impact counts, and catalog permissions.
- Acquire the material-catalog advisory lock for commits that mutate shared catalog data.
- Review, test, and release this phase independently from the job-only wizard.

## Test Plan

- Check in a sanitized, synthetic XLSX fixture under `test/fixtures/files` with every supported sheet shape, shuffled columns, aliases, multiline headers, duplicate codes, relation mappings, terminal rows, and unknown statuses. Keep the fixture small enough for normal CI and document how to regenerate it.
- Parser tests for all six sheets, shuffled columns, header aliases, multiline headers, missing sheets, unsupported columns, and unknown statuses using the committed fixture.
- Reconciliation tests for normalized codes, workbook duplicates, ambiguous legacy matches, blank codes, relation mappings, default actions, nonblank update merging, and explicit clears.
- Material-list tests for automatic unique matches; unknown and ambiguous descriptions; keeping, replacing, and clearing Existing associations; New rows choosing an existing list or no list; and unresolved rows remaining skipped.
- Prove individual material columns remain ignored and the job-only importer never creates materials, material lists, or memberships.
- Commit tests proving stale-data detection, serialization, idempotent retry, and complete rollback when any selected change fails.
- GraphQL integration tests for authorization, numbered pagination, server sorting, session transitions, polling, one-reparse column mapping, row corrections without reparsing, staged selection actions, and state-aware retention cleanup.
- Malware-boundary tests proving V2 upload queues scanning rather than parsing, only the signed current-attachment clean callback queues parsing, stale callbacks are rejected, infected/error results never parse, and parser and commit independently reject non-clean V2 workbooks. Legacy V1 mutations remain available temporarily and do not share this gate.
- Frontend tests for two-step navigation, direct Review commit, immutable committing/succeeded rows, checkbox-staged Add/Update/Skip actions, default New-row selection, issue icons, populated New/Existing forms, conditional Database display, minimal overrides, inferred clears, database-difference warnings, corrected conflict warnings, and database-equivalent explicit resolutions.
- Run targeted Rails tests, JavaScript tests, Sorbet, RuboCop on touched Ruby files, TypeScript typecheck, and the production build.
- A private local validation run against Master Work Summary 2022 identified 4,204 coded rows, excluded 3,604 recognized terminal rows, and presented 600 rows for review. This workbook is not a repository dependency or a separate import interface; V2 acceptance runs through `/jobs/import-v2`.

## Assumptions

- XLSX is the only supported format.
- This Phase 4 change is a code/UI refactor of the existing importer, not an ActiveRecord data migration.
- Existing staged sessions and previously stored mappings remain readable.
- Unsupported costs, calculated metrics, individual materials, and per-row material amounts are never persisted.
- Global mappings are scoped to the import session; the revised UI creates only column mappings globally.
- Add/Update/Skip choices persist to staging; production jobs change only during the atomic commit initiated from Review.
- Unresolved rows may remain skipped without blocking the commit of resolved selected rows.
- Existing duplicate codes and legacy blank Stack Cleaning jobs are reported when encountered but are not cleaned up by this project.
- Existing duplicate codes and newly ambiguous matches must be resolved in the wizard or source data before those rows can be committed.
- Material/list creation and membership changes are explicitly out of scope until Phase 5; their absence does not block delivery of the job-only wizard.
