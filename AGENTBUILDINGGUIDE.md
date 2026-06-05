# Siteline Agent-Building Guidebook

> **Purpose:** A repeatable, sourced playbook for building smart, proactive Claude Code
> agents in each of our projects (salt-home-studio-web, MoneyWell, ProPunch, …).
> Open a Claude Code session in a project, follow the workflow, and you get a set of
> specialist agents that understand *that* codebase and encode *how Siteline builds*.
>
> **How to use:** Read Part 1 once (concepts). Then for each project, run Part 4
> (the per-project workflow) — or just paste the kickoff prompt in Part 5 and let
> Claude do the heavy lifting. Part 6 is the copy-paste template library.
>
> **Every claim here is sourced to Anthropic's official docs** (`code.claude.com/docs`)
> or the Anthropic engineering blog, except where explicitly marked as practitioner
> opinion. Source links are in Part 12.
>
> **Reviewed:** technical claims fact-checked against the live docs (98% verified, no critical
> errors) and the strategy stress-tested by a staff-level review. Their findings are folded in:
> validate borrowed rules before use (Phase 1 step 0), deterministic hook-based proactivity
> (Part 3.6), and a maintenance/cost/rollback discipline (Part 8.5).
>
> **Part 9** adds a grounding-and-accuracy layer for **data-facing agents** (reporting, DB,
> Supabase), adapted from Anthropic's analytics-agents playbook — governed definitions, a
> retrieval router, a colocation/CI staleness defense, and an offline-eval + ablation discipline
> that closes the "does the agent actually help?" measurement gap.

---

## Part 1 — The mental model: three layers, not one

The biggest mistake is treating "agents" as the only tool. Claude Code has **three
layers of knowledge**, and a great setup uses all three deliberately. Anthropic's own
decision model ([features-overview]):

| Layer | What it is | When it loads | Use it for |
|-------|-----------|---------------|------------|
| **CLAUDE.md / `.claude/rules/`** | Always-on context | Full content, **every request** | Things Claude must *always* know: build commands, conventions, "never do X" rules |
| **Skills** (`SKILL.md`) | On-demand knowledge / workflows | Description always; body only when relevant or `/invoked` | Reference material needed *sometimes*; repeatable playbooks triggered by `/name` |
| **Subagents** (`.claude/agents/`) | Isolated worker with its own context window | Only when spawned; returns a summary | A task that would flood the main context, or work needing tool limits / a cheaper model |

> **How `.claude/rules/` actually loads:** a rules file loads **every session by default**
> (same cost as CLAUDE.md). It only becomes load-on-demand if it declares a `paths:` glob in
> its frontmatter, in which case it loads *only when a matching file is open*. So a rules file
> with **no `paths:` is always-on context** — that's why an oversized one (see Part 7) is a real
> context cost and a candidate to move into a Skill. ([memory])

**Anthropic's "build it over time" triggers** (verbatim intent from [features-overview]):

- Claude gets a convention wrong **twice** → add a line to **CLAUDE.md**
- You paste the same multi-step playbook a **third** time → make a **Skill**
- A side task **floods your conversation** with output you won't reuse → make a **Subagent**
- You want something to happen **every time without asking** → a **Hook** (not a prompt
  instruction — CLAUDE.md is "context, not enforced configuration")

> **Key constraint that shapes everything:** context is finite and degrades as it fills
> ("context rot"). Subagents are powerful *because* each runs in a fresh, isolated context
> window and returns only a ~1–2k token summary, even after burning tens of thousands of
> tokens internally. Keep always-on context lean so the agents stay sharp. ([context-engineering])

---

## Part 2 — What a subagent actually is (verified format)

A subagent is a Markdown file with YAML frontmatter. **The body becomes the system prompt.**
([sub-agents])

```markdown
---
name: design-reviewer            # REQUIRED — lowercase + hyphens; this is the routing identity
description: >                   # REQUIRED — how Claude decides to delegate. Make it a trigger.
  Use PROACTIVELY after any UI/component change to review against our design guidelines.
tools: Read, Grep, Glob, Bash    # OPTIONAL — allowlist. OMIT to inherit all tools.
model: inherit                   # OPTIONAL — opus | sonnet | haiku | <full-id> | inherit (default)
---

You are a senior design reviewer for Siteline.
<the rest of the system prompt goes here>
```

Verified field semantics:

- **Only `name` and `description` are required.** Everything else is optional.
- **`tools` is an allowlist.** Omit it → the agent inherits every tool. For reviewers/auditors,
  restrict to `Read, Grep, Glob` (optionally `Bash`) so they "analyze without modifying."
- **`model`** accepts `opus`, `sonnet`, `haiku`, a full model id, or `inherit` (the default —
  same model as the main conversation). Route cheap mechanical work to `haiku` to control cost.
- **Each subagent starts with a fresh, isolated context window.** It does **not** see your
  conversation history, files already read, or skills already invoked — so the delegation
  prompt must carry the context it needs.
- **Subagents cannot spawn other subagents.** Chain them from the main thread, or use Skills.
- **Built-in `skills:` field** lets an agent preload your conventions at startup
  (e.g. `skills: [supabase-conventions]`) instead of you hardcoding a stack into the prompt.
- **`/agents` is the recommended way to create/manage them** — interactive, has a
  "Generate with Claude" option, and applies immediately (editing files on disk needs a restart).

Two more optional fields worth knowing ([sub-agents]):
- **`memory: project`** — gives an agent persistent memory across sessions (stored under
  `.claude/agent-memory/<name>/`, shareable via git). Useful for an agent that should accumulate
  codebase knowledge over time — but it's the *same staleness liability* as baked-in facts, so
  treat it as something to review, not trust blindly (see Part 8.5).
- **`isolation: worktree`** — runs the subagent in a temporary git worktree. Good for parallel
  experiments that shouldn't touch your working tree; the worktree is cleaned up if unchanged.

---

## Part 3 — The proactivity doctrine (how to make agents *smart and proactive*)

This is the part you asked for most directly. "Smart and proactive" is not magic — it comes
from five concrete, documented levers.

### 3.1 The `description` is the brain of delegation
Claude decides whether to hand a task to an agent **based almost entirely on the `description`
field.** A vague description = an agent that never fires. Rules:

- **Lead with a trigger, not a noun.** `"Use proactively after editing any component to…"`
  beats `"A design reviewer."`
- **Include the literal phrase "use proactively"** (or "PROACTIVELY") — the docs state this
  encourages Claude to delegate without being asked. ([sub-agents])
- **Name the triggering events**: "after a migration", "before opening a PR", "when a fetch/auth
  flow changes", "whenever tests are added".
- **Disambiguate from other agents.** If a human couldn't say which of two agents should handle
  a task, Claude can't either. ([writing-tools])

**Bad:** `description: Reviews code.`
**Good:** `description: Use PROACTIVELY immediately after any code change to review the git diff
for correctness, security, and our conventions before the work is reported done.`

### 3.2 Single responsibility = sharper behavior
"Design focused subagents — each should excel at one specific task." ([sub-agents]) A broad
"do-everything" agent reasons worse than three narrow ones. Prefer a roster of specialists.

### 3.3 Tool scoping makes them safer *and* more focused
Restricting tools isn't just security — it focuses the agent. A reviewer with only
`Read, Grep, Glob` can't wander off and start editing. A read-only DB agent literally cannot
run a destructive query (see the `db-reader` hook template in Part 6).

### 3.4 Model routing controls cost and quality
`haiku` for mechanical/parallel work (test scaffolding, lint sweeps), `sonnet`/`opus` for hard
reasoning (architecture, security review). Default `inherit` is fine when unsure.

### 3.5 The system-prompt body sets the "altitude"
Aim for prompts "specific enough to guide behavior, yet flexible enough to give strong
heuristics." ([context-engineering]) Two failure modes to avoid: brittle hardcoded logic, and
vague hand-waving. Give the agent: a **role**, a **concrete procedure**, **what to read** (point
it at `.claude/rules/*` and key files), and an **output format**.

> **Anti-over-reporting rule (important for review agents):** A reviewer told to "find problems"
> will always find some, even in sound code — which leads to over-engineering. Always add:
> *"Flag only gaps that affect correctness or our stated requirements."* ([best-practices])

### 3.6 Wire proactivity end-to-end — and be honest about what's guaranteed
There are two tiers of "proactive," and they are **not** equally reliable:

- **Probabilistic (description-driven auto-delegation).** A good `description` (3.1) makes Claude
  *usually* delegate. This is the default mechanism, but it is advisory — Claude decides, and it
  won't fire every time. Good enough for reviewers you want "most of the time."
- **Deterministic (hook-driven).** If an agent MUST run on a specific event (e.g. review every
  diff before a commit), don't rely on wording — fire it from a **Hook**. Hooks are deterministic.

**Concrete auto-fire pattern** — run the code-reviewer automatically when a session stops, via a
`Stop` (or `SubagentStop`/`PostToolUse`) hook in the repo's `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      { "matcher": "", "hooks": [
        { "type": "command", "command": ".claude/hooks/run-reviewer.sh" }
      ] }
    ]
  }
}
```
…where `run-reviewer.sh` checks for an unreviewed diff and invokes the reviewer (e.g. via
`claude --agent code-reviewer -p "review the staged diff"` or by emitting a prompt the session
picks up). **Caveat:** hooks live in committed `.claude/settings.json` — and **plugin-distributed
agents ignore the `hooks` field**, so a hook-fired agent must be a committed project agent, not a
plugin one (see Part 8). Verify the hook actually fires (Part 11) — a hook that silently no-ops is
worse than none.

> Rule of thumb: use `description` proactivity for "nice to have, most of the time," and a hook
> for "this must happen every time." Don't claim an agent is proactive until you've *watched* it
> fire unprompted.

---

## Part 4 — The per-project build workflow (run this in each repo)

This is the repeatable process. Do it inside each project on the server, one project at a time.

### Phase 0 — Prerequisites
- Open a Claude Code session at the project root.
- Confirm the stack: `package.json` / `pyproject.toml` / `go.mod`, test runner, lint config.

### Phase 1 — Understand the project (don't skip — this is what makes agents *smart*)
0. **Validate existing rules BEFORE trusting them.** ⚠️ Critical: drop-in template rules are
   common and dangerous. Open every file in `.claude/rules/` and ask: *does this actually describe
   THIS product and codebase?* Watch for tells that a file was copied from elsewhere — wrong
   product/persona language, references to directories or tooling that don't exist in this repo,
   generic boilerplate. **Agents pointed at borrowed rules will learn the wrong conventions.**
   Reconcile or delete mismatched rules first. (Concrete example: salt-home-studio-web shipped a
   `design-guidelines.md` written for "a construction foreman" and a `naming-conventions.md` for
   "claude-mods components" — neither describes this app. Those must be fixed before any agent
   maps to them.) Do **not** point agents at a rule you haven't confirmed is real for this repo.
1. **Run `/init`.** It analyzes the codebase and writes a starter `CLAUDE.md` (build system, test
   framework, conventions). Treat the output as a **first draft, not gospel** — review and prune.
2. **Interview the codebase** like a senior engineer joining the team. Ask:
   - "Walk me through the architecture. Trace one request from entry to response."
   - "How does auth/data access work? Where are the integration boundaries?"
   - "What are the naming, error-handling, and testing conventions here?"
   - "What are the non-obvious gotchas a new dev would trip on?"
3. **Codify confirmed answers** into `CLAUDE.md` / `.claude/rules/` (keep it lean — Part 7).
   Only encode facts you verified at `file:line` — guessed conventions become stale lies.

### Phase 2 — Decide the agent roster for *this* project
Pick specialists that match the stack and the work you repeat. A good default roster:

| Agent | Role | Tools | Model |
|-------|------|-------|-------|
| `code-reviewer` | Reviews the diff before "done" | `Read, Grep, Glob, Bash` | inherit |
| `design-reviewer` | UI vs. design guidelines (frontend projects) | `Read, Grep, Glob` | inherit/opus |
| `test-writer` | Generates tests in the project's framework | `Read, Grep, Glob, Edit, Write, Bash` | haiku |
| `debugger` | Root-cause + minimal fix for failures | `Read, Grep, Glob, Edit, Bash` | inherit |
| `{stack}-expert` | DB/API/payments specialist (e.g. supabase-expert) | scoped to its domain | sonnet |
| `security-reviewer` | Auth, secrets, input validation, injection | `Read, Grep, Glob, Bash` | opus |
| `reporting-analyst` | Answers data questions via governed definitions (read-only) | `Read, Grep, Glob, Bash` (read-only) | sonnet |

> The `reporting-analyst` is a **data-facing** agent — it has a different, harder failure profile
> than the code agents. Build it per **Part 9** (governed definitions, a retrieval router, and
> offline evals), not just a prompt.

> Add an agent only when you'd otherwise "keep spawning the same kind of worker with the same
> instructions." ([sub-agents]) Don't pre-build agents you won't use.

### Phase 3 — Write the agents
Use `/agents` → "Create new agent" (Personal vs. Project) → optionally "Generate with Claude",
**or** drop files into `.claude/agents/` from the Part 6 templates. Wire each to the project's
rules via the prompt body and/or `skills:`.

### Phase 4 — Verify they actually work (mandatory)
- Trigger each agent on a real task ("review this diff", "write a test for X") and confirm it
  fires and produces useful output.
- For any agent with a hook (e.g. read-only DB), confirm the hook blocks the forbidden action.
- Confirm the `description` triggers **auto-delegation** — make a small change and see if the
  reviewer fires without being named.

### Phase 5 — Commit
Project agents live in `.claude/agents/` and **belong in version control** so the team shares
them. Commit with a conventional message (`feat(agents): Add code-reviewer and supabase-expert`).

---

## Part 5 — The kickoff prompt (paste this into each project's session)

This lets the project's own Claude session do Phases 1–3 for you, using its full view of that
codebase. Edit the bracketed bits per project.

```
You are setting up custom Claude Code subagents for THIS project ([PROJECT NAME]).
Follow our guidebook at docs/AGENT-BUILDING-GUIDE.md (or the copy in our shared plugin).

Work in phases and check in with me between each:

PHASE 1 — UNDERSTAND
- FIRST: read every .claude/rules/* file and tell me if any looks like a borrowed
  template that does NOT describe this product (wrong persona, references to tools/
  dirs that don't exist here). Flag mismatches and STOP for my decision before using
  them — do not point agents at unverified rules.
- If CLAUDE.md is missing or thin, run /init, then refine it.
- Explore the architecture, data layer, integration boundaries, and conventions.
- Summarize: stack, key directories, request flow, naming/error/test conventions,
  and the top 5 non-obvious gotchas. Cite file:line for each claim. Mark anything you
  could not verify as "unverified" rather than guessing.

PHASE 2 — PROPOSE A ROSTER
- Recommend 3–6 specialist subagents tailored to THIS stack and the work we repeat.
- For each: name, one-line proactive description, tools (allowlist), model, and why.
- Map each to our existing .claude/rules/* so they encode how we build and design.

PHASE 3 — BUILD (after I approve the roster)
- Write each agent to .claude/agents/ as Markdown + YAML frontmatter.
- Descriptions MUST include a concrete proactive trigger ("Use proactively when…").
- Restrict tools appropriately; reviewers are read-only.
- Reviewers must "flag only gaps that affect correctness or our stated requirements."
- Where a stack convention exists, bind it via the skills: field or by pointing the
  prompt at the relevant rule file.

PHASE 4 — VERIFY
- Trigger each agent on a real task and show me it fires and is useful.
- Confirm auto-delegation works (make a small change; see if the reviewer fires unprompted).

PHASE 5 — COMMIT
- Commit the new .claude/agents/ files with a conventional commit message.

Be proactive and thorough. Prefer a few sharp specialists over one broad agent.
```

---

## Part 6 — Template library (copy-paste)

> These are Anthropic's canonical patterns ([sub-agents]) adapted to Siteline. Drop into
> `.claude/agents/<name>.md`. Tune the body and the `skills:`/rule pointers per project.

### `code-reviewer.md`
```markdown
---
name: code-reviewer
description: >
  Use PROACTIVELY immediately after any code change, and before any work is reported
  done or a PR is opened. Reviews the git diff for correctness, security, and our conventions.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer for Siteline. When invoked:

1. Run `git diff` (or `git diff --staged`) and focus only on the modified files.
2. Read .claude/rules/* to load our conventions before judging anything.
3. Review against this checklist:
   - Correctness: does it do what was asked? Edge cases handled?
   - Naming and structure match the surrounding code
   - No duplicated logic; reuse existing helpers
   - Error handling and input validation present
   - No exposed secrets or credentials
   - Adequate test coverage for the change
4. Output organized as: Critical (must fix) / Warnings (should fix) / Suggestions (nice to have).

Flag ONLY gaps that affect correctness or our stated requirements. Do not over-report or
suggest speculative refactors. If the diff is sound, say so plainly.
```

### `design-reviewer.md` (frontend projects)
```markdown
---
name: design-reviewer
description: >
  Use PROACTIVELY after any UI, component, or styling change. Audits the result against
  our design guidelines and runs the "AI Slop" test before the work is considered done.
tools: Read, Grep, Glob
model: opus
---

You are a senior product designer reviewing UI for Siteline.

1. Read .claude/rules/design-guidelines.md and load our standards.
2. Run `git diff` to see what changed; read the affected components.
3. Run the AI Slop test: would someone immediately say "AI made this"? Check for the
   documented anti-patterns (gradient text, glassmorphism, identical card grids,
   centered-everything, every-button-primary, pure black/white, etc.).
4. Verify the 8 interactive states, motion timings, accessibility (focus rings, ARIA,
   color-contrast), and responsive/touch-target rules from the guidelines.

Output: Critical / Warnings / Suggestions. Flag only real deviations from our standards.
Reference the specific guideline section for each finding.
```
> **Reconcile with existing skills before adopting:** if a project already has a design-review
> workflow (e.g. the Impeccable `/critique`, `/audit`, `/harden` skills referenced in some of our
> `.claude/rules/`), don't duplicate it. Either have this agent **invoke** those skills (preload
> via `skills:`) or skip the agent and use the skills directly. One design-review path, not two.

### `test-writer.md`
```markdown
---
name: test-writer
description: >
  Use PROACTIVELY when new logic is added without tests, or when asked to add coverage.
  Writes tests in this project's existing framework and matches existing test style.
tools: Read, Grep, Glob, Edit, Write, Bash
model: haiku
---

You write tests for Siteline projects.

1. Detect the test framework and existing test conventions (read a few neighboring tests).
2. Cover the happy path, edge cases, and the error path — not just the obvious case.
3. Match the existing naming, structure, and assertion style exactly.
4. Run the test suite and ensure your new tests pass before finishing.

Never weaken an assertion to make a test pass. If code is untestable as written, say so.
```

### `debugger.md`
```markdown
---
name: debugger
description: >
  Use PROACTIVELY when a test fails, the build breaks, or an error/stack trace appears.
  Finds the root cause and applies a minimal fix.
tools: Read, Grep, Glob, Edit, Bash
model: inherit
---

You are a debugging specialist. Process: capture the error → reproduce it → isolate the
cause → apply the minimal fix → verify the fix resolves it and breaks nothing else.

Fix the underlying issue, not the symptom. Explain the root cause in one or two sentences.
```

### `supabase-expert.md` (example stack specialist — adapt per project)
```markdown
---
name: supabase-expert
description: >
  Use PROACTIVELY for any Supabase work — schema changes, RLS policies, migrations, edge
  functions, and client queries. Knows our data-access conventions.
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
---

You are the Supabase specialist for this codebase.

- Migrations live in supabase/migrations/. Never edit the database directly.
- All tables have RLS enabled; default-deny, then add explicit policies.
- Client queries go through our integrations/supabase layer and react-query hooks —
  never raw fetch in components.
- For any data/auth/storage change, trace the full chain (UI → query → RLS → result) and
  verify RLS before declaring done.

Read the project's CLAUDE.md and .claude/rules/* for project-specific data conventions.
```

### `db-reader.md` (read-only DB agent — enforced by a hook, not just a prompt)
```markdown
---
name: db-reader
description: >
  Use PROACTIVELY to answer questions about live data via read-only SQL. Cannot mutate data.
tools: Bash
model: haiku
hooks:
  PreToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: .claude/hooks/validate-readonly-query.sh
---

You answer questions by running READ-ONLY SQL (SELECT only). A safety hook blocks any
INSERT/UPDATE/DELETE/DROP/ALTER/TRUNCATE. Explain results in plain language.
```
With `.claude/hooks/validate-readonly-query.sh` exiting code `2` on any write keyword.
> Note: **plugin-distributed agents ignore `hooks`/`mcpServers`/`permissionMode`** for security
> ([plugins]). An agent that *needs* a hook must be committed directly into `.claude/agents/`,
> not shipped via the shared plugin.

### `security-reviewer.md`
```markdown
---
name: security-reviewer
description: >
  Use PROACTIVELY when auth, payments, file uploads, env vars, or any external input
  handling changes. Audits for security issues before merge.
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior security engineer. Review the diff for: auth/authorization gaps, exposed
secrets, missing input validation, injection (SQL/command/XSS), insecure direct object
references, and unsafe redirects. For each finding give severity, the file:line, and the fix.

Flag only real, exploitable issues — not theoretical concerns. If clean, say so.
```

### `reporting-analyst.md` (data-facing — build with Part 9, not just this prompt)
```markdown
---
name: reporting-analyst
description: >
  Use PROACTIVELY to answer questions about our data (projects, leads, invoices, calendar) with
  read-only SQL. Maps each question to a governed definition before querying.
tools: Read, Grep, Glob, Bash
model: sonnet
hooks:
  PreToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: .claude/hooks/validate-readonly-query.sh
---

You answer data questions for Salt Home Studio. A safety hook blocks any non-SELECT SQL.

Process (do NOT skip step 1):
1. Resolve the question to a GOVERNED definition first. Read .claude/rules/data-model.md. If the
   concept (e.g. "active project", "qualified lead", "revenue") has a definition there, use it
   verbatim. If it's ambiguous and undefined, ASK a clarifying question — do not guess.
2. Open the reference doc for the relevant domain to find the right table, grain, and gotchas.
3. ALWAYS apply the org/RLS hygiene filter (scope to the current org_id).
4. Run the read-only query. State the definition you used and any assumptions, so the asker can
   sanity-check what you measured — they're asking because they don't already know the answer.

Never invent a metric definition. A confident wrong number is the worst possible output.
```
> This agent is only as good as its governed definitions, reference docs, and evals — see Part 9.
> Keep it **committed per-repo** (it relies on a hook, which plugin-distributed agents drop).

---

## Part 7 — Right-sizing always-on context (CLAUDE.md / rules)

Agents only stay sharp if the always-on layer stays lean. Documented guidance:

- **Target CLAUDE.md under ~200 lines.** "Longer files consume more context and reduce
  adherence." "Bloated CLAUDE.md files cause Claude to ignore your actual instructions." ([memory], [best-practices])
- **The pruning test:** for each line, ask "Would removing this cause Claude to make a mistake?"
  If not, cut it.
- **Include:** bash commands Claude can't guess, non-default style rules, test instructions,
  repo etiquette, project-specific architectural decisions, non-obvious gotchas.
- **Exclude:** anything readable from the code, standard conventions, detailed API docs (link
  instead), file-by-file descriptions, self-evident advice.
- **Move sometimes-relevant or long material to a Skill** (loads on demand) or a path-scoped
  `.claude/rules/*.md` (loads only when matching files are open). `@import` does **not** reduce
  context — imported files still load every session.

> **Action item for salt-home-studio-web:** `design-guidelines.md` has no `paths:` frontmatter,
> so its ~27KB loads into **every session** — a real, recurring context cost. Split it into a
> `/design-review` **Skill** (loaded on demand) plus a short always-on rule, and let the
> `design-reviewer` agent pull in the full guide when it runs. (Also: this file is a borrowed
> template — fix its content per Phase 1 step 0 before relying on it.)

---

## Part 8 — Sharing agents across MoneyWell, ProPunch, and beyond

There are three scopes; precedence on a name conflict is
**managed > `--agents` CLI > project `.claude/agents/` > user `~/.claude/agents/` > plugin.**

| Approach | When to use | Caveats |
|----------|-------------|---------|
| **Per-repo `.claude/agents/`** (committed) | Genuinely project-specific agents | Copy-paste drift across repos; no central update |
| **User `~/.claude/agents/`** | Solo, single-machine personal helpers | **Not shared via git; does NOT survive cloud/CI sandboxes** (sandboxes ignore `~/.claude`) |
| **Plugin marketplace** ✅ | Shared Siteline-wide agents across 3+ repos | The Anthropic-endorsed path; versioned, namespaced, centrally updatable |

> **Why a plugin, not `~/.claude/agents/`:** the docs are explicit that cloud sandbox sessions
> "don't pick up user-level configuration from your host, such as `~/.claude`." For Claude Code
> on the web / CI, only repo-committed `.claude/` and installed plugins reach the session.
> ([sandboxing], [sub-agents])

### Recommended setup for Siteline (do this once)
1. Create a private repo `sitelineco/claude-plugins` with `.claude-plugin/marketplace.json` and a
   plugin bundling the shared `agents/`, `skills/`, `rules/`, and `hooks/`. Set an explicit `version`.
2. In **each** consuming repo, **create** `.claude/settings.json` if it doesn't exist (it does not
   in salt-home-studio-web yet) and register the marketplace. **Pin a version** so a bad push can't
   hit every repo at once:
   ```json
   {
     "extraKnownMarketplaces": {
       "siteline-tools": {
         "source": { "source": "github", "repo": "sitelineco/claude-plugins" }
       }
     },
     "enabledPlugins": ["siteline-agents@siteline-tools"]
   }
   ```
   > Note: registering the marketplace prompts teammates to **trust and install** on first use —
   > it is not a silent auto-install. **Verify it actually loads in a fresh session** (Part 11):
   > open a clean session in the repo and confirm `/agents` lists the shared agents. Don't assume.
3. **Staged rollout.** Bump the plugin `version`, roll it to **one repo first**, confirm the
   agents still behave, then advance the other repos' pins. Never let all three track `latest`.
   To roll back, repin the affected repo to the previous version. Keep truly repo-specific agents
   (and any hook-dependent agents) in that repo's own `.claude/agents/`.

> **Split strategy:** Shared = the "how Siteline builds/designs" agents (code-reviewer,
> design-reviewer, security-reviewer, test-writer) → plugin. Project-specific = anything that
> knows one app's schema/architecture, or needs a hook/MCP → committed `.claude/agents/` in that repo.

> **⚠️ Don't ship write/Bash agents through the plugin lightly.** Plugin agents **cannot** carry a
> `hooks` guardrail (the field is stripped for security), so a plugin-shipped `Bash`/`Edit`/`Write`
> agent runs unsandboxed in every repo and CI session that enables it — a supply-chain surface.
> Keep `test-writer`, `debugger`, `supabase-expert`, and `db-reader` (anything with write or Bash)
> as **committed per-repo agents** where you can scope and hook them. Ship only **read-only**
> reviewers (`Read, Grep, Glob`) through the plugin.

---

## Part 8.5 — Keeping agents alive: maintenance, cost, and measurement

Shared-agent setups don't fail on day one — they rot over a quarter. Build the maintenance in now.

### Staleness (the silent killer)
Agents encode "how the codebase works." Code drifts; the agent's assertions go stale and start
giving **false confidence** (e.g. a `supabase-expert` that insists "all tables have RLS enabled"
after that stops being true is worse than no agent).

- **Assign an owner** to each shared agent. Unowned agents rot.
- **Re-verify trigger:** after any major refactor or architecture change, re-run Phase 4 on the
  affected agents. Add this to your refactor checklist.
- **Prefer pointers over baked-in facts.** Have agents *read* `.claude/rules/` and `CLAUDE.md` at
  runtime rather than hardcoding architectural claims in the prompt — then there's one place to
  update. The more a prompt asserts about the codebase, the bigger its maintenance liability.

### Cost & latency (the proactive doctrine has a bill)
Auto-firing `opus` reviewers on every change across a team is real spend and a real tax on the
inner loop. Guidance:

- **Reserve `opus` reviewers for PR-time / pre-merge**, not every keystroke-level change.
- For an always-on inner-loop reviewer, use `haiku` or `sonnet`.
- Don't auto-fire more than one or two reviewers per event; run the rest at PR time.

### Measurement (does the agent earn its keep?)
"It fired and produced output" (Part 11) is a smoke test, not value. Lightweight loop:

- Track, informally, when a review agent's finding led to a **real fix** vs. was dismissed as noise.
- **Prune agents that don't earn their keep.** The roster should shrink as well as grow. A reviewer
  that mostly emits dismissed suggestions is adding latency, cost, and alert fatigue — cut it.
- For **data-facing agents**, this informal loop isn't enough — use the **offline-eval + ablation**
  discipline in **Part 9.6** to measure accuracy quantitatively and catch regressions.

---

## Part 9 — Grounding & accuracy for data-facing agents (analytics, reporting, DB)

Most of this guide assumes agents that write code — an open solution space where tests and the
compiler are natural guardrails. But some agents **answer questions about data**: a reporting
agent ("how many active projects closed in Q2?"), the `supabase-expert`, the `db-reader`. These
have a different, harder failure profile — often there's **one correct answer, no test proves it,
and the user can't validate it** (they're asking *because* they don't know). This Part adapts
Anthropic's own analytics-agent playbook to our scale.

> **Source & honest scoping:** these practices come from Anthropic's "How Anthropic enables
> self-service data analytics with Claude" (June 2026), which describes a warehouse with millions
> of fields across many teams. Salt Home Studio is one app, ~60 Supabase tables, one org. So we
> take the transferable kernel — governed definitions, retrieval routing, staleness defense, and
> evals — and **skip** the heavy machinery (semantic layer, knowledge graph, query-corpus
> retrieval, multi-surface sync). See §9.7 for when to revisit those.

### 9.1 The reframe: accuracy is a context + verification problem, not code-gen
The hard part of a data question is **mapping the question to the right governed entity**, not
writing the SQL — once the entity is unambiguous, the query is trivial. The post's most important
result is a *negative* one: they gave an agent grep access to thousands of prior queries and
verified it read them — accuracy moved **less than one point**. The information was present, the
agent saw it, and still didn't use it. The bottleneck was **structure, not access**. Implication
for us: don't fix data-agent accuracy by dumping more context at it — invest in governed
definitions and routing. (This is the same lesson that made fixing `.claude/rules/` high-leverage.)

### 9.2 The three failure modes (adapted to our model)
1. **Concept ↔ entity ambiguity.** One concept maps to several plausible tables/columns with
   subtly different meaning. Our real examples: what counts as an **active project** (status?
   has activity in N days?), a **qualified lead** (which pipeline stages?), **recognized revenue**
   (invoice totals vs. paid line items vs. quotes?), and is everything correctly **`org_id`-scoped**?
   → Fix: one governed definition per concept (§9.3).
2. **Staleness.** Schema, generated `types.ts`, and business definitions drift; the agent's
   doc/knowledge goes subtly wrong. → Fix: colocation + a CI hook (§9.5).
3. **Retrieval failure.** The answer is in the model but the agent doesn't find it among 60 tables.
   → Fix: a router skill that narrows to a few curated docs (§9.4).

### 9.3 Governed definitions (a human owns the meaning)
- Keep a **small set of canonical definitions** for the ambiguous concepts — e.g. *"active
  project = status in ('in_progress','installing') AND updated within 90 days"* — each clearly
  owned. Store them in a `.claude/rules/data-model.md` (always-on if short, or a Skill if long).
- **Generate drafts with Claude, but a human owns the final definition.** The post tried
  auto-generating metric definitions from raw tables/query logs; it "encoded the very ambiguities
  we were trying to eliminate" and was net-negative on evals. A small human-curated set beat it.
- Annotate **the source, not the generated file** — `src/integrations/supabase/types.ts` is
  generated, so put definitions in migrations / column comments / the data-model rule, never in
  `types.ts`. Always state the **`org_id` + RLS** filter as part of the definition.

### 9.4 The router pattern + reference docs written for LLM retrieval
Rather than let a data agent grep all 60 tables, give it a **thin "knowledge" router**: *for a
data question — (1) use a governed definition if one exists; (2) else open the reference doc for
that domain; (3) always apply the org/RLS hygiene filter.* The router points to **one short
reference doc per domain** (projects, leads, invoices, calendar). Reference-doc skeleton (adapted
from the post — write it for an LLM reader, not a human tutorial):

```markdown
# [Domain] data — e.g. Leads

## Quick reference
- Business context: [what this domain means in plain words]
- Entity grain: [what one row represents — e.g. "one lead per inquiry"]
- Standard hygiene filter: [the filter EVERY query applies — for us, always `org_id = <current>`]

## Key tables
### [table_name]
- Grain: [...] · Scope/exclusions: [...]
- Usage: [when to use, when NOT to, join keys, required filters]

## Dimensions
- [How key dimensions/statuses are encoded; where the same concept is named differently]

## Gotchas
- [The wrong-answer modes a senior person would warn about — e.g. "leads can have null org_id
  from the public create-lead function; exclude or attribute them deliberately"]

## Routing triggers
- IF the question is about [X] → use [table/definition]. DO NOT use [Y] for [Z].

## Cross-references
- [Neighboring domain docs that own adjacent questions]
```

### 9.5 Staleness defense: colocation + a CI hook (the strongest lesson)
The post watched offline accuracy drift from **~95% to ~65% in a single month** when docs went
untended — then treated it as an engineering problem:
- **Colocate** the data-model rule / reference docs **in this repo**, next to `supabase/migrations`
  and `src/integrations/supabase`. The PR that changes a model is the PR that updates its doc.
- **Add a CI / code-review hook** that flags any PR touching `supabase/migrations/**` or
  `src/integrations/supabase/types.ts` **without** touching the data-model docs. At Anthropic ~90%
  of data-model PRs now include a doc change in the same diff. (This also makes our `code-reviewer`
  agent's job concrete — add the check to its checklist.)
- Prune doc scaffolding as models improve and old failure modes stop applying.

### 9.6 Validation: offline evals + ablation (fills the audit's measurement gap)
This is the answer to "does the agent actually help?" Right-sized for us:
- **Offline evals = a handful of Q&A pairs per domain** ("active projects closed in Q2 2026" → a
  known number). A dozen or two per complex domain is plenty — diminishing returns past that, and
  the ceiling drops with each new model generation.
- **Anchor ground truth so it can't drift:** pin each eval to a **snapshot date** / a stable fact
  table, or **grade the agent's query rather than its number**. Otherwise the eval rots the moment
  the data moves.
- **Ablate at PR granularity:** when you change a rule/skill/reference doc, **re-run the affected
  eval slice and put the before/after delta in the PR description.** This keeps "I improved the
  docs" honest and catches the common case where a well-meant addition makes things *worse*.
- **Store results as telemetry, not test logs:** one row per run (date, git SHA, model id,
  per-eval pass/fail). "Did that change help?" becomes a query, and you catch slow regressions.
- **Gate per domain:** don't tell stakeholders the reporting agent is trusted for, say, "leads"
  until that eval slice clears a threshold (~90%).
- **Harvest corrections:** every time someone corrects the agent in a thread, that correction is a
  candidate eval. This is your cheapest, highest-signal source of new evals.

### 9.7 What we deliberately skip (and when to revisit)
Skip for now: a compiled **semantic layer**, a company **knowledge graph**, **query-corpus
retrieval**, and **multi-surface auto-sync**. Revisit if/when: you onboard **multiple orgs/clients**
(true multi-tenant analytics), you have **several analysts** asking overlapping questions, or
**metric/dashboard sprawl** appears (the same number computed differently in two places). Until
then, governed definitions + a router + a few evals get you most of the value.

---

## Part 10 — Pitfalls (named, documented)

- **Over-broad agents.** "Each subagent should excel at one specific task." Split them.
- **Weak descriptions** → agents never fire. Lead with a proactive trigger (Part 3.1).
- **Reviewers that over-report.** Add "flag only correctness/requirement gaps."
- **Bloated CLAUDE.md** → rules get ignored. Keep it under ~200 lines.
- **Forgetting subagents start blank.** They don't see your conversation or files already read —
  pass needed context in the delegation. (Built-in Explore/Plan even skip CLAUDE.md, so restate
  critical rules like "ignore vendor/" in the prompt.)
- **Expecting nesting.** Subagents can't spawn subagents — chain from the main thread.
- **Relying on `~/.claude` in cloud sessions** — it isn't there. Use the plugin/committed config.
- **Plugin agents silently dropping hooks/MCP/permissionMode** — commit those agents directly.
- **Hooks vs. prompts.** "Always do X" as a CLAUDE.md line is advisory. If it MUST happen every
  time, make it a Hook.
- **Building agents on borrowed/template rules.** Validate `.claude/rules/` describe THIS product
  before pointing agents at them (Phase 1 step 0). Wrong rules → agents that confidently apply
  the wrong conventions.
- **Stale agents giving false confidence.** An agent asserting an out-of-date architectural fact
  is worse than no agent. Assign owners; re-verify after refactors (Part 8.5).
- **Cost creep from proactive opus reviewers.** Keep heavyweight reviewers at PR-time; use
  haiku/sonnet for always-on (Part 8.5).
- **Shipping write/Bash agents via plugin.** Plugins strip the `hooks` guardrail — keep those
  agents committed per-repo (Part 8).
- **Data-facing agents that guess.** A reporting agent without governed definitions answers what
  was *asked*, not what was *meant*, and the user can't tell. Resolve to a governed definition or
  ask — never invent a metric (Part 9).
- **Adding context to fix a data agent.** The bottleneck is structure, not access — more context
  rarely helps; governed definitions + routing do (Part 9.1).
- **Evals that drift.** An eval written against live data rots when the number moves. Anchor to a
  snapshot or grade the query (Part 9.6).

> **Naming note:** this repo's `naming-conventions.md` uses `{domain}-expert.md` for agents. The
> roster here adds role-based workers (`code-reviewer`, `test-writer`, `debugger`) that don't fit
> that pattern. That's fine — `{domain}-expert` for stack specialists, `{role}` for cross-cutting
> workers — but pick one convention per category and apply it consistently.

---

## Part 11 — Verification checklist (before calling a project "done")

- [ ] Existing `.claude/rules/` were validated as describing THIS product (Phase 1 step 0), not borrowed templates
- [ ] `CLAUDE.md` exists, is under ~200 lines, and was pruned (not raw `/init` output)
- [ ] Each agent has a proactive, disambiguated `description`
- [ ] Reviewers/auditors are tool-restricted (read-only) and told not to over-report
- [ ] Each agent was triggered on a real task and produced useful output
- [ ] Auto-delegation works (a small change makes the relevant reviewer fire unprompted)
- [ ] Any hook-backed agent's hook actually **fires** (auto-run) and **blocks** the forbidden action — verified by watching it, not assumed
- [ ] If using the shared plugin: opened a **fresh/clean session** and confirmed `/agents` lists the shared agents (sandbox-load works)
- [ ] No write/`Bash` agent is shipped via the plugin (those stay committed per-repo)
- [ ] Each shared agent has an **owner** and a re-verify-on-refactor note
- [ ] `opus` reviewers are scoped to PR-time, not every inner-loop change (cost)
- [ ] Agents are committed to `.claude/agents/` with a conventional commit message
- [ ] Shared agents live in the plugin; repo-specific/hook agents live in the repo
- [ ] **Data-facing agents only:** ambiguous concepts have governed definitions; a retrieval router/reference docs exist; queries are org/RLS-scoped (Part 9.3–9.5)
- [ ] **Data-facing agents only:** a handful of snapshot-anchored offline evals exist and pass; the staleness CI hook is wired (Part 9.5–9.6)

---

## Part 12 — Sources

Official Anthropic documentation (`code.claude.com/docs` — older `docs.claude.com/en/docs/claude-code/*` URLs redirect here):

- **Subagents** — `https://code.claude.com/docs/en/sub-agents`
- **Memory / CLAUDE.md** — `https://code.claude.com/docs/en/memory`
- **Skills** — `https://code.claude.com/docs/en/skills`
- **Extend Claude Code (feature comparison)** — `https://code.claude.com/docs/en/features-overview`
- **Best practices** — `https://code.claude.com/docs/en/best-practices`
- **Plugins** — `https://code.claude.com/docs/en/plugins`
- **Discover plugins / team marketplaces** — `https://code.claude.com/docs/en/discover-plugins`
- **Settings** — `https://code.claude.com/docs/en/settings`
- **Sandboxing** — `https://code.claude.com/docs/en/sandboxing`

Anthropic engineering blog:

- **Building Effective Agents** — `https://www.anthropic.com/engineering/building-effective-agents`
- **Effective context engineering for AI agents** — `https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents`
- **Writing effective tools for agents** — `https://www.anthropic.com/engineering/writing-tools-for-agents`
- **Equipping agents for the real world with Agent Skills** — `https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills`
- **How Anthropic enables self-service data analytics with Claude** (basis for Part 9) — `https://www.anthropic.com/news/how-anthropic-enables-self-service-data-analytics-with-claude`

In-repo references cited by `[best-practices]`/`[context-engineering]` shorthand above map to the
two corresponding links in this section.

---

*Maintained for Siteline. When a project session corrects a pattern, fold the lesson back into
this guide so every future project benefits.*
