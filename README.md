# Siteline Claude Playbook

Shared methodology for building customized Claude Code agents (and an automated functional safety
net) in any Siteline project — Salt Home Studio, MoneyWell, ProPunch, and whatever comes next.

- **`AGENT-BUILDING-GUIDE.md`** — the full methodology (three-layer model, build-and-verify,
  grounding & evals, sharing, pitfalls). Read once.
- **`KICKOFF.md`** — the master prompt to run in a fresh session in any repo.

> This lives in ONE repo so it's maintained in one place. Each *project* repo keeps only its own
> `.claude/`, `CLAUDE.md`, `e2e/`, and CI — never a copy of this.

---

## The optimal, user-friendly setup (do this once)

Goal: open a session in **any** project on Claude Code (web) and have this playbook already there,
with zero per-project work.

### Step 1 — Put this folder in a git repo
Create `sitelineco/claude-playbook` and push these three files. (Private is fine — see the note
below.)

### Step 2 — Add ONE line to your environment's setup script
In Claude Code on the web → your environment's settings → **Setup script**, add:

```bash
git clone --depth 1 https://github.com/sitelineco/claude-playbook.git ~/claude-playbook 2>/dev/null \
  || git -C ~/claude-playbook pull --ff-only
```

The setup script runs at the start of **every session in that environment, for every project**, so
the playbook is now available at `~/claude-playbook/` everywhere. Update the playbook once → every
future session gets it.

### Step 3 — In any project, paste this one-liner to start building
```
Read ~/claude-playbook/AGENT-BUILDING-GUIDE.md and ~/claude-playbook/KICKOFF.md, then run the
KICKOFF for THIS project ([PROJECT NAME]) — phase by phase, checking in with me between phases.
```

That's it. No per-repo files, no copy-paste of the methodology, one source of truth.

---

## Private repo? (recommended — it names internal projects)

A `--depth 1` clone of a **private** repo needs a token, and the setup script may run before tokens
are available. Two clean options:

- **Use a SessionStart hook instead of the setup script.** It runs once the session (and its GitHub
  token, `$GH_TOKEN`) exists, so a private clone works. (Ask Claude to scaffold this — it has a
  `session-start-hook` skill.)
- **Or** keep the repo public (the content is methodology + public Anthropic docs; the only
  "internal" bits are project names) and use the simple setup script above.

---

## Next level: ship the shared agents as a plugin

Once 2–3 projects have proven rosters, package the genuinely reusable reviewers (code-reviewer,
design-reviewer, security-auditor, frontend-auditor) as a Claude Code **plugin** in this same repo
(add `.claude-plugin/marketplace.json` + an `agents/` dir) and register it in each repo's
`.claude/settings.json`. Then the agents themselves — not just the doc — are shared and
version-updated centrally. See `AGENT-BUILDING-GUIDE.md` Part 8.
