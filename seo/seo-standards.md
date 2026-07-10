# SEO Standards (Siteline)

> **Governed definition — a human owns this file.** This is the single source of truth the
> `seo-auditor` agent enforces. It is distilled from EXPECT (EXPECTseo.com), Siteline's productized
> SEO engine — the thresholds, JSON-LD field lists, intent taxonomy, and linking rules below are the
> ones EXPECT actually checks in production, not generic best-practice. Keep it project-agnostic;
> each project fills the `«fill per project»` slots and may tighten (never loosen) a rule.
>
> **How to apply (for the LLM):** treat every rule as a check with a severity. A rule is only
> violated if you confirm it **in the code/markup**, not because this doc lists it. When a value is
> marked `«fill per project»` and the project hasn't filled it, flag that the standard is incomplete
> rather than inventing a value.

---

## 0. Per-project configuration (fill once, then enforce)

Fill these in the project's own copy before the auditor can fully apply the standard:

| Key | Value |
|---|---|
| Brand name (title suffix) | `«fill per project»` |
| Canonical base URL (absolute, https) | `«fill per project»` |
| Business type | `«fill per project: local-service \| multi-location \| ecommerce \| SaaS \| content/blog»` |
| NAP — Name / Address / Phone | `«fill per project»` (must be byte-identical everywhere it appears) |
| Primary service areas (`areaServed`) | `«fill per project»` |
| Social / authority profiles (`sameAs`) | `«fill per project»` |
| Primary topic silos (2–4 pillars) | `«fill per project»` |

> If **business type = local-service or multi-location**, the LocalBusiness rules in §7 are
> **required**. If **SaaS or content/blog**, LocalBusiness is N/A — use Organization + WebSite instead.

---

## 1. Per-page target keywords

**One page = one primary keyword.** No two indexable pages may target the same primary keyword
(keyword cannibalization — Critical; EXPECT blocks this at generation time). Each page gets 1
primary + 2–5 secondary/long-tail keywords, chosen to **match the page's search intent**.

Intent → page-type mapping (EXPECT's taxonomy — informational / commercial / transactional /
navigational):

| Page | Dominant intent | Primary keyword (fill) | Secondary keywords (fill) |
|---|---|---|---|
| Home | navigational + transactional | `«fill: brand + core service + locality»` | `«fill»` |
| Services / Products | transactional | `«fill: "[service] in [city]" / "[service] cost"»` | `«fill: "[service] near me", "[service] quote"»` |
| Individual service / product | transactional | `«fill: the single service term»` | `«fill»` |
| Portfolio / Work | commercial | `«fill: "[service] examples / portfolio / before and after"»` | `«fill»` |
| Process / How it works | informational→commercial | `«fill: "how [service] works", "[service] process"»` | `«fill»` |
| About | navigational | `«fill: brand + "[service] company [city]"»` | `«fill»` |
| Contact / Location | transactional + local | `«fill: "[service] [city]", "[service] near me"»` | `«fill: "[neighborhood] [service]"»` |
| Blog post | informational (commercial for "best/vs/review") | `«fill: the long-tail question/topic»` | `«fill: 3–5 related long-tail»` |

**Blog topic selection** (EXPECT's opportunity model — apply when proposing topics): favor keywords
with real volume, beatable difficulty, and **transactional/commercial intent weighted above
informational** (EXPECT bonus: transactional +5, commercial +3, informational 0, navigational −2).
A site that is **>80% informational** by clicks should add commercial/transactional pages
(best-X, X-vs-Y, pricing, comparison). Zero commercial content with meaningful traffic is a gap.

**Keyword placement (on-page):** primary keyword in the title, the single H1, the first 100 words,
the URL slug, and naturally 3–5× through the body (no stuffing). Secondary keywords woven in
subheadings/body.

---

## 2. Title tags

- **Length:** target **50–60 characters**. **Critical** if missing. **Important** if **> 60** (Google
  truncates) or **< 30** (under-using the slot). Measured on the rendered `<title>` / `document.title`.
- **Uniqueness:** every indexable page has a **unique** title. Duplicate titles = Important.
- **Formula:** `[Primary Keyword] – [Modifier or Benefit] | [Brand]`. Front-load the primary keyword;
  brand suffix last. Modifier = locality, year, outcome, or qualifier.
- **No** boilerplate-only titles ("Home", "Untitled", brand-only on inner pages).

## 3. Meta descriptions

- **Length:** target **120–160 characters**. **Important** if missing, **> 160** (truncated), or
  **< 80** (thin). 
- **Content:** include the primary keyword once + a clear CTA ("Get a free quote", "Book a visit").
  Written to earn the click, not to keyword-stuff. **Unique** per page.
- Meta description does **not** affect ranking directly but governs SERP CTR — treat low-CTR,
  well-ranked pages (position ≤3, CTR <4%; position 3–10, CTR <2%) as a rewrite opportunity.

## 4. Headings

- **Exactly one `<h1>` per page.** Zero H1 = **Critical**; more than one H1 = **Important**.
- The H1 contains the primary keyword and matches the page's search intent. The H1 is **not** the
  same string as the `<title>` verbatim (title carries the brand suffix; H1 reads naturally).
- **No hierarchy skips:** never jump H1→H3 without an H2 (any heading whose level exceeds the
  previous level by more than 1 = Important). H2 = main sections, H3 = subtopics.

## 5. Canonical, Open Graph & Twitter

| Tag | Requirement | Severity if wrong |
|---|---|---|
| `<link rel="canonical">` | **Self-referencing**, absolute `https://` URL = `{base}{path}`. Never canonicalise every page to the homepage. | Critical (missing → dupes) / Important |
| `og:title`, `og:description`, `og:url`, `og:type`, `og:site_name` | All present. `og:type` = `website` (pages) or `article` (posts). | Important |
| `og:image` | Absolute URL, **1200×630**. Article pages add `article:published_time` / `article:modified_time`. | Important |
| `twitter:card` | `summary_large_image` + `twitter:title`, `twitter:description`, `twitter:image`. | Nice-to-have |

## 6. Image alt text

- Every informational `<img>` has descriptive `alt`. Missing or empty `alt=""` on a meaningful image
  = a finding (**Important** if **>5** images affected on a page, else Nice-to-have).
- Decorative images use `alt=""` deliberately. Alt text describes the image + works in the primary
  keyword only where genuinely accurate.
- Images compressed (**target < 100KB**), modern format (WebP/AVIF), explicit `width`/`height` to
  prevent CLS.

---

## 7. Structured data (JSON-LD)

Emit JSON-LD in `<script type="application/ld+json">`. **Every field must have a real value — never
ship bare or placeholder schema.** Validate against validator.schema.org. Required coverage by page:

**Every page:** `Organization` (or `LocalBusiness`, below) + `BreadcrumbList`.
**Homepage:** also `WebSite` (enables the sitelinks search box).
**Blog/article pages** (`/blog`, `/post`, `/article`, `/news`): `BlogPosting` (or `Article`).
**FAQ pages / sections:** `FAQPage`.
**Product/shop pages:** `Product` (with `offers`).

### Organization (non-local: SaaS, ecommerce, content)
Required: `name`, `url`, `logo` (ImageObject). Recommended: `sameAs[]`, `description`, `foundingDate`,
`contactPoint` (`contactType`, `email`).

### LocalBusiness (required for local-service / multi-location)
Use the most specific subtype (`Plumber`, `HVACBusiness`, `Dentist`, `HomeAndConstructionBusiness`, …).
Fields:

| Field | Requirement |
|---|---|
| `@id` | stable URL anchor (e.g. `{base}/#business`) |
| `name` | = NAP name, exact |
| `image`, `logo` | absolute URLs |
| `url` | canonical site URL |
| `telephone` | = NAP phone, exact, E.164 or display-consistent |
| `priceRange` | `«fill per project»` (e.g. `$$`) |
| `address` | `PostalAddress`: `streetAddress`, `addressLocality`, `addressRegion`, `postalCode`, `addressCountry` — all = NAP |
| `geo` | `GeoCoordinates`: `latitude`, `longitude` `«fill per project»` |
| `areaServed` | `«fill per project»` — city/region list or `GeoCircle` |
| `openingHoursSpecification[]` | `dayOfWeek`, `opens`, `closes` `«fill per project»` |
| `sameAs[]` | GBP, Yelp, Facebook, and other authoritative profiles `«fill per project»` |
| `aggregateRating` | only if backed by **real, on-page** reviews — never fabricate |

> Multi-location: emit one LocalBusiness node **per location page**, each with its own `@id`, address,
> geo, and hours. NAP must be byte-identical to GBP and all citations.

### BlogPosting / Article
Required: `headline` (≤110 chars), `description`, `image` (ImageObject), `datePublished`,
`dateModified`, `author` (`Person`, or `Organization` for house byline), `publisher` (`Organization`
with `logo` ImageObject), `mainEntityOfPage` (`WebPage` `@id` = article URL). Recommended:
`wordCount`, `keywords`. `dateModified` must update when the article is materially edited.

### FAQPage
`mainEntity[]` of `Question` → `name` + `acceptedAnswer` (`Answer.text`). Only mark up Q&A that is
**visibly present** on the page.

### BreadcrumbList
`itemListElement[]` of `ListItem` (`position`, `name`, `item`). Position 1 = "Home". **Omit the
`item` URL on the last (current) item** per Google's spec.

---

## 8. Crawlability

- **`sitemap.xml`** exists, is valid (`<urlset>` or `<sitemapindex>`), lists every indexable page with
  accurate `<lastmod>`, and is declared in `robots.txt` (`Sitemap: {base}/sitemap.xml`). Missing or
  invalid sitemap = **Critical**.
- **`robots.txt`** exists: `Allow` public content; `Disallow` admin/api/auth paths (`/dashboard`,
  `/api`, auth/signup routes). Do **not** disallow CSS/JS or anything you want indexed.
- **AI crawlers:** unless the project deliberately opts out, **explicitly allow** `GPTBot`,
  `OAI-SearchBot`, `ChatGPT-User`, `ClaudeBot`, `Claude-SearchBot`, `PerplexityBot`, `Google-Extended`,
  `CCBot`. A blanket `Disallow: /` for these silently removes the site from AI answers (a growing
  discovery surface) — flag it.
- **`llms.txt`** at the root is a Nice-to-have for AI discoverability.
- **HTTPS** required; any `http://` `src`/`href`/`action` on an https page (mixed content) = Critical.
- **No `noindex`** (meta robots or `X-Robots-Tag`) on any page that should rank; `noindex` on a public
  money page = Critical. Public pages must return **200**, not 4xx/5xx.
- Each page reachable by **≥1 internal link** (no orphans — see §10).

---

## 9. Content quality (blog / landing pages)

- **Depth:** substantive pages target **~2000 words**, **≥1500** for pillar/competitive topics; flag
  thin content **< ~500 words** where depth is expected. Word count serves the query — don't pad.
- **Structure:** intro answers the query in the first 2–3 sentences (quotable for AI/featured
  snippets); H2/H3 outline; 2–3-sentence paragraphs; bullet/numbered lists; clear conclusion + CTA.
- **Entities & GEO/AEO:** bold key term definitions (`**[Term]**: …`); include a `## Frequently Asked
  Questions` block of **5–7** Q&As for FAQ-rich results; cite data with the source named inline.
- **Outbound links:** **2–4** to authoritative, **non-competing** sources (.gov/.edu/Wikipedia/major
  industry pubs). Never link competitors. Back factual claims with a source.
- **Freshness:** dates and "current year" references use the **actual current year**; bump
  `dateModified` on real edits.
- **No AI-voice tells** (Important for E-E-A-T credibility): avoid "In today's…", "Let's dive in",
  "In this comprehensive guide", "When it comes to", "It's important to note", "In conclusion",
  "game-changer"; don't start paragraphs with "Additionally/Furthermore/Moreover"; vary sentence
  length; use contractions. Never label content "AI-written" or surface "AI" in user-facing copy.

---

## 10. Internal linking

**Links per page (scale with length):**

| Page length | Internal links |
|---|---|
| < 1,500 words | 2–3 |
| 1,500–3,000 | 3–5 |
| 3,000–5,000 | 5–8 |
| Pillar (5,000+) | 8–15 |

A short article with 12 links "looks manipulated" — keep density natural. Place the **first internal
link in the first 300 words / top third**.

- **Anchor text:** descriptive, ≤ **8–10 words**, varied. Target mix ≈ 30% exact/close-match,
  40% partial/descriptive, 20% natural/topical, 10% branded. **Never** "click here" / "read more".
  Don't reuse the identical anchor for the same destination across pages.
- **No orphans:** every indexable page (esp. every article) has **≥1**, ideally **≥2**, inbound
  internal links. An article whose slug appears in no other page's body = **Important**.
- **Siloing (hub-and-spoke):** 2–4 pillar topics; each cluster article links **back to its pillar**
  and to **2–4 sibling** articles in the same silo. Don't cross-link unrelated silos heavily.
- **No link schemes:** no PBNs, no cross-site/cross-client backlink swaps, no paid do-follow links.
  Intra-site, editorial links only. Never recommend a link scheme — it's a Google spam violation.
- **Placement:** contextual inline links beat a footer "Related Articles" dump; never bunch many
  links in one paragraph.

---

## 11. Technical SEO & Core Web Vitals

Per-page checks (Critical unless noted):
- Status 200 (4xx/5xx = Critical); not `noindex`; canonical correct; not blocked by robots.txt.
- Compression enabled (gzip/br) — Nice-to-have if absent.
- **Core Web Vitals targets:** **LCP < 2.5s**, **INP < 200ms**, **CLS < 0.1** (field/lab). LCP > 4s
  or CLS > 0.25 = Important. Set explicit image dimensions; lazy-load below-fold; defer non-critical JS.
- Mobile-friendly / responsive (Google is mobile-first indexing).

---

## 12. Local SEO ranking factors (2026)

For local-service and multi-location businesses, enforce/advise:

1. **Google Business Profile completeness** — the strongest local lever. Complete: verified, correct
   primary category + **≥2** secondary categories, full description (≥250 chars), **≥5** photos,
   regular + special hours, a Google Post within the last 30 days, Q&A seeded, services/menu,
   attributes. (Off-site, but the auditor should flag site copy that contradicts GBP.)
2. **NAP consistency** — Name/Address/Phone **byte-identical** across the site, GBP, and every
   citation. Inconsistent NAP directly suppresses local rank. The auditor checks on-site NAP matches
   the configured NAP and the LocalBusiness schema.
3. **Citation coverage** — listed on the high-weight directories (GBP, Yelp, Facebook, Apple Maps,
   Bing Places, BBB, Angi, Nextdoor) with consistent NAP; target ≥10 quality listings.
4. **Reviews** — volume, recency, rating, and owner responses; only mark up `aggregateRating` with
   real on-page reviews.
5. **Proximity / relevance / prominence** — the local triad. Relevance is earned with
   service-specific and city/neighborhood landing pages; prominence with citations, reviews, links.
6. **Local landing pages** — a distinct, substantive page per service × service-area (not doorway
   pages); each with localized title/H1/content, `LocalBusiness` schema, embedded map, and NAP.
7. **`areaServed`** in schema and copy; embedded Google Map on contact/location pages.

---

## 13. Severity model (how the auditor grades a finding)

- **Critical** — blocks indexing or ranking, or breaks SERP/UX integrity: missing/multiple H1 (zero
  H1), missing `<title>`, `noindex` on a public page, missing/broken `sitemap.xml`, missing canonical
  causing duplicates, HTTPS/mixed-content, 4xx/5xx on a live page, missing `Organization`/`LocalBusiness`
  schema on a business site, fabricated schema (`aggregateRating` without real reviews), NAP mismatch.
- **Important** — materially suppresses performance: title/meta out of range or duplicated, missing OG
  tags, missing `BlogPosting` schema on posts, >5 images missing alt, thin/duplicate content, intent
  mismatch, keyword cannibalization, orphan pages, AI crawlers blocked unintentionally, CWV failing.
- **Nice-to-have** — incremental: `llms.txt`, `FAQPage`/`BreadcrumbList` schema, Twitter cards,
  anchor-text diversity, compression, minor microcopy.

> When every checked item passes for a page, **say so explicitly** ("Home — solid: unique 54-char
> title, single keyworded H1, valid LocalBusiness + BreadcrumbList, self-canonical"). Don't manufacture
> findings to fill a report.
