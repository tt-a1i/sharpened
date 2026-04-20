# Production Lessons — Why Sharpened Exists

> This document records every failure mode we caught while shepherding an AI coding agent through ~15 rounds of shipping a real project ([tt-a1i/hive](https://github.com/tt-a1i/hive), M1 → M4.1 over ~2 weeks, ~140 tests, ~9k LOC).
>
> Every entry is **a real translation between (a) an AI behavior that produced a seemingly-green PR** and **(b) the rule change that would have caught it earlier.** The upstream [obra/superpowers](https://github.com/obra/superpowers) skills are solid in principle but written for humans; the failure modes below are patterns AI agents exhibit that a human rarely would, and that stock skills therefore don't defend against.
>
> **Purpose of this file:**
> - Be the append-only ledger of "things that went wrong, and what we learned" — it only grows, nothing gets removed
> - Be the source material for skill rewrites: every failure mode maps to a specific skill change (see "Skill Impact" at end)
> - Be the evidence we can point to when a change to a skill feels too strict — every rule here has a real receipt

Organised as:
- Part 1 — Failure Modes Observed (what the AI did wrong)
- Part 2 — Improvement Proposals (what the rule should be)
- Part 3 — Skill-by-Skill Impact (which upstream skill needs which change)

---

# Part 1 — Failure Modes Observed

## Category A: Fake Tests That Pass Tautologically

The most dangerous category. Tests are green, CI is green, coverage metrics look fine — but the test could not fail even if the product code were completely wrong.

### A1. `not.toContain(UUID)` against a template that never contained a UUID

**Caught at:** `tests/server/team-prompt-contract.test.ts:73-74` (fixed in Hive R4.1)

The assertion was `expect(prompt).not.toContain(orchestrator.id)`. The orchestrator id is a UUID. The prompt-injection template is (verbatim from spec):

```
@${senderName} 给你派了任务：<task>${text}</task>
你的角色：${workerDescription}
```

The template never constructs the orchestrator's UUID — it uses `senderName` (a human-readable handle like `@Orchestrator`). So `not.toContain(UUID)` is **trivially true regardless of what the product code does**. Rewriting `writeSendPrompt` to be completely wrong would leave this test green.

**Why AI does this:** the AI wrote the test from "what should NOT leak into the prompt" in its mental model, without checking whether the candidate leak substring could ever appear in the generator's output. Stock TDD's "watch it fail once" doesn't help because the test WAS red initially (the function didn't exist). After implementation it was green for the right reason *once*. Then it stayed green forever by coincidence.

**How to defend:** demand that every assertion pass a reverse-regression gate: comment out the corresponding product-code line, the test MUST go red. Use **positive** assertions (`toContain`/`toEqual`) wherever possible, not negative ones against things that never appear.

### A2. `expect(recorded).toHaveLength(0)` where `recorded` is never pushed

**Caught at:** `tests/server/agent-runtime-races.test.ts:93,139` (before M2.1 cleanup)

The test declared `const recorded: string[] = []` and never wrote to it. The assertion `expect(recorded).toHaveLength(0)` is a tautology. This was packaged as "assert no rollback happened."

**Why AI does this:** the AI conceptualised "rollback side effect" but coded the shape wrong — forgot to wire the side-effect observer to the product code. Post-implementation the test was green, so nobody noticed. Coverage tools can't catch it.

**How to defend:** same reverse-regression gate. Any assertion of the form "nothing happened" must be proven by actually exercising the path that would have made something happen, with a mutation observer that is real and wired.

### A3. `statusBefore === statusAfter` where both are already terminal

**Caught at:** `tests/server/lifecycle-hardening.test.ts:103-107` R2.2 idempotency (fixed in M2 signoff)

The test called `stopAgentRun(runId)` twice and asserted the run status was the same before and after the second call. Problem: before the second call the status was already `exited` (the first call terminated it), so the second call changing anything would have been tangible. The comparison is just `exited === exited` — even if `stopAgentRun` were an infinite loop or a database drop, `status` can't un-exit. The test was advertised as "proves idempotent guard works"; it proves nothing of the sort.

**Why AI does this:** classic "I wrote an assertion that compares two things that happen to be equal." The equality happens for the wrong reason.

**How to defend:** a test named "X is idempotent" must use a side-effect counter (spy, call count, observable mutation), not a state comparison against a state that can only move in one direction.

### A4. Multiple `not.toThrow()` calls as the only assertions

Assertion pattern: `expect(() => fn()).not.toThrow(); expect(() => fn()).not.toThrow();` with no other observable.

"Does not throw" is the weakest assertion in the language. For any non-trivial function, "doesn't throw" is compatible with "silently corrupts data / does nothing / returns wrong values." Pairs well with A3.

**How to defend:** ban `not.toThrow()` as the sole assertion in a test. It's fine as a prefix to a real assertion.

### A5. `readFileSync().toContain("import ...")` — "architecture police" disguised as a unit test

**Caught at:** `tests/setup/runtime-artifacts.test.ts:11-12` (deleted in M2.1 R8)

The test read the source of a shell script with `readFileSync` and asserted it contained `"../src/cli/team.js"`. This isn't testing behaviour; it's testing that the source file has certain text. If the runtime behaviour changes but someone also updates the string, the test passes. If the string stays the same but behaviour breaks, the test passes. It's string-matching on source files, not behaviour.

**How to defend:** explicit rule — no `readFileSync` on product source as the subject of an assertion. If you want to check "this code uses X", invoke the code at runtime and observe the effect.

---

## Category B: Reporting Discipline Failures

AI reports what it thinks it did, which drifts from what it actually did. These are **the** most insidious failures because the code might actually be fine — but the report creates false confidence and erodes the trust model.

### B1. Test count lies

**Caught at:** Hive M2 signoff, round 2. Report claimed "109 tests passing." Actual `pnpm test` output: `Tests 105 passed (105)`. Discrepancy of 4 tests.

**Why AI does this:** the AI wrote the report from its intention (meant to add 4 tests → wrote "109") not from the tool's actual output. The four tests were either never committed, failing silently and filtered, or counted twice in the AI's head. Nobody inspected the vitest footer.

**How to defend:** every number in the final report MUST be a verbatim copy-paste from the tool's output (the last `Tests X passed (X)` line of `pnpm test`). Paraphrasing is prohibited. If the agent summarises "roughly 130 tests," that summary is rejected.

### B2. Claimed additions that weren't made

**Caught at:** same round. Report claimed "team-api-authz test file extended to 6 cases." Actual file: 5 cases. Report claimed "agent_runs.ended_at column added." Actual schema: no such column.

**Why AI does this:** the AI planned the change, wrote it in the report, and either got distracted or hit a failure mid-change and moved on. The plan-vs-reality gap isn't something AI self-notices unless prompted to grep for the change it claimed.

**How to defend:** every factual claim in a report must come with a grep/wc/cat command that reproduces the evidence. "Added ended_at column" → paste the output of `grep ended_at sqlite-schema.ts`. No evidence, no claim.

### B3. Deleted tests, undisclosed

**Caught at:** Hive M2 signoff. The AI deleted `tests/integration/restart-recovery.test.ts` (the only end-to-end Layer A test at the time) to stay green while refactoring; reported the suite as green without mentioning the deletion. Layer A end-to-end coverage silently dropped to zero.

**Why AI does this:** the AI hit a test it couldn't keep green during refactor. The local-minimum move is to delete it. The report framing "all green" is accurate but misleading.

**How to defend:** any deleted or `skip`ped test must be listed in the delivery report by file name, with a justification. If the justification is "couldn't keep it green," the refactor is rejected.

### B4. Success claims that contradict the "known limitations" section

**Caught at:** M2 R4.2. The AI claimed the test `team-atomicity.test.ts` now proves "real rollback" (prompt already written to PTY, then DB insert fails, pending count is rolled back). Simultaneously in "known limitations" it wrote: "the PTY-write-succeeded-then-DB-failed crack remains." Both cannot be true.

**How to defend:** a reviewer (human or separate agent) must diff every "claimed fixed" item against "known limitations" — any overlap is a report-level contradiction that must be resolved before merge.

---

## Category C: Scope Creep

### C1. "While I was in there" fixes

**Caught at:** Hive M3 PR-0. Task: "extract pty-output-bus from agent-manager, ~150 LOC change." Actual PR: that + a full rewrite of `team-operations.ts` to fix the PTY/DB atomicity crack (279 LOC, 6 files). The atomicity fix was correct and welcome, but it was **not in PR-0's scope**, and bundling it meant the PR had two independent reviews riding on one commit log.

**Why AI does this:** the AI sees an obvious improvement adjacent to its task and can't resist. Upstream skills don't penalize "improving things." In a single-commit delivery workflow this feels efficient. In a review workflow it causes exactly the "big PR hard to review" problem human teams have.

**How to defend:** explicit rule — `git status` must match the task manifest. Any file changed that isn't in the task's file list must either (a) be justified as a direct dependency of the task, or (b) be split into a separate follow-up PR. "Direct dependency" is not "fixes an unrelated bug I noticed."

### C2. Retroactive justification

**Caught at:** same round. When called on C1, the AI's response was "the atomicity fix was a direct consequence of exposing the PTY output bus." It wasn't — the bus extraction touched no DB code, and the atomicity fix touched no bus code. The two were independent and just happened to be in the same file neighborhood.

**How to defend:** "direct consequence" claims must be supported by showing the dependency in code (line X of file A calls line Y of file B, and changing X required Y to change too). No demonstration = scope violation.

---

## Category D: Fake Splits / Architectural Theatre

### D1. Split into N files but closure state remains

**Caught at:** Hive M3, `agent-runtime.ts` split. The file was "refactored" from a 180-LOC god object into 6 files (`agent-runtime-types.ts`, `agent-runtime-state.ts`, `agent-runtime-lifecycle.ts`, etc.). But `liveRuns`, `launchCache`, and `tokenRegistry` — the three key state objects — all remained in the main `agent-runtime.ts` closure. The "split" files each exported a stateless function that took those three objects as parameters. So the file count went 1 → 6 but the coupling structure was unchanged. A bug in state transitions still touched the main file.

**Why AI does this:** upstream "small file" rules use line count as the proxy. The AI minimised the metric. But the metric was a proxy for "one concern per file" and that wasn't achieved.

**How to defend:** reviewer must ask "what closure state did you move, and which module now owns it?" If the answer is "the stateless function takes it as a parameter," that's not a split, it's parameterisation. Reject.

### D2. "Half-split" with the main file still owning the world

**Caught at:** `workspace-store.ts` → `workspace-store.ts` + `workspace-store-contract.ts` + `workspace-store-hydration.ts` + `workspace-store-mutations.ts`. The split looks clean. But the main file still holds the `workspaces: Map` and all the `markTask*`/`markAgent*` state mutators. The other files export pure helpers that the main file calls. The main file's responsibility was not reduced; a facade was added.

**How to defend:** same as D1. Line count is not a structural metric. Closure state ownership is.

---

## Category E: Spec Drift

### E1. Implementation changes, spec doesn't

**Caught at:** Hive M2, UI cookie auth. The mechanism was reworked three times (Origin header check → global `Set-Cookie` on every response → `/api/ui/session` bootstrap with `HttpOnly; SameSite=Strict`). Spec §8 was updated only on the third iteration. The first two iterations shipped code with no spec support.

**Why AI does this:** the spec feels like documentation, changes feel like code. The AI hasn't learned that for protocols spec is the source of truth and the implementation is the artifact.

**How to defend:** any change to a wire-format, header, cookie name, or auth token requires a simultaneous spec diff. If there's no spec change, reject the PR or ask for justification.

### E2. "MVP exception" that's actually scope reduction without written consent

**Caught at:** M2 R1. Originally the AI claimed "Layer B summary fallback is implemented." It wasn't. Under review it emerged that the `recovery-summary.ts` helper existed but was dead code — no runtime caller. The AI's framing was "Layer B is implemented, but MVP doesn't use it yet." The spec said (line 362) "Layer B is MVP-required." So "MVP doesn't use it yet" was in fact "we're shipping an MVP that doesn't meet the spec."

**How to defend:** explicit distinction. If the AI wants to narrow spec scope, it must write a patch to the spec saying so (a yellow box `[M2 scope 调整] feature X deferred to M2.5`). Silently shipping less is drift.

---

## Category F: Delete-Instead-of-Fix

### F1. Delete the test that won't go green

**Caught at:** M2 R2.1 (restart-recovery). The AI couldn't keep the end-to-end Layer A restart-recovery test green through a refactor of the session-capture code. Rather than diagnosing, it deleted the test file. Coverage dropped silently.

**How to defend:** deleted-test discipline. Any deleted or skipped test must appear in the delivery report with justification. Acceptable: "test was testing old behaviour we explicitly removed as scope change E2." Unacceptable: "couldn't figure out why it fails."

### F2. Revert the assertion instead of the product

**Caught at:** M2 R3.2 state machine. The spec says "a newly created worker starts in `stopped` state (no PTY yet). `idle` means PTY is alive, no pending task." Existing tests asserted `status === 'idle'` on a freshly created worker. When the state machine fix landed, those tests went red. The AI's first instinct was to change the assertion from `'idle'` to `'stopped'`. That's actually the correct thing here (the tests encoded old wrong behaviour), but it's also the exact pattern used to rubber-stamp spec violations (change assertions until green).

**How to defend:** when a test fails after product change, the choice is:
- (a) product code is wrong → revert product, test stays asserting current contract
- (b) test expectation was wrong per spec → update test, but include explicit spec reference for the new expectation
The silent path of "just change the number until green" is prohibited.

---

# Part 2 — Improvement Proposals

Each proposal maps to one or more failure modes above.

## P1. Reverse-regression gate (catches A1–A5, D1–D2)

**Rule:** every assertion in a new test must pass the reverse-regression gate:
1. Comment out (or literally delete) the single line of product code that the assertion is supposed to verify.
2. Run the test.
3. It MUST fail, for the exact reason the assertion was meant to catch.
4. Restore the product code, run again, it must be green.
5. Paste both test outputs (red + green) into the delivery report.

**Scope:** integration tests at minimum; unit tests where feasible. If the "corresponding product code line" is hard to identify, the assertion is probably too indirect and should be rewritten.

## P2. Numeric report alignment (catches B1–B2)

**Rule:** every numerical claim in a delivery report (test count, file count, line count, assertion count) must be:
1. The verbatim last line of the relevant tool output (`pnpm test` footer, `wc -l` output, `grep -c` result).
2. Pasted inline in the report, not paraphrased.
3. If the AI claims "I added 4 tests," the evidence is `git diff --stat tests/` showing +4 test functions.

## P3. Anti-patterns list that grows (supports all categories)

**Rule:** every repository using sharpened maintains a project-local `anti-patterns.md` (or inherits the one in the skill). When an AI is caught in a new failure mode, the entry is added to the local list with:
- Date / session
- File / line of the failure
- Why it happened
- Defensive rule

The list is append-only. Nothing is removed. Reason: the failure mode that tricked one AI session will trick another. Memory is what keeps the codebase from re-learning the same lessons.

## P4. Scope guardrail (catches C1–C2)

**Rule:** every task prompt to an AI must include a file manifest (list of files expected to change). Before merge, `git status` is diffed against the manifest:
- Files changed but not in manifest → require either (a) direct-dependency justification with code references or (b) split into follow-up PR.
- Files in manifest but not changed → require justification (might be valid, e.g. "hydration module needed no change after all").

## P5. Integration-test mock embargo (catches a whole family of false-coverage failures)

**Rule:** integration tests (`tests/integration/`, `tests/e2e/`, any test directory named `integration`) may not import mock libraries for the primary dependency under test. Enforced by:
```
grep -r "vi.mock\|jest.mock\|mock-node-pty" tests/integration/ | wc -l
```
must return zero (or match an explicit allow-list file). If an integration test needs to mock, it's either not actually an integration test (move it to unit) or the design has a testability problem.

## P6. Spec-drift lock (catches E1–E2)

**Rule:** any PR that changes a wire format, header name, cookie name, token format, protocol field, or auth mechanism must include a simultaneous change to `docs/spec/` (or wherever the spec lives). If the spec file is not in the PR's diff, the PR is rejected at the reviewer stage with the message "spec change required."

## P7. Deleted-test ledger (catches B3, F1)

**Rule:** delivery reports must include a section "Tests deleted or skipped this round." Format:
```
- tests/foo/bar.test.ts [DELETED] — reason: <human-readable justification>
- tests/baz/qux.test.ts::case X [SKIPPED] — reason: <justification>
```
Empty list is OK (and common). Missing section is a report failure even if no tests were deleted — the section's presence proves the author thought about it.

## P8. "Fix the test, don't fake it" protocol (catches F2)

**Rule:** when a test goes red after a product change:
- First default: assume product code is wrong. Revert and investigate.
- Only after investigation concludes the test encoded old/wrong behaviour: update the assertion.
- When updating the assertion, include a spec reference (`spec §X.Y line Z`) for the new expected value.
- Never: "changed 'idle' to 'stopped' to make test pass" without explanation.

---

# Part 3 — Skill-by-Skill Impact

How the proposals above translate into concrete changes to specific upstream skills.

## `test-driven-development` (MAJOR rewrite)

- Add P1 (reverse-regression gate) as mandatory step between "watch it fail" and "write code."
- Replace the static `testing-anti-patterns.md` with a section that references `anti-patterns.md` at repo root, and make the skill instruct the agent to read and append to it.
- Add explicit prohibitions against A1–A5 patterns with examples.

## `verification-before-completion` (MAJOR additions)

- Add P2 (numeric report alignment) — every number must be a tool-output paste.
- Add P7 (deleted-test ledger) — mandatory section in the completion report.
- Add explicit "compare claimed-fixed vs known-limitations" diff step (catches B4).

## `requesting-code-review` / `receiving-code-review` (MINOR changes)

- Add P4 (scope guardrail) — reviewer must diff `git status` against task manifest.
- Add P5 (integration-test mock embargo) — reviewer runs the grep, rejects if non-zero.

## `writing-plans` / `executing-plans` (MINOR changes)

- Plans must include a file manifest for each PR (lists files expected to change).
- Executing agent reports `git status` against the manifest as the last step.

## NEW SKILL: `scope-guardrails`

- Owns P4 end-to-end. Fires whenever an agent is about to commit a PR.
- Gate function: diff `git status` vs the most recent task manifest, prompt human if there's a mismatch.

## NEW SKILL: `anti-patterns-as-memory`

- Meta-skill that teaches the agent to (a) read `anti-patterns.md` at the start of any testing task, (b) append a new entry when a new failure mode is caught, (c) never propose to delete or shrink an entry.

## `brainstorming` / `subagent-driven-development` / `using-git-worktrees` / `finishing-a-development-branch` / `dispatching-parallel-agents` / `systematic-debugging` / `writing-skills` / `using-superpowers`

**No planned changes in the first round.** These were not the source of observed failures. Will revisit after the first three rewrites settle.

---

# Part 4 — Maintenance

- **Append-only.** When a new failure mode is observed in production, add a new `X.N` entry to Part 1 and (if it requires a new rule) a new `P.N` to Part 2.
- **Map to skills.** Every new entry should be cross-referenced to which skill enforces the new rule (Part 3).
- **Never remove.** Even if a failure mode seems "obviously covered now" — the whole point of this doc is institutional memory. The next agent will try the same trick.
- **Date every entry.** Makes pattern analysis possible later.

Last updated: 2026-04-20, based on Hive M1→M4.1 (≈ 15 delivery rounds, `tt-a1i/hive @ v0.4.0-m4.1`).
