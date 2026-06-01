---
name: async codex post-commit reviews spiral
description: Async codex post-commit hooks consistently produce 2000-5000+ line investigations without final VERDICT line. Per-file pre-commit reviews complete reliably.
type: feedback
---

The async codex hook fires twice on commits:
- **Per-file pre-commit** (`peer-feedback/<ts>-codex-_path.md`) — completes reliably with severity-tagged findings + VERDICT line
- **Post-commit forensic** (`peer-feedback/<ts>-codex-postcommit-<sha>.md`) — spirals into 2000-5000+ line code investigation, NEVER produces VERDICT

**Why:** Codex CLI's post-commit forensic mode lacks the convergence signal — it grep/reads code indefinitely. The per-file pre-commit mode has a tighter prompt that produces concrete findings.

**How to apply:**
1. When checking async codex feedback files, **grep for `VERDICT:` and check file size**:
   - File ≤2000 lines AND has VERDICT line → reliable signal, parse findings
   - File >2000 lines AND no VERDICT line → spiraled, ignore
2. For convergence gates: dispatch explicit `/lgtm` Round instead of relying on async post-commit. The async signal is early warning at best.
3. Pre-commit per-file reviews are gold — they're the safety net before /lgtm Round 1.
4. Skip reading the full post-commit file. Use `wc -l` + `grep VERDICT:` to triage in 2 bash calls.

**Real example (kernel-v2-redo 2026-05-11):**
- C2 post-commit codex (2877 lines, no verdict) — spiraled
- C2 per-file codex (3919 lines but `VERDICT: BLOCK` at end with 3 findings) — usable
- C4 R1.1 per-file codex on container_manager.rs (266 LOC change) — completed cleanly
