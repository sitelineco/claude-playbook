# Post-mortem — Salt Home Studio: stranded fixes & repo/deploy drift

**Status:** Resolved · **Severity:** High (live cross-tenant security vulnerabilities) ·
**Surface:** Salt Home Studio (marketing + client/team portal)

A case study for the agent-building playbook. The lessons are folded into the guide
(see "Deployment reality" there); this is the concrete story behind them.

---

## What happened

We spent a long session building agents and fixing real bugs in
`sitelineco/salt-home-studio-web` — a homepage hero that rendered empty, a broken portfolio
"View More," per-page SEO, and **five cross-tenant security holes** (invoice IDOR, an
unauthenticated invoice oracle, a phishing relay, privilege escalation, a data-export leak).
All were committed and, in the case of the portfolio, "verified" locally.

Then the user reported the portfolio *still* broken in production. Investigation revealed the
real problem: **none of it was live, and most of it never could be from that repo.**

- **The live site deploys from a different repo.** Production (`salthomestudio.com`, on Vercel)
  builds from **`SaltHomeStudio/salt-home-studio-web`** — a *different repo with the same name*,
  under a different GitHub owner than the one we were working in (`sitelineco/...`). The deployed
  Portfolio contained image-optimization code that **existed in no commit of our repo.**
- **The bug had a second layer.** The live "View More" de-duplicated photos by comparing
  *image-optimized* URLs; the optimizer collapses every URL's filename to the same value, so 100%
  of the extra photos were filtered out as "duplicates." (Separately, Supabase had stored the JPGs
  with `text/plain` metadata, so a mimetype guard would silently filter everything too.)
- **Edge functions deploy separately from the frontend.** Even once the security fixes landed in
  the right repo, the Vercel deploy did **not** touch them — Supabase edge functions require a
  separate `supabase functions deploy`. The vulnerabilities stayed live until that ran.

## Impact

- Live cross-tenant vulnerabilities (a client portal token could read/charge another client's
  invoice; an unauthenticated invoice-status oracle; a domain-spoofing phishing relay) remained
  exploitable for the entire period they were "fixed" in the wrong repo.
- Hours of work (hero, SEO, portfolio, security) were committed to a repo that does not deploy.

## Root causes

1. **We never mapped the deployment before building.** The methodology understood the *codebase*
   deeply but assumed the repo we were in was the source of truth. A handoff doc *claimed*
   "everything ships from this repo" — it was never verified, and was wrong.
2. **Two repos, same name, different owner.** `sitelineco/...` vs `SaltHomeStudio/...` looked
   identical; nothing prompted us to confirm which one Vercel actually builds.
3. **"Committed" was treated as "live."** Local verification proved the *code* worked, not that it
   was *serving users*.
4. **Frontend ≠ full deploy.** Vercel (frontend) and Supabase (edge functions, migrations) are
   separate deploy targets with separate triggers and secrets.

## How it was resolved

- Identified the real deploy repo (`SaltHomeStudio/salt-home-studio-web`) via response headers
  (`server: Vercel`) + the fact the deployed code wasn't in any of our commits.
- Re-fixed the portfolio in the live repo (de-dup on the **raw storage object name**, not the
  optimized URL; switched the mimetype guard to a **filename-extension** check).
- Re-applied the five security fixes in the live repo and deployed the edge functions to Supabase
  via a `workflow_dispatch` Action (after adding `SUPABASE_ACCESS_TOKEN`).
- **Independently verified live**, not just committed: probed the deployed functions and confirmed
  unauthenticated calls now return `401` *before* any side effect.

## Lessons (now in the playbook)

1. **Map the deployment before building agents** — host, the exact **owner/repo/branch** it builds
   from, and every separate deploy target (frontend, edge functions, DB migrations) + their secrets.
2. **"Committed" ≠ "live." Verify against the running system** — probe the deployed site/API, don't
   trust the diff.
3. **A repo is not authoritative until proven** — confirm the host actually builds from it; ignore
   handoff claims, verify.
4. **Edge functions / migrations deploy on their own track** — a green frontend deploy says nothing
   about them.
5. **De-dup and compare on raw identifiers**, never on transformed/optimized URLs.
6. **Don't trust stored mimetypes for user uploads** (Supabase stored JPGs as `text/plain`); gate on
   the filename extension.
