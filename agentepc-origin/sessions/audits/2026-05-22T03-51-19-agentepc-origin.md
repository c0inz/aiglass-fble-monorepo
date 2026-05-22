---
schema_version: 1
audit_id: 2026-05-22T03-51-19-agentepc-origin
product: agentepc-origin
generated_by: claude-sonnet-4-6
generated_at: '2026-05-22T03:50:00Z'
sources_summarized:
- kind: claude_session
  path: /home/readystack/.claude/projects/-home-readystack/6b1b904e-830e-46df-8074-c0b811a90b15.jsonl
  session_started: '2026-04-18T16:28:35.731Z'
  session_ended: '2026-05-02T19:42:19.657Z'
  session_size_bytes: 92495015
  truncated: true
---

## Decisions made

Phases 1a and 1b of the workplan were confirmed already complete from a prior session; no SQL fallbacks remained in specs 69, 80, or 81, and spec 69 already used the `#stepPickerModal` UI.

Phase 2 was completed: worker creation in specs 80 and 81 was converted from direct SQL inserts to driving the `/settings/members` UI, preserving the `workerMap` shape downstream tests depend on.

Phase 4a was proceeded with after operator approval. The work expanded from filling form fields in tests to fixing an underlying product bug (bug #31): fresh tenants had an empty `am_field_spec` table, making every form field save fail with a 400 error.

The decision was made to fix the root cause in the product rather than soft-assert in tests. Migration 037 (`037_derive_form_field_specs.sql`) was written to derive `am_field_spec` rows mechanically from arrangement and attribute data.

After operator challenge, the scope of migration 037 was extended to cover all applicable origins (`form_field` and `project_property`), not just `form_field`. The `task_virtual` origin was explicitly deferred because the WO save path writes directly to `ai_task_instance` / `ai_step_instance` columns rather than through `ai_field_entry`, making task_virtual specs inert until that save path is refactored.

`routes/forms.js` was fixed so spec lookup is keyed by `arrangement_ref` plus an `origin='form_field'` filter, preventing the same attribute placed multiple times on a form from collapsing to one value.

`routes/projects.js` was fixed to filter property-tab reads and saves by `arrangement_ref IS NULL` and `origin='project_property'`, isolating project-level values from form-field values.

All code changes were committed and deployed to `app.agentepc.com` via rsync after git pull was found not to be wired. Deployment was confirmed by matching MD5s across four files and a successful pm2 restart.

Operator directed that the WO context save path must be refactored to write through `ai_field_entry` so task_virtual specs are load-bearing. The decision was made to use dual-write (keep column writes, add field_entry writes) rather than swap.

A `data-state` attribute marker was added to editor partials (`form-editor.ejs`, `work-order-editor.ejs`, `workflows.ejs`) as a load-state contract. Disabled-button enforcement was attempted (round 11) but caused a regression at Phase 6 field arrangements and was reverted to marker-only.

The `renderFields()` null-dereference bug in `form-editor.ejs` was fixed and kept regardless of the enforcement revert: `document.getElementById('fieldsEmpty')` returns null after the first non-empty render because `container.innerHTML = html` removes the element, so both `empty.style.display` calls were guarded with `if (empty)`.

A pre-existing race in spec 71 where the Standard Brighthouse workflow's label was silently overwritten to "Project" during Phase 7 fill-metadata was identified and fixed by explicitly re-filling the label in the Edit Workflow modal before save.

Spec 71 reached 31/31 passing at round 12 (28.0 min) in the marker-only state with the null-dereference fix and the workflow label fix applied.

## Open questions

Whether the WO save path should be refactored to write through `ai_field_entry` for task_virtual specs was raised and the operator confirmed yes, but implementation was not completed in this session.

Ad-hoc steps added to projects (not placed from a blueprint) lack a `step_blueprint_ref`, so task_virtual spec resolution for those steps was noted as needing a fallback spec keyed on `(step_instance_id, field_key)`. The operator asked about this case and the question was raised but not fully resolved in code before the session ended.

Whether the `/logout` session-store pool hang was caused by migration 037 or by long-uptime pool leakage remains unconfirmed. The root cause hypothesis (connect-pg-simple pool exhaustion) was noted but not verified.

Disabled-button enforcement on `data-state !== "ready"` remains deferred. The precondition for re-attempting is: (1) audit all code paths inside editor `.then()` blocks for null-dereference bugs, and (2) update spec selectors to wait on `data-state="ready"` rather than inferring readiness from form name population.

The `project_property` derivation for BRI0 produced 875 specs (up from 472). Whether the 472 legacy specs that were not reproduced by the derivation represent dead data or missing context was not audited.

`ai_step_instance.step_blueprint_ref` does not exist as a column and needs to be added before task_virtual derivation and dual-write can be completed.

## People mentioned

No people were named or referenced by name in the session inputs.

## Dates and deadlines referenced

Session started 2026-04-18 and ended 2026-05-02 per the session metadata header.

BRI0 backfill was applied during the session (noted as occurring around 23:36 UTC in the `/logout` incident discussion).

The Brighthouse fixtures snapshot used is dated 2026-04-16.

No explicit deadlines were referenced.

## Thoughts not yet in the canvas

The `/logout` endpoint hung indefinitely during the session. The server was SSHed into, investigated, and restarted via pm2. The root cause hypothesis is session-store pool exhaustion (connect-pg-simple or equivalent), independent of migration 037. A follow-up to add pool timeout and logging was noted but not filed as a formal item.

The `ai_field_entry.step_instance_ref` column exists in the schema as nullable but the save handler does not populate it. The read path keys on `(venture_ref, field_spec_ref)` so it is not functionally broken, but the column is described as dead data. No action was taken.

`service_ticket_field` origin was identified as a one-row edge case requiring its own derivation rule. It was noted but not acted on.

The session ran multiple full rounds of spec 71 (rounds 7 through 12) during a single operator conversation, partly due to the load-state contract attempt introducing regressions that required multiple fix-and-rerun cycles. The operator questioned why the suite was being re-run and was given an explanation.

A GitHub PAT was shared by the operator in plain chat and added to `~/credentials/github.json` under the key `pat_extra_classic`. The value is redacted here. The assistant noted the token should be rotated as it was shared in plain text.

A database password appeared in tool call commands during the session. The value is redacted here.

## Status changes implied

No canvas track or milestone slugs were referenced in the inputs. The following status changes are implied by the session content:

Phase 1 (remove SQL fallbacks, specs 69/80/81): confirmed already complete from a prior session; no action needed.

Phase 2 (worker creation via UI): completed and verified green (24/24 in specs 80+81, then 63/63 full suite).

Phase 4a (form field CRUD with canonical values): completed with bug #31 fixed end-to-end; 63/63 full suite passing at 16.0 min; deployed to production.

Bug #31 (empty `am_field_spec` on fresh tenants, all form saves returning 400): fixed via migration 037, provisioner wiring, routes/forms.js and routes/projects.js corrections, and a one-shot backfill of BRI0 (form_field: 726 → 1576; project_property: 472 → 875).

Spec 71 load-state contract: `data-state` marker added to three editor partials; disabled-button enforcement attempted and reverted; `renderFields` null-dereference fixed; workflow label race fixed; 31/31 at round 12.

