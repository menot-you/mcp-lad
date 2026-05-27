---
name: lgtm partial-mitigation cascade
description: Every /lgtm Round closure can introduce its own regressions. Plan for cascade rounds (R1.5 → R1.6 → R1.7) on non-trivial cutovers.
type: feedback
---

When running `/lgtm` on complex commits (cutovers, schema migrations, actuator implementations), expect 2-4 mitigation rounds before convergence. Each fix can introduce its own narrower regression.

**Why:** Empirically observed across PR 1 (V2 actuator), C2 (K8s real), C5 (dual-state reader), C4 (kernel cutover). Pattern:
- R1 surfaces 3-7 HIGH findings
- R1.5 fix introduces 2-3 narrower findings
- R1.6 fix introduces 1-2 more
- R1.7+ converges

Real examples (kernel-v2-redo, 2026-05-11):
- PR 1 R1 → R1.5 → R1.6 (async codex caught sandbox plugin mount drop) → R1.7 (worker ownership race)
- C2 R1 → R1.1 (4/4 LGTM converged)
- C5 R1 → R1.1 (NULLable migration root-cause) → R1.2 (writer-side asymmetry codex caught after read-side fix)
- C4 R1 → R1.1 (gen idempotency + poll match + boot validate; CRITICALs from codex)

**How to apply:**
1. Don't promise "1 round and ship" on cutover-class commits. Budget 3 rounds wall-clock.
2. After each mitigation, dispatch full /lgtm round again. Don't skip "just to save tokens" — partial-mitigation cascade is the dominant cost driver.
3. When opus says LGTM with soft conditions but codex BLOCKs, trust codex on concurrency/race. Opus misses cross-call concurrency analysis under load.
4. Each round's findings get NARROWER and harder to spot. R1 finds architectural class bugs (e.g., "no sandbox parity"). R2+ finds specific writer-side asymmetries.
5. When you hit 4+ rounds without convergence, switch to advisor-consult `/nott:selfreview` for blind-spot coverage rather than another mitigation cycle.
