---
name: autonom
description: Fully scope a task via interactive dropdowns, then work AUTONOMOUSLY NON-STOP until it is complete — self-pacing across turns (ScheduleWakeup), consulting a web-LLM advisor often for help, advice, optimization, and new angles. Use on "/autonom", "autonom", "work autonomously on", "non-stop until done". Combines front-loaded scoping + a self-paced loop + web-lane advisory.
---

# /autonom — scope-then-grind: ask everything, then finish it solo

A Claude Code skill. **Scope first via dropdowns, then run unattended to a machine-checkable DONE.**
NOT a one-shot Q&A, NOT a fixed recurring tick — `/autonom` front-loads every outcome-determining
question, confirms a *checkable* definition of done, then drives an autonomous self-paced loop that
runs until the task is genuinely complete, pulling a web-LLM advisor in each iteration.

> **Note on rule references.** Lines below cite the author's personal Claude Code ruleset
> (`BOGDAN.*`, `HARD RULE n`). They are NOT required to use the skill — treat each as a named
> convention and map it to your own house rules (e.g. "BOGDAN.3 = always ask via a dropdown,
> never a free-text question"). The design is fully standalone.

## Contract
1. **SCOPE FIRST (ASK).** Before any work, ask the user — via dropdowns, ONE question per call —
   every outcome-determining question until nothing ambiguous remains: goal, definition-of-DONE
   (the explicit completion signal), constraints, scope boundaries, files/systems in play,
   acceptance criteria, what NOT to touch, the **pre-authorized dangerous action-classes** (which of push / delete / publish / install / paid the run may do unattended — approved ONCE here so the run never interrupts later), and the **budget** (max turns / tokens / $). Ask as
   MANY as needed. Do not start until unambiguous.
2. **CONFIRM THE SPEC + ENFORCE CHECKABLE DONE (design-time guard — sharpest single lever).** Echo
   the full scoped spec in one block. The **DONE condition MUST be machine-checkable assertions**
   (test rc=0 + assertion-count>0 / command returns X / file matches regex|AST / endpoint returns
   status+body). **REFUSE to start if any DONE criterion isn't checkable** — bounce a concrete
   counter-proposal to the user ("Done when `/health` returns 200 + body `{status:ok}`?"). Then one
   final dropdown: `Start autonomous run? [Start / Adjust scope / Cancel]`.
3. **PRE-FLIGHT CHECK (at Start, before turn 1).** Validate the environment supports the plan:
   required CLI tools installed · spec-named files exist · working tree clean · required env vars set ·
   the DONE test command actually runs. Failing fast here is 100× cheaper than discovering it on turn 5.
4. **WORK AUTONOMOUSLY, NON-STOP.** After "Start", DO NOT ask the user. Drive to completion across
   as many turns as needed. No per-step gates, no check-ins. Mechanical safety conditions (§Reference) write
   `.autonom/ABORT` and stop — they NEVER prompt.
5. **CONSULT A WEB-LLM ADVISOR OFTEN (scope-bounded, quarantined).** Each iteration, use a web-LLM
   for advice/optimization/new angles — especially when stuck or a result looks suboptimal.
   **Prefix every autonomous advisory prompt** with: *"ADVISORY ONLY — do NOT suggest work outside
   this DONE condition: <condition>. If your advice would expand scope, say OUT-OF-SCOPE."*
   **Quarantine code advice:** advisor code is NEVER applied raw — read it line-by-line, write your
   OWN version, commit only yours; log the draft→applied diff to `applied_advice`. Throttle: cap
   advisory calls (default 5/run) and skip alternate turns if the adopt-rate < 20% over the last 10.
6. **SELF-PACE THE LOOP.** At end of each not-yet-DONE turn, write state then schedule a wakeup.
   The wakeup `prompt` is a FUSE only — `resume /autonom (.autonom/state.json)` — NEVER the spec
   (prompts truncate; the spec lives in state, §State). delaySeconds per the table in §Robustness.
   If a tick is gated on an external event, arm a tracked Monitor instead of a wakeup.
7. **STOP CONDITION.** When DONE: **re-verify ALL acceptance criteria** (not just the last substep)
   via an AUTHORITATIVE signal; if any criterion fails → un-DONE, `git reset --hard` to last snapshot,
   append the failing criterion as a new todo, continue the loop. When all pass → emit the DONE-evidence
   block, cancel ALL tracked Monitors, do NOT reschedule, report. Also stop on the kill switch / interrupt.

## Initialization + design gates
- **Manifest-only init.** If handed a `task_manifest.json` (e.g. from a scoping skill like `/askme`), that JSON
  is the ONLY init payload — `{goal, verification, guardrails, state_schema, out_of_scope, executor}`. Reject
  prose-only init; if no manifest exists, run the §Contract scope to PRODUCE one before turn 1.
- **Classify DONE + dead-man switch.** Tag the DONE check as `deterministic` (cmd rc=0 + optional output regex),
  `fuzzy` (AI-judge rubric + threshold + a reject example), or `browser` (HAR / accessibility-tree baseline diff).
  If `done?` fails 3× with DIFFERENT outputs (not one stable error) → ABORT and report; do not ping-pong.
- **Split `done?` from `allowed?`.** Two distinct predicates: a `## Done criteria` block (observable success,
  checked POST-action) and a `## Guardrails` block (action allow/deny: forbidden paths/commands/actions, checked
  PRE-action). Run `allowed?(action)` before EVERY tool call and `done?(state)` after each step. A state whose only
  progressing action is disallowed → escalate, never silently treat as done.
- **Goal-derived state schema.** Do not force one fixed `state.json` shape — derive the keys from (intent,
  done_criteria) at turn 1, store as `state_schema`, read/write per it (a PR-babysitter tracks open PRs + last SHA;
  a migration tracks files done/remaining).
- **Plan minimizer.** Each turn, before acting: "what is the MINIMAL next action that yields observable progress?"
  Do only that, then re-evaluate. Stops 20-step plans where one step suffices.
- **Shadow-execution mode (optional).** For repo-scoped tasks, `git worktree add .autonom/shadow HEAD` at turn 1
  and run ALL mutations in the shadow worktree. On full DONE verification → merge shadow to main; on verify-fail →
  discard the worktree (instant, zero `git reset --hard` risk to the user's real tree). An ephemeral venv/container
  per task extends this to dependency + OS-state isolation (nuke on fail/out-of-scope).
  CONCRETELY (else it's aspirational): at turn 1 set `GIT_DIR=.autonom/shadow/.git` + `GIT_WORK_TREE=.autonom/shadow`
  for the session and route every edited path through the shadow root. If you can't commit to that remapping, treat
  shadow-exec as EXPERIMENTAL and do NOT claim repo isolation for this run.

## Wrong-success guards (the real risk is confident wrong-success, not loops)
The no-interrupt model's dominant failure is "the agent successfully does the WRONG thing and still passes its own
machine-checkable DONE." These four guards target exactly that:
- **Assumption budget (CLARIFY-FIRST safety).** Every self-answer to a missing fact is recorded as a FIRST-CLASS
  assumption in `assumptions[]` — never silently absorbed. Cap `assumption_budget` (default 3). If a CRITICAL-path
  decision (target env, remote, account, dataset, prod-vs-staging) rests on an UNVALIDATED assumption → DONE FAILS
  until it is validated by a cheap check, or — if unvalidatable from context — it is the ONE thing surfaced to the
  user. Budget exceeded → halt + report.
- **FALSE_DONE checklist (machine-checkable ≠ correct).** Before accepting DONE, run a FIXED 6-pattern check — NOT
  open-ended self-generation (a biased judge writes weak tests it can beat). Confirm NONE apply: (1) test/check edited
  or stubbed instead of the code; (2) asserted on a stale / pre-existing fixture; (3) wrong environment or target;
  (4) mocked the thing under test; (5) check passes on pre-existing state, not on YOUR change; (6) partial-DONE — one
  criterion passes, others silently skipped. Any pattern applies → un-DONE, continue.
- **Mutation-surface classification (shadow ≠ global safety).** The shadow worktree protects ONLY repo/filesystem state.
  Tag every mutating action by surface — `repo · filesystem · network · cloud · SaaS/external-API · comms` — and require
  an isolation/rollback strategy PER surface (shadow for repo; dry-run/staging target or scoped pre-auth for
  network/cloud/SaaS/comms — `git reset` does NOT undo these). **For every external-surface mutation, PRE-REGISTER a
  compensating/rollback action before executing; on ABORT run the compensations in reverse order** (repo is covered by
  `git reset`; external surfaces are not). An action on an unisolated external surface without a
  pre-authorized scope is `necessary?`-routed (FAIL if required for DONE).
- **Pre-run design review (critical-assumption + blast-radius).** Before turn 1, self-review the plan and NAME the single
  `critical_assumption` (what, if wrong, ruins the whole run) + the `blast_radius` (worst-case surfaces touched). Both go
  in state; the critical_assumption is auto-added to `assumptions[]` for validation.

## State (single source of truth — atomic)
Maintain `.autonom/state.json` from turn 1. Schema:
`{task, done_condition, spec, scope_boundary, turn, last_updated, budget:{turns_max,tokens_max,usd_max,
spent, web_lane_calls}, todos[], attempts:{intent_hash:n}, seen_errors:[hashes], applied_advice:[],
deferred_ideas:[{idea,deferred_at}], monitors_armed:[{id,purpose,armed_at_turn}], intent,
pending_gate_resolution, wakeup_failed, last_verified, assumptions:[{q,self_answer,critical,validated}],
assumption_budget, critical_assumption, blast_radius, mutation_surfaces}`.
- **Atomic writes ONLY:** write `state.json.tmp` → fsync → `os.replace` (never a partial file). Same for the `ABORT` sentinel.
- **Spec re-read from STATE at turn top, never from the prompt** (the prompt is a fuse).
- **Compaction:** `seen_errors` + `applied_advice` FIFO-evict at 100; `deferred_ideas` SPLIT into `deferred_required` (NEVER evicted or expired) vs `deferred_optional` (cap 10, drop oldest + auto-expire 24h); at DONE surface top-3 optional AND every `deferred_required` prominently.
- **Cross-run error journal** `.autonom/error_journal.json` (PERSISTS across invocations): `{error_hash: {recipe, fixed_n}}`. Turn-1 reads it and auto-applies known fixes (e.g. "port in use", "missing env var") instead of re-discovering them every run.

## Reference — robustness detail (read ON a GUARD-phase flag, NOT every turn; ranked by impact)
1. **Stuck/loop detection — distinguish STUCK from PRODUCTIVE ITERATION.** Key the STOP counter on the
   **(intent + normalized_error/diagnostic)-hash** `sha256(intent + normalized_error)`, NOT the bare
   intent-hash. **INCREMENT** the strike count only when an attempt repeats a PRIOR (intent,error)
   signature (same failure, ZERO new information). **RESET that intent's strike count to 0** whenever
   an attempt surfaces a NEW error-hash, a new diagnostic fact, or a new hypothesis — that is productive
   iteration, NOT a strike. **STOP → write `.autonom/ABORT` + report at end (no dropdown), do NOT reschedule, ONLY when the SAME
   (intent,error) signature recurs 3×.** Backstops: (a) HARD cap `N_total ≤ 8` attempts per bare intent
   → escalate; (b) cycle-detection — if the last 4 DISTINCT error-hashes form a repeating set, treat as
   stuck even if each looks "new". No-progress detector: 3 turns with zero todo delta AND zero new
   error-hash → halt. **HYPOTHESIS forcing-function** — after each failure emit a normalized
   `HYPOTHESIS:` (one line: WHY the next attempt will differ); strike on the `(intent+hypothesis)`
   fingerprint with `MAX_STRIKES_PER_STRATEGY=2`; if it cannot produce a hypothesis DISTINCT from all
   prior ones → escalate. **Per-intent resource bound** — `MAX_WALL_TIME_PER_INTENT=300s` +
   `MAX_TOKENS_PER_INTENT=100k` → escalate on either, independent of strike caps.
2. **Min-viable-progress/turn.** Every turn MUST yield ≥1 of {todo delta · file edit · git commit ·
   DONE report} — else it's a wasted spin → write `.autonom/ABORT` with a "stuck, no progress" diagnostic.
3. **Budget cap.** Track `spent` + `web_lane_calls`; report % each turn (in the delta header); **HARD-STOP
   when any cap hit.** Advisory calls are $0 API but burn orchestration tokens — count them against the cap.
4. **Infeasibility threshold.** Halt + report "task may be infeasible — recommend re-scoping" when
   `count(intents with same-(intent,error) attempts≥3) ≥ 3` OR (`budget.spent > 75%` AND `todos.done/total < 50%`).
5. **Kill switch.** Check `.autonom/ABORT` at the TOP of every turn → stop immediately, cancel Monitors, report.
6. **Failure taxonomy + tag rollback.** Classify each verify-fail: transient (retry w/ capped backoff) ·
   logical (re-plan substep) · environmental (write `.autonom/ABORT` + report) · scope (defer). **Before any mutating substep:
   `git tag autonom/pre-<intent_hash>` on HEAD; on verify-fail `git reset --hard autonom/pre-<intent_hash>`;
   sweep tags every 10 successful steps.**
7. **Idempotency + crash recovery.** Write `intent` BEFORE acting, clear after. Turn-top: if `intent` set +
   incomplete → wakeup double-fired or crash → resume/skip, never double-execute.
8. **Monitor contract (tracked).** `monitors_armed[]` in state, each `{id,purpose,armed_at_turn}`. A Monitor
   watches a concrete event (file appears / endpoint status / process exit) and signals wake by touching
   `.autonom/RESUME`; it is ABORT-sensitive. At DONE, cancel each by ID — else ghost fires resurrect a dead loop.
9. **delaySeconds decision table:**

   | Situation | delaySeconds |
   |---|---|
   | Made progress, no external wait | 60 |
   | Retrying after error (attempt n) | `min(60×2^n, 600)` |
   | Waiting on external event | don't wakeup — arm a Monitor |
   | No progress this turn | 120 (cool-down) |

10. **Wakeup failure backstop.** Wrap the wakeup: on failure retry once (+30s); on 2nd failure set
    `wakeup_failed=true` and surface it next user interaction — never pretend a dead loop is alive.
11. **Stale-state recheck.** Turn-top: if `now - last_updated > 6h`, re-validate spec-named files still exist /
    weren't renamed; on drift → halt + report.
12. **Working-tree dirty check.** Turn-top `git status --porcelain`: if there are uncommitted changes NOT on the
    latest `autonom/pre-*` tag (user/another agent edited), HALT — never `reset --hard` over their work.
13. **DONE rigor + evidence template.** Re-verify ALL criteria (§7). Emit:
    ```
    ## DONE — verified
    - [✓] <criterion> — proof: <command + output excerpt>
    - Turns: N/max | Tokens: x/max | $: y/max | Deferred: <top-3>
    ```
14. **Verification-authority allowlist.** Authoritative = tests pass (rc=0 AND assertions>0) · linter clean ·
    typecheck clean · endpoint status+body match · file/AST match. NEVER: "looks right" · "advisor said fine" ·
    "no errors printed" · bare rc=0 on an empty suite.
15. **Scope-creep guard (autonomous — no dropdown).** Out-of-spec advisory advice → DEFER to `deferred_ideas`
    (report at DONE), keep driving the scoped task. Spec-drift diff at turn top → self-correct back to spec silently.
16. **Periodic plan-validity re-check.** Every 10 turns, re-derive the plan from spec+state, diff vs the original
    numbered steps; if >30% of original steps are no longer relevant → halt + re-plan.
17. **Prediction log.** Each action states a one-line prediction of the observable change it should cause ("after
    this edit, `pytest -k X` passes"). A prediction/observation MISMATCH triggers re-plan (evidence-based), not
    merely a strike — sharpens the stuck-vs-productive call beyond the error-hash alone.
18. **Differential verification.** `done?` checks INVARIANTS, not just final state: tests pass AND no new warnings
    AND only intended files changed AND no new dependencies added. Catches silent regressions a single check waves through.
19. **Monotonic convergence metric.** Define a progress metric that must decrease (e.g. failing-test count); if it
    rises or stays flat for N steps → abort as non-converging, even if `done?` never trips. A termination proof
    against infinite loops, independent of the strike logic.

> **ZERO mid-run interrupts.** Every halt/abort/escalate in this skill writes `.autonom/ABORT`, STOPS, and reports
> at the END — NEVER a mid-run dropdown. That is the literal contract. Runs UNINTERRUPTED to DONE — the user is NEVER pulled back in mid-run: no "still
> on track?" dropdowns AND no safety-gate dropdowns. Dangerous action-classes are PRE-AUTHORIZED (or denied) ONCE
> at the SCOPE step; anything dangerous NOT pre-authorized is DEFERRED + reported at DONE, never asked. Missing
> info is resolved CLARIFY-FIRST (the agent self-answers from task context), only a genuinely user-only fact is
> deferred. Per-turn user-visible delta header:
> `T<turn>/<max> | step k/n ✓verified | budget: a% turns, b% $ | c deferred`.

## Dangerous actions — decided at SCOPE, never an interrupt mid-run
Dangerous action-classes — deletions (move to `.retired/` + log), git mutations / pushes, sending comms,
publishing public content, paid-API calls, purchases, anything destructive or outward-facing — are decided ONCE,
at the SCOPE step: the manifest's `permissions` block lists which classes are PRE-AUTHORIZED for this run. During
the run there is ZERO user interruption — a pre-authorized class just runs (subject to the automatic safety floors
below, which BLOCK or DEFER, never prompt); a dangerous action NOT pre-authorized at scope is DEFERRED (logged to
`deferred_ideas`, reported at DONE) and the run continues. The ONE approval moment is the scope gate.

**`necessary?` predicate — the key that makes zero-interrupt SAFE.** DEFER is only safe for an action the task
does NOT need to reach DONE. Before deferring a non-pre-authorized dangerous action, evaluate `necessary?(action)`
= is it REQUIRED for any DONE criterion?
- `!allowed && necessary` → **FAIL LOUDLY** (abort + report `blocked-required: <action>`); NEVER defer — deferring
  a REQUIRED action silently turns IMPOSSIBLE into apparently-DONE.
- `!allowed && !necessary` → DEFER **with a one-line proof** (`DONE criterion [X] checks [Y]; this action affects
  [Z], not in any criterion`), log + continue. At DONE-time, if DONE turns out unreachable, surface ALL such proofs
  for audit. Since a stuck-loop ABORT usually fires FIRST, the loop-ABORT report (§Reference item 1) MUST also scan
  `deferred_ideas` for a `!necessary` deferral as a candidate root cause — the audit is a POST-halt diagnostic, not a
  pre-halt gate (a mis-judged "not necessary" is itself a silent-failure path).
- `allowed` → execute.

**Capability-scoped permissions (not a bare class flag).** Each pre-authorization carries SCOPE, so "approve push"
can't hit the wrong target: `permissions: {push:{remote:origin, branch:feature/*, max_pushes:1}, install:{packages:[pytest,requests]}, publish:{environment:staging}}`.
`allowed?(action)` matches the action against the SCOPED capability; an out-of-scope push, an unlisted package, or
a prod publish is treated as NOT pre-authorized → routed through `necessary?` above.
**Mechanical target validator:** before any scoped dangerous action, COMPUTE the actual target (real remote/branch/
package/env) and ASSERT it matches the scope — never rely on LLM parsing alone; a mismatch → treated as NOT pre-authorized.

**Added gates (adversarial-review hardening):**
- **Silent-destruction floor.** AUTO-BLOCK (or DEFER if the run wasn't pre-authorized for destruction) any action that truncates a tracked file (>50% size
  reduction), zeroes a file, or changes perms/ownership (`chmod`/`chown`/`icacls`). A plain delete-gate does NOT
  cover these — an unattended run could brick a config without surfacing.
- **Strict pattern-match allowlist (honest: it IS an LLM match, just a crisp one).** `allowed?` matches the command
  against an explicit allowlist (`git`, `npm`, `node`, `python <named-script>`, `pytest`, …). MECHANICAL backstops
  that need zero judgment: any command with `python -c` / `bash -c` / `eval`, a dynamically-written or temp file, OR
  **2+ chained shell operators** (`|` `&&` `;` `>` `$(`) auto-blocks → break it into separate allowed calls.
- **Dependency-install gate.** Treat `npm/pip/cargo/... install <pkg>` with the same severity as a git push —
  DEFER unless pre-authorized at scope (supply-chain + env-break risk); best run inside the shadow/ephemeral env below.
- **Flaky-test escape hatch.** Track `resets_per_criterion` in state; if a `git reset --hard` fires for the SAME
  acceptance criterion 3× → HALT, do NOT reset again, escalate "potential flaky env / non-deterministic test."

## Routing (free-first)
- Heavy code / multi-file authoring → a code-capable free lane draft → main-model skim; long prose → free lane.
- Research / advice / angles → a web-LLM advisor (+ a cross-LLM debate when worth it).
- Hard reasoning → a strong reasoning lane; bulk/classify → a local LLM. Main model = orchestration + final verify only.
- Before each non-trivial sub-step: pick the right skill/script/lane for it (capability lookup).

## Each turn — 3 phases (GUARD → ACT → CLOSE)
Keep the turn to THREE phases; the detailed sub-rules live in §Reference and are read ONLY when a GUARD flag trips
(don't run 25 checks every turn — adherence degrades faster than coverage improves).
- **GUARD (≤5 fast checks):** (1) `.autonom/ABORT`? → stop. (2) read state (atomic) + resume an incomplete `intent`.
  (3) env/spec drift — stale-state >6h | dirty-tree | spec-drift. (4) budget / loop / infeasible (loop + convergence +
  hypothesis detail in §Reference, consulted ONLY if a strike / no-progress flag trips). (5) for the chosen action:
  `allowed?` + `necessary?` + mutation-surface. **Any STOP condition → write `.autonom/ABORT`, halt, report at DONE —
  never a dropdown.**
- **ACT (one minimal step):** pick the MINIMAL next action (Plan-minimizer) → state a one-line PREDICTION of the
  observable change it should cause (§Reference prediction-log — run on EVERY action, NOT flag-gated) →
  `git tag autonom/pre-<intent>` → write `intent` → execute. Consult the advisor (scope-prefixed, quarantined) ONLY when stuck.
- **CLOSE (proactive controls run EVERY turn, never flag-gated):** verify via the authority allowlist + **differential
  verification** (tests pass AND no new warnings AND only intended files changed AND no new deps) + confirm the
  PREDICTION held; on fail → classify + `git reset --hard` the tag + clear `intent`. Update the **monotonic-convergence
  metric** (must trend down; flat/up for N turns → write `.autonom/ABORT`). Update state atomically — when recording a
  FAILURE, update `attempts` per §Reference item-1 logic (error-hash-keyed, reset on a new diagnostic/hypothesis) so GUARD
  reads an already-correct count (`spent`/`seen_errors`/`assumptions` too). **Write `.autonom/status.json`** (state ·
  last_turn_at · eta · next_action) so the run is inspectable without interrupting. **Every 10 turns:** plan-validity
  re-check + a **GOAL_COHERENCE check** — summarize the current deliverable vs the ORIGINAL high-level goal; on
  drift or criteria-gaming → write `.autonom/ABORT` (catches passing a wrong/incomplete DONE). Emit the 1-line delta header. If DONE →
  run the FALSE_DONE checklist + re-verify ALL criteria + evidence block → cancel Monitors → report. Else → schedule
  the wakeup (fuse, delaySeconds per §Reference table).

## Design lineage
Combines three primitives: a front-loaded **scoping** pass (ask everything first), a self-paced
**loop** (schedule your own next wakeup), and a **web-LLM advisor** consulted each iteration. The
robustness layer (stuck-vs-productive detection, hypothesis forcing-function, tag-rollback, tracked
Monitors) was hardened against real autonomous-run failure modes — most importantly the lesson that
*persistence past a naive "3 strikes" cap can be correct when each attempt surfaces genuinely new
information*, which is why the strike counter keys on `(intent + error)` and resets on new diagnostics.
