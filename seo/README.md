# SEO agent kit

EXPECT-grade SEO standards, packaged so any Siteline project inherits them. EXPECT (EXPECTseo.com) is
our in-house SEO engine and the **single source of truth**; these two files distill its productized
standards (real thresholds, JSON-LD field lists, intent taxonomy, linking rules) into a portable,
project-agnostic pair.

| File | Lands in the target repo as | Role |
|---|---|---|
| `seo-standards.md` | `.claude/rules/seo-standards.md` | The governed SEO definition — a human owns it. |
| `seo-auditor.md` | `.claude/agents/seo-auditor.md` | The agent that applies the standard and **verifies against the actual code**. |

## Adopt in a project

```bash
mkdir -p .claude/rules .claude/agents
cp ~/claude-playbook/seo/seo-standards.md .claude/rules/seo-standards.md
cp ~/claude-playbook/seo/seo-auditor.md   .claude/agents/seo-auditor.md
```

Then **fill the `«fill per project»` slots** in §0 of `seo-standards.md` (brand, canonical base URL,
business type, NAP, service areas, social profiles, topic silos) and the per-page keyword table in §1.
The auditor refuses to guess at unfilled values — it flags the standard as incomplete instead.

Invoke it after any public-page or blog change: *"Run the seo-auditor over the pages I changed."*

## Notes

- **Generic by design.** Keep these project-agnostic. Project-specific keywords/schema live in the
  *copied* `.claude/rules/seo-standards.md`, not here.
- **Not EXPECT's internal agents.** EXPECT's own `seo-specialist` / `ai-citation-strategist` audit
  EXPECT's pipeline output and stay in EXPECT's repo. This kit is for *other* sites' public pages.
- **Update once, inherit everywhere.** Improve a rule here; every project that re-copies (or, once
  shipped as a plugin, every project that pulls) gets it. When EXPECT changes a real threshold, update
  `seo-standards.md` to match so it stays the source of truth.
