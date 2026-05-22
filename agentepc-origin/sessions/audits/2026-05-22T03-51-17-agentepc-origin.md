---
schema_version: 1
audit_id: 2026-05-22T03-51-17-agentepc-origin
product: agentepc-origin
generated_by: claude-sonnet-4-6
generated_at: '2026-05-22T03:50:00Z'
sources_summarized:
- kind: claude_session
  path: /home/readystack/.claude/projects/-home-readystack-projects-AgentEPC-origin/97960f6c-7b74-4d3d-a39a-c45b21daa5f9.jsonl
  session_started: '2026-04-24T10:45:58.575Z'
  session_ended: '2026-05-06T17:27:24.746Z'
  session_size_bytes: 21325694
  truncated: true
---

## Decisions made

The operator and assistant agreed on a full production database audit scope and approach through an 11-question design session. Decisions locked:

- Deliverable is findings report plus fix plan only; no remediation during the audit pass.
- Schemas in scope: AIEPC_BRI0 and master only.
- All 8 audit dimensions are in scope: naming and convention compliance (A), referential integrity (B), index hygiene (C), EAV consistency (D), blueprint vs instance separation (E), BRI0 to master drift (F), sync vs UI write collision matrix (G), and derivation freshness (H).
- Output format: single human-readable doc (docs/AUDIT_2026-05-02.md), machine-readable JSON sidecar (docs/audit/2026-05-02-findings.json), and per-dimension SQL files (scripts/audit/2026-05-02/<letter>-<name>.sql). The operator initially chose a single doc, then revised to option D (doc plus sidecar plus per-dimension SQL).
- Fix plan ranking: severity only (Critical / High / Medium / Low); effort and risk-of-fix are deferred to the implementation plan pass.
- DB execution mode: read-only with every query wrapped in BEGIN; ... ROLLBACK;.
- Naming and blueprint-vs-instance violations use a three-way diff: DATA_DICTIONARY.md vs migration headers vs live information_schema; disagreement between any pair is itself a finding.
- EAV consistency depth: deep (coverage plus write-path matrix plus read-path matrix plus live-row round-trip on BRI0 for a 100-row sample).
- Sync vs UI collision depth: validate 4 known SYNC_CONCERNS.md items plus systematic sweep of all sync-write tables plus ON CONFLICT semantics audit.
- Derivation freshness depth: visibility cascade plus badge counts plus materialized inbox plus visibility-rule tree integrity.
- Execution parallelism: two-phase hybrid — 5 parallel subagents for schema-only dimensions (A, B, C, E, F), then one sequential agent for code-tracing dimensions (D, G, H).
- All 5 design sections were approved by the operator.

A separate earlier decision (pre-design-session) was made to pause a prod deploy and not proceed without explicit confirmation. No prod changes were made during the session. The operator instructed the assistant to pause rather than auto-proceed.

The assistant determined that the ai_step_instance / ai_task_instance two-table split is intentional and correct, mirroring Coperniq's ElementInstance / Task separation. The split is not redundant; tasks-without-workflow (SERVICE tickets) and workflow-positions-without-tasks both exist in live BRI0 data.

The assistant concluded that the root cause of the audit's expected findings is sync writing poorly across schema splits, not schema design defects. Specifically: (1) the task split is written inconsistently across the two halves, producing id-space confusion and orphan rows; (2) the catalog split (am_attribute_catalog / am_field_spec) is written with broken bindings (376 specs with no attribute_ref, 8 duplicates per attribute, 5,065 unanchored task_virtual specs, 721 specs with legacy id instead of local id); (3) task_virtual reprojection rows in ai_field_entry are written but consumed by no read path.

Memory files (memory/MEMORY.md, memory/project_state_sources.md, memory/project_deploy_and_infra.md) were confirmed wiped after a session crash. The two git-tracked repo files (docs/DEPLOYMENT.md and docs/V1_GO_LIVE_PLAN.md) survived.

## Open questions

Whether to recreate the three wiped memory files was raised but not acted on during the session (the operator moved to a different task after the crash was surfaced).

The Phase 8 manifest-drift product decision was listed as a pending open item from the prior session and was not resolved.

The operator asked whether Claude updated memory files before the session crashed; the answer was no (memory dir was empty), but no follow-up decision was made on reconstruction.

## People mentioned

No named individuals are mentioned in this session.

## Dates and deadlines referenced

- 2026-04-24: session date for the prior work session whose results doc is referenced (docs/SESSION_2026-04-24_RESULTS.md). That session ran spec 71 31/31 green.
- 2026-04-29: due date on the open service ticket visible in the screenshot (ticket #12557, Felipe Santiago project).
- 2026-05-01: date on multiple uploaded screenshots (Screenshot_2026-05-01_141534.png, Screenshot_2026-05-01_143756.png, Screenshot_2026-05-01_144221.png, Screenshot_2026-05-01_143756.png).
- 2026-05-02: target date embedded in the audit output file names (docs/AUDIT_2026-05-02.md, scripts/audit/2026-05-02/).

## Thoughts not yet in the canvas

The operator uploaded a HAR file named JDInbox.har (357,754 bytes) and saved it to the .spyglass/uploads/ directory. No action was taken on it and no canvas milestone exists for HAR-based inbox analysis.

Multiple screenshots and HAR files were uploaded across several exchanges: a PTO/interconnection status lifecycle diagram, an agentepc status section screenshot, project Financials tab, project Properties tab, project Docs tab, account/site detail for Mario Gonzalez-Silvestre at 4519 East Mono Avenue Fresno, and corresponding HAR files (changelog_projects.har, Financials.har, projects_docs.har, projects_properties.har, accounts_sites.har). All were saved to .spyglass/uploads/ with no analysis or action taken beyond confirming file names and landing paths. The purpose of this upload batch is not captured in a canvas milestone.

The operator expressed concern about "lala-land" EAV inconsistency and "spaghetti joins" as a data integrity risk. This framing preceded the formal brainstorming session and is captured now in the design doc, but the underlying root cause analysis (sync writing splits inconsistently) surfaced later in the session and is not yet reflected in any known canvas milestone or track.

The assistant noted that ai_work_ticket is deprecated — sync-coperniq.js still writes to it after migration 100 consolidated service tickets into ai_task_instance, making that sync path broken end-to-end. No milestone exists for fixing the broken ai_work_ticket sync path.

The operator asked about the session transcript location (not auto-saved by Claude Code); a session doc was explicitly created at docs/SESSION_2026-04-24_DEPLOY.md.

The finding categories labeled A-01, B-007, B-008, D-04, D-08, D-10, S-7, S-8, B-021 through B-028 were surfaced in the tail of the truncated session, indicating the audit was at least partially executed and produced labeled findings. The specific counts cited: 12,342 orphans on ai_task_instance.author_ref (A-01), 376 specs missing attribute_ref (D-10), 8 duplicate specs per attribute (D-08), 5,065 unanchored task_virtual specs (D-04), 721 specs with legacy id in arrangement_ref (B-008). These findings and their severities are not yet mapped to any canvas milestone.

## Status changes implied

No track or milestone slugs are present in the canvas snapshot (§7 is empty), so no slug-based status changes can be stated. Implied from the session content: the database audit design work transitioned from planned to active (design approved, execution plan written, design doc committed to docs/superpowers/specs/2026-05-02-prod-database-audit-design.md), and at least partial audit execution occurred based on the finding references visible in the session tail.

