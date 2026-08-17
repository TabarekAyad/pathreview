# PathReview Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/47

**Issue title:** Agent state isn't persisted across API restarts, causing in-progress reviews to be lost

**Tier:** [ ] Tier 1  [ ] Tier 2  [x] Tier 3

**Problem summary:**
The agent orchestrator holds in-progress review session state entirely in memory and only writes it to Redis upon completion. When the API server restarts mid-review — whether due to a crash, a deployment, or an intentional restart — any long-running review in flight is lost with no way to resume it. This affects the agent layer of the codebase, specifically how session state is managed during multi-repository reviews that can take several minutes. A successful fix would write agent state to Redis incrementally throughout execution, so that a restarted server can recover and continue an in-progress review rather than dropping it.

**Branch name:** fix/47-persist-agent-state

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

## "Is This Issue Right for Me?" — Checklist Reasoning

### Part 1 — Understanding the Issue

**Can I explain what this issue is asking for in my own words?**
Yes. If the API restarts mid-run, the in-flight results are lost because nothing was written to Redis yet. The fix is to persist state incrementally after each tool completes, and skip already-completed tools on a restart.

**Do I understand which part of the app is affected?**
Yes. The issue lives in `agent/orchestrator.py` and `agent/memory/session_store.py` as noted. The Redis infrastructure is in place. The labels confirm this is in the `agent` scope.

**Do I understand what "done" looks like?**
Before: a review of 5+ repos starts, the server restarts mid-run, all progress is lost and the user gets nothing.
After: each tool's result is written to Redis immediately after it completes; on restart loads the existing session state, skips already-completed tools, and continues from where it left off.

---

### Part 2 — Tier Fit

This is a **Tier 3** issue — it requires understanding the broader session lifecycle across the API. I chose it because it requires changes across multiple modules in the agent layer, which is something that triggered my curiousity.

---

### Part 3 — Codebase Readiness

I used Claude Code to navigate the codebase for this section — it helped me locate the relevant files, read `orchestrator.py` and `session_store.py`, and identify the exact lines where the bug lives before I read them myself.

**Can I find the relevant code?**
Yes. They are in `orchestrator.py` — the tool execution loop and the single end-of-run `session_store.set()` call. The fix point is clear.

**Do I understand the surrounding code well enough to change it safely?**
Yes. The loop iterates `(tool_name, tool_input)` pairs, appends to `results`, then saves once. Moving the `set()` call inside the loop and adding a skip-if-already-done check at the top are the two changes needed, with no impact on the tool execution logic itself.

**Have I read the relevant test file?**
There is no `test_orchestrator.py`. The closest existing test is `tests/unit/test_review_service.py`. Writing a new test for incremental persistence using a mock Redis client will be part of the deliverable.

---

### Part 4 — Scope and Time

**How many others are already working on this issue?**
Checked the cohort ledger — issue is still open. And claim count is 10, which should be acceptable.

**Is the scope realistic for Weeks 8–9?**
Yes. The core change is small (moving one call inside a loop and adding a resume check), but writing tests against a mock Redis client and handling edge cases (partial state, TTL, tool failures mid-run) adds time. Estimated 8–16 hours total.

**Are there any blockers or dependencies?**
No open blockers or dependent issues referenced in #47.

---

## Week 8 - Reproduction & solution planning

**Reproduction commit link:** [d13369f](https://github.com/TabarekAyad/pathreview/commit/d13369fcf37c1a6bb7cda42b7c3048e37111f3c6)

**Reproduction summary:**
I used Claude Code to write the reproduction tests in `tests/unit/test_orchestrator.py`, understand the mypy pre-commit hook errors, and draft PLAN.md.


Added two failing unit tests in `tests/unit/test_orchestrator.py` that directly demonstrate the bug: the first confirms that `session_store.set()` is only called once (at the end of the loop) instead of after each tool, and the second confirms that already-completed tools stored in Redis are re-run unconditionally on restart instead of being skipped. Both tests fail against the current code, confirming the issue is real and exactly located in `agent/orchestrator.py` lines 52–67.

**PLAN.md link:** [PLAN.md](https://github.com/TabarekAyad/pathreview/blob/fix/47-persist-agent-state/PLAN.md)

**Walkthrough video (recommended):** [not recorded]

**Blockers or open questions:**
`_run_agent_orchestration` in `review_service.py` is a stub that never calls `Orchestrator` — need to decide how deeply to wire the fix during Week 9 without scope-creeping into replacing the stub entirely.