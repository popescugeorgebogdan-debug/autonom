# 👁️➜🤖 /autonom — give your AI agent *autonomy*, not just reach

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that does one thing well:
**scope a task exhaustively up front, then work on it autonomously, non-stop, until it hits a
machine-checkable definition of DONE** — pausing only for a small set of non-negotiable safety gates.

Most "agentic" tooling races to give an agent more **reach** (read the whole web, call more APIs).
`/autonom` is the other axis: **autonomy** — the discipline to *finish a job by itself* without
babysitting, and without the two ways unattended agents usually fail (drifting off-scope, or
spinning forever on the same error while reporting "progress").

---

## The idea in one table

|  | "Reach" tools | **/autonom** ("autonomy") |
|---|---|---|
| What they add | the agent can *see/fetch* more (X, Reddit, GitHub, YouTube…) | the agent can *finish a scoped task* unattended |
| Shape | external data-access CLIs/MCPs | an orchestration skill (scope → loop → verify) |
| Failure they fix | "my agent is blind / rate-limited" | "my agent wanders or loops, and I have to watch it" |
| One-liner | *eyes to see the internet* | *hands to finish the job* |

They're complementary. Reach without autonomy still needs a human driving every step; autonomy
without reach finishes the wrong things confidently. This repo is the autonomy half.

---

## What makes it different from "just loop the agent"

Claude Code ships a built-in `/loop`. Its official description, in full: *"Run a prompt or slash
command on a recurring interval (e.g. /loop 5m /foo). Omit the interval to let the model self-pace."*
That is a timer. As a way to actually **finish** a multi-step task it is, in my blunt opinion,
garbage, and that is fine: a timer was never meant to be a worker. It has no definition of done, no
memory of why the last attempt failed, and no way to undo a change it just broke. `/autonom` is the
worker. Concretely, in a ~150-line skill file:

1. **Scope-first, refuse-if-unscoped.** It asks every outcome-determining question (one dropdown at
   a time) *before* touching anything, and **refuses to start unless the definition of DONE is
   machine-checkable** — `test rc=0 + assertions>0`, an endpoint returning a specific status+body, a
   file matching a regex/AST. No "looks done."
2. **Productive-iteration vs stuck.** The hard part of unattended work is knowing when to quit. The
   naive "stop after 3 tries" is wrong — persistence is *correct* when each attempt surfaces new
   information. So the strike counter keys on `sha256(intent + normalized_error)` and **resets the
   moment a new error, fact, or hypothesis appears.** It only escalates when the *same* failure
   repeats. A `HYPOTHESIS:` forcing-function catches the "flaky error looks new but nothing changed" case.
3. **Tag-rollback around every mutation.** `git tag autonom/pre-<intent>` before a mutating step;
   `git reset --hard` that tag on verify-fail. Never resets over a human's uncommitted work.
4. **Self-paced, crash-safe.** Atomic `state.json` is the single source of truth; the wakeup prompt
   is a *fuse* (`resume from state`), never the spec. Idempotency keys + a cross-run error journal
   mean a double-fired wakeup or a crash resumes cleanly instead of double-executing.
5. **Safety gates that never auto-pass.** Deletions, `git push`, sending comms, publishing public
   content, paid-API calls, purchases — these always stop and ask, even mid-autonomous-run.

See [`SKILL.md`](./SKILL.md) for the full contract, state schema, the 16-point robustness layer, and
the per-turn checklist.

---

## Install

Drop it into your Claude Code skills directory:

```bash
git clone https://github.com/popescugeorgebogdan-debug/autonom.git
mkdir -p ~/.claude/skills/autonom
cp autonom/SKILL.md ~/.claude/skills/autonom/SKILL.md
```

Then in any Claude Code session: `/autonom build X until the test suite is green`.

> **A note on the rule references inside `SKILL.md`.** It cites the author's personal Claude Code
> ruleset (`BOGDAN.*`, `HARD RULE n`). Those are *named conventions*, not dependencies — e.g.
> "ask via a dropdown, never free-text", "free-lane-first dispatch", "clickable file links". Map
> them to your own house rules; the skill works standalone.

## Why I built it

I run a lot of long-horizon, single-machine automation. The recurring pain was never *can the agent
do the step* — it was *can I leave the room*. `/autonom` is the answer I converged on after watching
unattended runs fail the same two ways (off-scope drift, and confident infinite loops), so the whole
design is really a list of countermeasures to those two failure modes.

## License

MIT — see [`LICENSE`](./LICENSE).
