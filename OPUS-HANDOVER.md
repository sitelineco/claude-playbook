# Model step-down operating manual (the "Fable → Opus handover")

> **Audience: any Claude session running on a smaller/cheaper model than the one that built the
> project's harness** — Opus after Fable, Sonnet after Opus, whatever comes next. Written by Claude
> Fable 5 (2026-07-06; refactored 2026-07-07 after using it on a live multi-repo security pass).
>
> **This file is 100% GENERIC — it names no project.** It ports unchanged to MoneyWell,
> PerformaTrack, MyWager, Vacay Verify, Salt, and anything future. The CONCRETE instances of each
> failure mode below — the actual bugs, flags, and gotchas that bit a specific codebase — live in
> that repo's own `.claude/rules/step-down-ledger.md` (the per-project ledger). Two layers on
> purpose: the discipline is universal; the scar tissue is local. Do not paste project specifics
> back into this file.
>
> **The premise:** a capability gap between models is largely compensable by PROCESS. What a
> stronger model does in one pass — hold distant context, distrust plausible-sounding claims,
> notice the second-order effect — a smaller model recovers by externalizing those checks:
> cite-then-verify, re-derive, small steps, adversarial review, and — the highest-leverage move —
> turning a rule into a machine check so no model has to remember it.
>
> **What does NOT transfer, so don't pretend it does:** one-shot depth on genuinely hard novel
> problems, and long-horizon coherence. The countermeasure is not a prompt — it is decomposition
> plus verification loops (§5). Budget more turns and more tokens for hard work, not more confidence.
>
> **What transfers completely:** the project rules files, the specialist agents, the runbooks, and
> the discipline below. Those are the asset. Keep them colocated and current — a stale rules file is
> worse than none, because every model trusts it.

---

## 1. The install block (paste into each project's CLAUDE.md — this stub is meant to be standalone)

This 8-line stub is the ONE thing that is intentionally copied per repo: it must work even in a
session that has no playbook clone. Everything longer lives once (here) and is pointed to, not
duplicated.

```markdown
## Model step-down discipline (binding for every session)
Full doc: ~/claude-playbook/OPUS-HANDOVER.md · project scar tissue:
.claude/rules/step-down-ledger.md. The short form:
1. Any claim about existing code cites `file:line` — if you can't cite it, read it first.
2. Re-derive every number before quoting it (percentages, rates, deltas) — never trust
   prose arithmetic, including your own from earlier in the session.
3. Never assert live state (flags, cron, DB rows, deploys) from docs or memory — query it.
4. Say what's verified vs inferred, out loud, every time. An unlabeled guess is a defect.
5. Several small verified changes beat one large one. Verify by running the flow, not the build.
6. Non-trivial diff → run the reviewer agents, then adjudicate each finding with evidence —
   rubber-stamping acceptance OR dismissal is the failure, not just missing a bug.
7. Every fix has an "other end" — name what else it could break and go check that file too.
8. Before ending a turn, reread your last paragraph: if it's a plan or promise, execute it now.
```

---

## 2. The step-down failure MODES (generic shapes — concrete instances live in the per-project ledger)

Each is a way a smaller model degrades, stated without any project's specifics. When one bites in a
real repo, the fix PR records the concrete instance in that repo's `step-down-ledger.md` — that
ledger is what makes the shape recognizable next time. Read your project's ledger alongside this.

| # | Failure mode | Generic shape | Countermeasure |
|---|---|---|---|
| 1 | **Plausible-but-wrong acceptance** | Asserting how distant code behaves because the claim *sounds* consistent with a name/pattern. Names lie — a function called `validate*` may validate nothing; a field called `*_score` may be a hardcoded constant. | Rule 1: cite or read. Treat every name as a hypothesis until the body confirms it. |
| 2 | **Cross-file blindness** | Fixing one end of a two-ended contract and shipping the other end broken (a write without its read; a producer without its consumer; a gate added to a callee while a caller still sends the old shape). | Rule 7: every change names its other end and you go open that file. Runbooks that enumerate the ends turn this into a checklist. |
| 3 | **Stale-state assertion** | Quoting a flag / cron / deploy / row count from a doc or from earlier in the chat. Config lies: two layers that can drift independently (a feature flag and the job that reads it; a grant in a migration vs the live grant) read "on" in one and "off" in the other. | Rule 3: query the live system, and check BOTH layers when two can disagree. |
| 4 | **Silent early stopping** | Declaring done after the happy path; leaving "next I'll verify X" as prose instead of doing it. | Rule 8 + rule 5's "verify by running." Build-passing ≠ working; observe the actual behavior. |
| 5 | **Metric / denominator invention** | Answering a data question with a self-invented metric instead of the governed definition; wrong time-window column; `COUNT(*)` where `COUNT(DISTINCT …)` was meant. | Every data claim resolves to a governed definition BEFORE querying. No definition → say so and propose one; don't improvise silently (§3). |
| 6 | **Reviewer rubber-stamping** | Accepting all agent findings (or dismissing all) without adjudication; padding a review with theoretical nits. On a step-down model the *reviewer* is also weaker — see §5. | Rule 6: adjudicate each finding against the code. "Nothing material found" is a valid, good outcome. |
| 7 | **Checklist-satisfying vs goal-satisfying** | Doing the steps but missing the point (the step is marked done while the outcome the step existed to produce doesn't hold). | After any checklist, one pass of "does the outcome actually hold?" against the stated goal, ideally by observing the running system. |
| 8 | **Confident compliance/security "fine"** | "This is fine" on PII / consent / auth / payment code without tracing the actual flow. A confidently-wrong "fine" on a real leak is the worst possible output. | Trace the flow end-to-end; when unsure, say unsure and show the basis; route to the security agent. Default to "unsafe until proven safe." |

---

## 3. Numbers discipline (the cheapest big win)

A stronger model catches bad arithmetic in passing; a smaller one must make it mechanical:
- **Percentages/deltas:** recompute from both endpoints yourself before repeating any claimed
  change. State the denominator with the rate, every time.
- **Windows and time columns:** the wrong time column silently windows on nothing. Name the column
  and the window in the query you show; the per-project data-model rules say which column per table.
- **COUNT discipline:** distinct-what? `COUNT(*)` where a distinct count was meant is a one-line
  check that prevents whole classes of wrong reads.
- **Anchors decay:** any economic number in a doc is a STARTING POINT — re-derive from live data
  before a recommendation rides on it.

---

## 4. Turn a rule into a check (the move the prose version of this doc kept getting wrong)

A written rule that everyone trusts and nothing enforces is the failure mode this whole file exists
to fight — so the first question for any recurring rule is **"can this be a hook, a lint, a CI gate,
or a test instead of a sentence?"** A machine check needs no model to remember it and fires on every
model forever.

**Worked example (a real recurrence):** a repo's conventions file documented "pin the Supabase
client import (`@supabase/supabase-js@2.39.0`) — an unpinned `@2` drifts and breaks `deno check`,"
citing a bug it had already caused once. The rule was written. The bug then recurred in *four more*
edge functions, because a sentence in a doc catches nothing at author time. The fix is not a
better-worded rule — it's one line of enforcement:

```bash
# prebuild / CI gate: fail if any edge fn imports the client unpinned
! grep -rEn 'supabase-js@2(["/]|$)' supabase/functions && echo "OK: all pinned"
```

Apply the test to every rule you're tempted to write or that keeps getting violated:
- **Mechanizable now** (import pin, "no PII in logs," "RLS on every new table," "no raw `fetch` in
  components") → write the check, not the sentence. Prefer the check even when a sentence exists.
- **Not yet mechanizable** (judgment calls — "is this the right product for this user") → the prose
  rule + the per-project ledger is the best you can do; keep it, but say so honestly.

When you add enforcement, record it (a repo's docs should say which rules are machine-enforced vs
prose-only, so nobody re-audits a thing CI already guards — and nobody trusts a prose rule as if it
were enforced).

---

## 5. Buying capability back with orchestration — and what it costs

When the task is genuinely hard (novel design, subtle bug, high-stakes audit), don't attempt the
one-shot a bigger model would do. Spend tokens on structure instead:
- **Find → adversarially verify:** never ship findings from a single pass. Have a second,
  independent pass try to REFUTE each finding. Kill what doesn't survive.
- **Decompose to checkable pieces:** split the problem so each piece has an independent pass/fail
  test. If a piece can't be checked independently, that piece IS the risk — spend the effort there.
- **Judge panels for wide solution spaces:** generate 2–3 genuinely different approaches, score them
  against explicit criteria, then synthesize — beats iterating on the first idea.
- **Escalate honestly:** if after decomposition a piece still exceeds capability, say so and
  recommend running that piece on a stronger model as a paid one-off. One expensive call that's
  right beats ten cheap ones that converge on plausible-wrong.

**Orchestration is not free capability — know when it earns its price.** A cheaper model wrapped in
a heavy multi-agent harness can cost MORE per correct outcome than the expensive model answering
once. Use the fan-out when the work is **high-stakes, irreversible, or broad** (a security audit, a
migration, a release gate) — there, more independent passes are worth the tokens. For a routine edit
or a lookup, one grounded pass is cheaper and just as right; a fleet of agents is waste.

**Adjudicate the adjudicator.** §5's verify loop assumes a competent verifier — but on a step-down
model the verifier is *also* the weaker model. Sub-agents surface real findings AND mislabel severity,
miss the "other end," and occasionally confidently invent. So the orchestrator still owns the verdict:
re-check every critical finding against the source yourself before acting on it. Orchestration
amplifies the orchestrator's judgment; it does not replace it.

---

## 6. Measurement (does this discipline earn its keep?)

This file must meet the bar it sets for everything else: a claim of value needs a signal, not a
vibe. Cheap signals that a step-down session is actually holding the line:
- **Reviewer-catch rate:** how many of a session's own diffs get a real correctness finding from the
  reviewer pass before commit (rule 6 working) vs after merge (rule 6 skipped). Trending the latter
  down is the goal.
- **Ledger growth:** each new row in a project's `step-down-ledger.md` is a failure that escaped once.
  Rows should get rarer over time for a given failure shape; a shape that keeps recurring is a signal
  to mechanize it (§4), not to reword it.
- **Rework rate:** diffs that had to be re-opened because the "other end" (rule 7) was missed.

None of these needs tooling — they're countable from PRs and the ledger. If a rule here never shows
up in a catch and never prevents a ledger row across many sessions, it's costing attention without
preventing anything: delete it (§7).

---

## 7. Maintenance (how this stays alive without rotting)

- **This file (generic):** changes only when a failure *shape* or a *countermeasure* is genuinely
  new — rare. It carries no project specifics, so cross-project drift can't accumulate here.
- **Each project's `.claude/rules/step-down-ledger.md`:** the living layer. When a step-down failure
  bites — a wrong read, a shipped two-ended bug, a stale-state assertion — the fix PR (or the
  post-mortem) adds the concrete instance there, tagged with the §2 shape number. That ledger is
  what makes the generic shape recognizable in that codebase.
- **Prune both.** A rule or a ledger row that stops earning its place (the failure became impossible,
  or got mechanized per §4) gets deleted — discipline that costs attention without preventing
  anything is noise, and noise is the failure mode this whole file exists to fight.
