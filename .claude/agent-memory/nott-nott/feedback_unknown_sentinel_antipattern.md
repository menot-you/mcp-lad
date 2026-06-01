---
name: 'unknown' sentinel anti-pattern in dual-state migrations
description: NOT NULL DEFAULT 'unknown' on dual-state columns conflates "never observed" with "explicit unknown phase". NULLable + SQL COALESCE is the clean root-cause fix.
type: feedback
---

When migrating to a dual-state model (legacy `state` + new `actual_state`), do NOT use `NOT NULL DEFAULT 'unknown'` on the new column. The 'unknown' sentinel ends up doing double duty:
- "V2 actuator has never observed this row" (never reconciled state)
- "V2 actuator explicitly observed phase=unknown" (transient runtime state)

This conflation forces application-layer disambiguation helpers (`display_state(actual, legacy)`) that DON'T agree with SQL predicates, creating cascade bugs in readers, writers, and tests.

**Why:** Observed in nott `actual_state` migration (`20260425_guild_containers_generation.sql`). Required full R1.1 + R1.2 mitigation rounds on C5 to undo:
1. Drop NOT NULL + DROP DEFAULT
2. Backfill `WHERE actual_state='unknown' AND last_reconciled_at IS NULL` to NULL
3. Add idempotent bootstrap path in `pool.rs::ensure_schema` for old-shape DBs
4. Switch readers to SQL `COALESCE(actual_state, state)` (clean — NULL is "defer to legacy")
5. Update writers that bump legacy state to ALSO clear actual_state to NULL (e.g., `update_state`, `mark_failed`) so subsequent reads pick the kernel-decided phase

**How to apply:**
1. In dual-state migration plans, default new column to NULL. NEVER `NOT NULL DEFAULT '<sentinel>'`.
2. Use SQL `COALESCE(new_col, legacy_col)` everywhere. App-layer helpers are technical debt waiting to bite.
3. Document the invariant: NULL means "never observed by new actuator, defer to legacy"; non-NULL means "explicit observed value".
4. Add a "writer asymmetry" test for every legacy writer: assert that bumping legacy state also clears new column to NULL.

**Real example (kernel-v2-redo 2026-05-11):**
- Initial migration shipped with 'unknown' default → 3 rounds of /lgtm caught the conflation cascade
- opus @dba long-term right call (R1) → NULLable + COALESCE → executor implemented cleanly in R1.1
- Codex caught writer-side asymmetry post-R1.1 → R1.2 closed: `update_state` and `mark_failed` now clear actual_state to NULL
