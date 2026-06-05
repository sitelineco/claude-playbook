# Siteline — New-Project Agent Build Kickoff (in-depth)

Paste the master prompt below into a **fresh Claude Code session opened in the target repo**
(MoneyWell, ProPunch, etc.). One project per session, clean context. It drives the same depth we
built for Salt Home Studio: understand the real codebase → fix/author accurate rules → build and
**verify** specialist agents by running them on real code → stand up an automated functional safety
net → fix the real bugs that surface → commit.

> Keep this file OUTSIDE the project repos (it's cross-project). Each repo ends up with only its own
> `.claude/`, `CLAUDE.md`, `e2e/`, and CI — nothing about the other projects.

---

## How to run it
1. Open a Claude Code session with the target repo as the working directory.
2. Paste the master prompt. Change the project name on the first line.
3. It checks in between phases — approve the agent roster before anything is written.
4. Let it **run** each agent and the E2E suite, not just write files. Make it show you green.

---

## What "in-depth" means (the lessons from Salt — already baked into the prompt)
- **Validate borrowed rules first.** Salt shipped `.claude/rules/` written for a *different product*;
  pointing agents at them teaches the wrong conventions. Confirm rules describe THIS product.
- **Verify by running on real code.** Every agent gets pointed at real files/diffs; if it's weak or
  noisy, sharpen the prompt and re-run. Salt's reviewers caught a real shipped bug this way.
- **Probe the live site.** The publishable keys are in the deployed bundle; you can recover them,
  query the live API/Storage, and confirm repo-vs-deployed drift and that data-backed features work.
- **Fix what you find.** Agents that surface bugs (empty hero, broken "load more") → fix them in the
  same pass; don't just report.
- **Make it routine.** Hand-checking is how regressions slipped in. Stand up Playwright + a daily CI
  cron so breakage fails loudly.
- **Reviewers are read-only and must not over-report.** Write/Bash/hook agents stay committed per-repo.

---

## MASTER KICKOFF PROMPT

```
You are setting up customized Claude Code agents and an automated front-end safety net for THIS
project ([PROJECT NAME]). Build to real depth: understand the codebase, encode how we actually work,
and prove each agent works on real code. Go phase by phase and CHECK IN with me between phases.

PHASE 0 — VALIDATE EXISTING RULES
- Read every file in .claude/rules/ and any CLAUDE.md. For each, tell me plainly whether it actually
  describes [PROJECT NAME] or is a borrowed/generic template (wrong product/persona, references to
  tools/dirs that don't exist here). Flag mismatches and STOP for my decision before any agent is
  pointed at them. Do NOT build on unverified rules.

PHASE 1 — UNDERSTAND THE CODEBASE
- If CLAUDE.md is missing/thin, run /init, then refine.
- Map, citing file:line: what the app does and who uses it; the stack; routing/entry points; the
  data layer (how data is read/written, where types come from, generated files, auth, multi-tenancy);
  component/hook/naming conventions; testing setup; key integrations; the top non-obvious gotchas.
  Mark anything you can't verify as "unverified" — never guess.
- Write/refresh a LEAN CLAUDE.md (<200 lines): build/test commands, env rules, data-layer rules,
  "always do X" conventions, real gotchas. Prune anything inferable from the code.
- If rules were missing or wrong (Phase 0), write accurate .claude/rules/ from what you found
  (design/brand system, naming conventions, commit style — grounded in THIS codebase).

PHASE 2 — PROPOSE THE AGENT ROSTER (wait for my approval)
- Recommend single-responsibility specialists tailored to THIS stack and the work we repeat. Strong
  default set, plus stack-specific ones:
    • code-reviewer (read-only) — correctness, our data-layer rules, security, conventions
    • design-reviewer (read-only) — UI vs. our design system + the "looks AI-made" test (if it has a UI)
    • {data-stack}-expert — DB/schema/RLS/migrations/edge functions/data hooks (e.g. supabase-expert)
    • frontend-auditor (read-only) — front-end FUNCTIONALITY (dead buttons, forms that don't submit,
      empty render blocks, dead/duplicate components, repo-vs-deployed drift); maintains the E2E suite
    • security/RLS auditor — auth bypass, IDOR, org-isolation, secret exposure, injection (high value
      for any multi-tenant or token-auth app)
    • a build-advisor that remembers our conventions/decisions across sessions (like Salt's salt-architect)
    • plus any stack specialists the codebase warrants (payments, SEO, performance, test-writer)
- For each: name, a PROACTIVE description ("Use proactively when…" + triggering events), tools
  (allowlist — reviewers/auditors get Read/Grep/Glob[/Bash] only), model (haiku=mechanical,
  sonnet/opus=hard reasoning), and one-line rationale. Map each to our validated rules.

PHASE 3 — BUILD THE AGENTS (after I approve)
- Write each to .claude/agents/ as Markdown + YAML frontmatter. Body = role + a concrete procedure
  (what to read first, what to check, output format).
- Descriptions MUST contain a real proactive trigger. Reviewers/auditors are read-only and told to
  "flag only gaps that affect correctness/security/our standards — do not over-report."
- Bind conventions by pointing the prompt at .claude/rules/* and CLAUDE.md, not by hardcoding facts
  that go stale. Keep any agent needing a hook/Bash/write committed in THIS repo (not a shared plugin).

PHASE 4 — VERIFY EACH AGENT ON REAL CODE (mandatory — this is the depth)
- Run each agent's procedure against REAL files/diffs in this repo. Confirm concrete, grounded,
  project-specific output — not generic advice — and that it does NOT over-report. If weak/noisy,
  sharpen the prompt and re-run.
- Use live probing where useful: recover the app's publishable keys from the deployed bundle, hit the
  live API/Storage read-only to confirm data-backed features work, and check repo-vs-deployed drift.
- FIX the real bugs the agents surface in this same pass (don't just report them); build + lint after.

PHASE 5 — STAND UP THE FUNCTIONAL SAFETY NET
- Add Playwright (@playwright/test) + a config that builds/serves the app for PR runs and accepts a
  BASE_URL for live runs. Tag data-writing tests @destructive and exclude them from production runs.
- Write E2E for the critical flows (key routes render, primary CTAs/links work, forms submit and give
  feedback, any "load more"/pagination actually loads, no uncaught JS errors).
- Add a GitHub Actions workflow: full suite on every PR (built preview) + a daily cron of the
  read-only suite against production. Note the required repo secrets.
- RUN the suite and show me it's green (against a local preview built with real config). Fix failures.

PHASE 6 — COMMIT
- Commit CLAUDE.md, .claude/rules/ fixes, .claude/agents/, e2e/, and the workflow with Conventional
  Commit messages. Push to the branch I'm working on. Tell me any repo secrets I must add.

Principles: a few sharp specialists over one broad agent; verify by RUNNING on real code; fix what you
find; make the checks routine. Don't report anything "done" until you've watched it work.
```

---

## After 2–3 projects each have a working roster
The genuinely shared "how Siteline builds" reviewers (code-reviewer, design-reviewer,
security/RLS-auditor, frontend-auditor) become candidates to promote into a private **plugin
marketplace** so they update in one place across all repos — but keep project-specific agents and
anything needing a hook committed per-repo (plugins strip hooks). Do this only once the per-repo
versions have proven themselves.
