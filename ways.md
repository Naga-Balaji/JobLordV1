# ways.md — Platform Strategy for the Job Hunt Agent
<!-- Companion to rules.md and job_agent_orchestration.md. Phase 1 search subagents read this. -->
<!-- Profile context: ~1 yr full stack (MERN/Next.js) + AI-product roles, India, remote or relocate. -->

## The Core Insight

Job platforms trade off **reach vs. competition**. Big platforms have every job but 500+ applicants per posting; niche platforms have fewer jobs but 10–30 applicants and often direct founder access. The agent should work BOTH lanes every run: Tier 1 for volume, Tier 2 for conversion.

---

## Data access — direct-first, Apify as fallback (NO browser login)

**Try the direct approach first; use the Apify connector only when direct fails.** Direct = plain HTTP fetch of public listing pages + official/public APIs (ATS JSON, hn.algolia.com). It's free, fast, and has no per-result cost. Apify Actors (managed proxies, no login) are the **fallback** for platforms that block direct fetch (bot-walls, heavy JS rendering) or when a direct fetch returns empty/errors. The old "browser automation (logged in)" flow is retired entirely.

Per-platform decision each run:
1. **Attempt direct** — HTTP fetch the public search/listing URL (or hit the public API). If it returns usable listings, use it. Done.
2. **Fallback to Apify** — only if the direct fetch is blocked, JS-gated, empty, or errors: call the platform's Actor with `mcp__Apify__call-actor`, then read the dataset with `mcp__Apify__get-dataset-items`.
3. Map fields to the tracker schema; tag `apply_route` (`easy_apply` vs `external` vs `company_site`).

### Access ladder (per platform)

| Platform | Direct approach (try first) | Apify fallback (only if direct fails) |
|---|---|---|
| Indeed | HTTP fetch (server-rendered; `fromage=7` freshness) — proven workhorse | `borderline/indeed-scraper` (`country:"IN"`, `query`, `location`, `fromDays`, `remote`, `level`, `maxRows`) |
| Wellfound | HTTP fetch of public `/jobs` listings | `crawlerbros/wellfound-scraper` (`keyword`, `location`, `remoteOnly`, `experience`, `maxItems`) |
| YC / Work at a Startup | HTTP fetch of workatastartup.com listings | `parsebird/yc-jobs-scraper` (`searchQuery`, `roleFilter`, `locationFilter`, `maxResults`) |
| HN "Who is hiring?" | hn.algolia.com API (direct) | — (direct is reliable; no fallback needed) |
| Peerlist / Cutshort | HTTP fetch of public listings | — (no dedicated Actor) |
| Career portals / ATS | Greenhouse / Lever / Ashby public JSON APIs (direct) | — (direct is reliable) |
| LinkedIn | Direct fetch usually blocked (anti-bot) — expect fallback | `cheap_scraper/linkedin-job-scraper` (`keyword[]`, `location`, `publishedAt`, `experienceLevel[]`, `workType[]`, `filterEasyApply`, `jobTitleExclude[]`); fallback-of-fallback `curious_coder/linkedin-jobs-scraper` (`urls[]` incl. `f_AL/f_TPR/f_E`) |
| Naukri | Direct fetch usually blocked (anti-bot) — expect fallback | `muhammetakkurtt/naukri-job-scraper` (`keyword`, `maxJobs`, `freshness`, `experience`, `workMode[]`, `cities[]`, `salaryRange[]`, `fetchDetails`) |

**No Apify Actor and no viable direct listing** for Instahyre (JS/login-gated) or Hirect (mobile-only) → skip in the automated run, note "skipped (no access)" in the digest; the human checks those manually. Optional multi-source catch-all if a lane runs dry: `lenient_grove/Daily-Job-Pulse` (`roles[]`, `location`, `sources[]`, `maxDaysOld`).

**Cost note:** direct fetch is free. Apify Actors are pay-per-result (~$0.0005–$0.005/job), so only invoking them on fallback keeps daily runs near-zero cost. When you do fall back, keep `maxJobs`/`maxRows`/`maxItems` bounded.

---

## Tier 1 — High-Trust, High-Volume (the reliable 5)

| # | Platform | Trust | Footfall | Why it earns its slot | Agent access |
|---|---|---|---|---|---|
| 1 | **LinkedIn** | Very high | Very high | Largest recruiter presence in India; Easy Apply = 30-second applications; recruiters also do inbound search on your profile — the profile itself works while you sleep | **Direct fetch first** (search URLs with `f_AL=true`, `f_TPR=r604800`, `f_E=2`); usually anti-bot-blocked → **Apify fallback** `cheap_scraper/linkedin-job-scraper` (fallback-of-fallback `curious_coder/linkedin-jobs-scraper`) |
| 2 | **Naukri** | Very high | Very high | India's default hiring database — most Indian HR teams search Naukri FIRST, before posting anywhere. A fresh, keyword-rich profile gets recruiter calls without applying | **Direct fetch first**; usually anti-bot-blocked → **Apify fallback** `muhammetakkurtt/naukri-job-scraper` (`fetchDetails: true` for JD enrichment) |
| 3 | **Instahyre** | High | Medium | Curated product-company roles; companies reach out to candidates (reverse apply); good salary transparency; strong for 1–3 yr engineers | **No direct listing and no Apify Actor** — skip, note in digest (optional catch-all via `lenient_grove/Daily-Job-Pulse`). Do NOT use browser login |
| 4 | **Wellfound** (ex-AngelList) | High | Medium | Startup-native: salary + equity shown upfront, apply goes straight to founders/hiring managers, no recruiter middle layer | **Direct HTTP fetch first** (public `/jobs` listings); → **Apify fallback** `crawlerbros/wellfound-scraper` |
| 5 | **Indeed India** | High | High | Broadest aggregator; server-rendered pages = most agent-friendly; `fromage=7` filter for freshness; catches SME/local jobs others miss | **Direct HTTP fetch first** — server-rendered, the agent's proven workhorse; → **Apify fallback** `borderline/indeed-scraper` (`country:"IN"`) only if blocked |

**Cut from Tier 1:** foundit (ex-Monster), Shine, TimesJobs — declining recruiter activity and high spam-posting ratios relative to the five above.

---

## Tier 2 — Low-Footfall, High-Worth (the hidden 5)

Fewer applicants per posting, higher reply rates, often direct founder contact. Check these EVERY run — one reply here is worth 50 Easy Applies.

| # | Platform | Footfall | Why it's a gem | Agent access |
|---|---|---|---|---|
| 1 | **Y Combinator — Work at a Startup** (workatastartup.com) | Very low from India | One profile → apply to hundreds of YC startups; founders read applications themselves; many US startups hire India-remote; AI startups heavily overrepresented — your AI-tooling edge lands here | **Direct HTTP fetch first** (public listings); → **Apify fallback** `parsebird/yc-jobs-scraper` (returns salary/equity/visa). Applying still needs the human's logged-in profile |
| 2 | **Peerlist Jobs** (peerlist.io) | Very low | Proof-of-work profile (projects > resume) — Stylio and your portfolio ARE the application; small dev-community job board, Indian-founded, very low competition | HTTP fetch; profile setup manual |
| 3 | **Hacker News — "Who is hiring?"** (monthly thread, 1st of each month) | Very low from India | Hiring managers post directly with email addresses; "remote" + "junior welcome" posts exist every month; a thoughtful email beats any ATS | HTTP fetch (hn.algolia.com API — trivially scrapeable); agent can filter by REMOTE + keywords |
| 4 | **Cutshort** | Low–medium | AI-matched Indian startup roles; fewer applicants than Naukri/LinkedIn; skill-assessment badges boost visibility for sub-2-yr candidates | HTTP fetch for public listings; the human applies via their own account |
| 5 | **Hirect** | Low | Chat-directly-with-founder model (no ATS black hole); Indian startup focused; strong for immediate-joiner narratives | **No Apify Actor / no public web listings** (mobile-first chat app) — out of scope for the automated run; surface manually if desired |

**Honorable mentions (rotate in if a Tier 2 source runs dry):** Weekday.works (engineer-referral based), RemoteOK + WeWorkRemotely (global remote; competition is global too), TopHire (curated but usually wants 2+ yrs — revisit next year), Turing / Crossover (remote-for-US; heavy vetting process).

---

## ⚠️ MANDATORY: Company Career Portals (the zero-footfall lane)

**Every run must also search company career pages directly.** This is where competition is lowest of all:
jobs often go live on the company's own portal **3–10 days before** they're syndicated to LinkedIn/Naukri —
applying in that window means being in the first handful of resumes.

How the agent does it:

1. **Maintain the target-company list in rules.md** (currently: Aerchain, 1Acre.in, Zoho, Freshworks, Razorpay, Postman, Hasura, Chargebee, Zerodha, Groww, CRED, Meesho, Swiggy, Zepto, Rapido, Darwinbox, Keka — expand continuously with every good company discovered in Tier 1/2 results).
2. **Hit ATS APIs directly** — most startup career pages run on an ATS with public JSON:
   - Greenhouse: `boards-api.greenhouse.io/v1/boards/<company>/jobs`
   - Lever: `api.lever.co/v0/postings/<company>`
   - Also check: Ashby (`jobs.ashbyhq.com/<company>`), Workable, SmartRecruiters, Keka/Darwinbox (common for Indian companies)
3. **X-ray search for unlisted boards** — query a search engine with:
   `site:boards.greenhouse.io "react" "india"` · `site:jobs.lever.co "full stack" "remote"` · `site:jobs.ashbyhq.com "react native"`
   This surfaces companies not on any list yet.
4. **Tag results `company_site`** — the dedupe rule already prefers `company_site` over `easy_apply` over `external`, because a direct application skips aggregator noise entirely.

---

## How the Agent Allocates Effort (per daily run)

| Lane | Share of run | Expectation |
|---|---|---|
| Tier 1 (volume) | ~50% | 15–25 fresh matches/day; feeds the tracker |
| Tier 2 (conversion) | ~30% | 2–5 matches/day; highest reply probability |
| Career portals (zero-footfall) | ~20% | 0–3 matches/day; best odds per application |

Ranking note for Phase 3 scoring: add a small bonus (+5) to Tier 2 and `company_site` results — lower footfall justifies surfacing them above an equivalent Tier 1 match.

## Prerequisites

**For the agent (scraping):** just the **Apify connector** — no platform logins. Actors run without the candidate's accounts.

**For the human (applying — profiles still matter for inbound + one-click apply):**

- [ ] LinkedIn: headline + "Open to Work" (recruiters-only mode), skills section synced to knowledge_base.md
- [ ] Naukri: profile created/refreshed, resume uploaded, keywords from skills matrix
- [ ] Instahyre: profile completed (their matching depends on it)
- [ ] Wellfound: profile + salary/remote preferences set
- [ ] YC Work at a Startup: profile with Stylio + portfolio links
- [ ] Peerlist: proof-of-work profile (link GitHub projects)
- [ ] Cutshort: profile + 1–2 skill assessments (visibility boost)
