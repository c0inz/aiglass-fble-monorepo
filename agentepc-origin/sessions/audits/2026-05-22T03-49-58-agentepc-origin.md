---
schema_version: 1
audit_id: 2026-05-22T03-49-58-agentepc-origin
product: agentepc-origin
generated_by: claude-sonnet-4-6
generated_at: '2026-05-22T03:50:00Z'
sources_summarized:
- kind: claude_session
  path: /home/readystack/.claude/projects/-home-readystack/6cf5624f-bb8d-49e0-b24a-6b84b5f309b6.jsonl
  session_started: '2026-05-06T17:15:54.507Z'
  session_ended: '2026-05-21T22:21:20.150Z'
  session_size_bytes: 223287858
  truncated: true
---

# Session 2026-05-06 — AgentEPC-origin UI sweep, BRI0 reload, backup pipeline

## Decisions made

The operator decided to restart the session with high thinking mode after a prior session run on no-think caused numerous problems. All work going forward uses the new session.

The operator directed a full route-and-lib sweep of every file referencing refactored tables (`ai_step_instance`, `ai_task_instance`, `ai_work_ticket_old`, and the 28 under-account tables from mig 118), rather than the narrower 3-file patch the workplan described. The assistant agreed to patch all 17 affected files.

The operator decided to truncate BRI0 and repull all three target accounts (615611, 693903, 694215) fresh from Coperniq using the canonical GraphQL extractor, rather than browse what was partially loaded.

The operator confirmed the master suite provisions a fresh schema and does not touch BRI0. The assistant confirmed this from the spec code (`provisionTenant` generates a new `AIEPC_<slug>` schema).

The Coperniq token was found to be valid through 2026-05-18 23:49 UTC — no refresh needed for the 3-hour export run.

The assistant decided (with operator lean-confirmation) to push the 16 unpushed local commits to origin as part of the cleanup commit.

The batch export cron was set to keep marching on individual batch failure rather than halt (each batch is independent and recoverable via the SUMMARY.md trail).

Migration 123 was applied: all 70 CASCADE foreign keys in BRI0 and MAS0 were converted to NO ACTION.

The `sitepilot` backup pipeline was built and deployed end-to-end: prod dumps at 02:00 UTC, a bombadil relay cron at 04:30 MT pulls and pushes to sadb, prunes prod to today-only, and applies tiered retention. Six legacy manual dumps (459 MB) were moved off prod to sadb.

The operator decided to use Resend for backup failure alerts instead of msmtp/Gmail app password. The Resend API key from `~/credentials/agentepc-resend.json` was deployed to bombadil. A test alert was sent and confirmed delivered to `johntdavenport@gmail.com`.

## Open questions

The image mirror (fixing `ai_attachment.retrieval_url` to point at the self-hosted copy on 64.23.183.9 instead of Coperniq URLs) was flagged but not completed during this session. The operator said they did not know what the image mirror topic was; the assistant deferred it.

The four open image-mirror owner decisions (path scheme, downtime tolerance, deleted-files policy, R2 vs single-host) remain unsigned.

Whether the tenant provisioner produces a post-rewrite schema (new `ai_task_instance` shape, no dropped columns) was flagged as a caveat for master suite readiness but not verified during this session.

`lib/sync-column-map.js` was intentionally left unpatched — it drives `sync-coperniq.js`, not the browsing path. Will need revisiting when sync resumes.

The Coperniq batch driver was mid-run (batches 7–9 of 23 visible in the blob tail). Final completion status of all 23 batches is not confirmed in this session record.

Account 308312 had a SQL syntax error in the loader (empty IN-clause in `deleteScoped`) during batch 4. It was not re-run within this session.

Three open PRs were listed at session end (#1 stub blueprints for 333 orphan form-tasks, #2 Row 1 consolidation + audit_review_ledger, #3 backup fix + offsite relay) — none merged.

The operator reported forms were no longer erroring after Phase 1 deploy but showed a massive rendering regression (nothing rendering) and only two projects visible with "untitled" labels. The assistant identified the root cause (schema drift in `am_field_arrangement` / `am_field_spec` split post-mig-118) and issued further fixes. Whether those fixes fully resolved the rendering regression is truncated from this record.

## People mentioned

Tom Davenport (listed as account owner in Coperniq data for account #243 david howe).

Isaac Rosa (listed as account owner in Coperniq data for account #544 Ismael Bejarano).

Keaton Garner (listed as account owner in Coperniq data for account #306 Michael Smith).

## Dates and deadlines referenced

2026-05-04: date the Coperniq token was last refreshed.

2026-05-18 23:49 UTC: Coperniq token expiry — confirmed sufficient for the 3-hour export run.

2026-06-06: gate date for migration 103 (final `_old` table drop), referenced in the workplan context.

2026-04-15: date the pm2 ecosystem guard (max_restarts / min_uptime) had been on the workplan but never landed.

02:00 UTC nightly: prod `pg_dump` schedule on GrandGallery (64.23.183.9).

04:30 MT nightly: bombadil relay cron schedule pulling and pushing dumps to sadb.

43 days: duration the nightly prod backup cron had been dead before this session fixed it.

## Thoughts not yet in the canvas

The blast radius of the schema refactor was larger than documented: the workplan cited 3 files and 52 CRUD findings, but 19 files reference `ai_step_instance` and 14 reference `ai_work_ticket_old`. The final patch count was 17 files across routes and lib.

The `ai_remark.work_ticket_ref` column was found not to exist — an undocumented schema drift point surfaced during the sweep.

BRI0 contained only 2 ventures (not 3) after the cascade recovery: accounts 615611 and 694215 were missing because the re-pull manifest had excluded them based on pre-recovery status. This was identified and corrected.

The wipe-acct script had a field_spec deletion logic bug that surfaced after CASCADE FKs were removed (it tried to delete specs shared with other accounts). The script was patched to skip field_spec/field_arrangement deletion and rely on UPSERT in the loader.

The loader was missing hydration for `process_ref`, `active_stage_ref`, `serial_number`, owner/manager, and address fields on projects. These were added.

The `routes/forms.js` `ai_event_stream` query referenced `es.assignment_ref` (does not exist on that table — correct column is `step_instance_ref`), causing all 116 step-detail page 500s before the fix.

Post-fix crawl: 179/179 step detail pages returned 200 and rendered clean across all 3 accounts and 4 projects.

The `am_field_arrangement` / `am_field_spec` column split introduced by mig 118 was not reflected in the form renderer. `arrangement_kind`, `caption`, and `attribute_ref` moved to `am_field_spec`; the renderer was querying them from `am_field_arrangement`. This caused the blank-form rendering regression reported by the operator after Phase 1.

Projects listed as "untitled" and only 2 of 4 visible in the project list were traced to a heading fallback bug and a filter issue in `routes/projects.js`.

A sitepilot offsite backup pipeline was built in this session as a distinct work item. This is unrelated to the AgentEPC-origin canvas but was performed in the same session. The relay cron lives in the `johntdavenport` crontab on bombadil; if bombadil is rebuilt, the cron and SSH keys need re-adding.

## Status changes implied

No named canvas track/milestone slugs were referenced in the inputs (§7 canvas snapshot is empty). The following status changes are implied by the session:

UI sweep (Phase 1 + 2): moved from planned to complete — 17 route and lib files patched and deployed.

BRI0 reload: 3 accounts (615611, 693903, 694215) and 4 projects loaded with full FK hydration. CASCADE FK audit (mig 123) applied.

Coperniq batch pull (batches 1–23): moved from planned to active — driver running, batches 1+ confirmed pulling accounts into `AIEPC_PARITY_TEST` staging.

Backup pipeline: moved from missing/broken to active and verified — prod, bombadil, and sadb all wired; Resend alerting confirmed.

