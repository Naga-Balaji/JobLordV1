# ways.md — Platform Strategy for the Job Hunt Agent
<!-- Companion to rules.md and job_agent_orchestration.md. Phase 1 search subagents read this. -->
<!-- Profile context: ~1 yr full stack (MERN/Next.js) + AI-product roles, India, remote or relocate. -->

## The Core Insight

Job platforms trade off **reach vs. competition**. Big platforms have every job but 500+ applicants per posting; niche platforms have fewer jobs but 10–30 applicants and often direct founder access. The agent should work BOTH lanes every run: Tier 1 for volume, Tier 2 for conversion.

---

## Tier 1 — High-Trust, High-Volume (the reliable 5)

| # | Platform | Trust | Footfall | Why it earns its slot | Agent access |
|---|---|---|---|---|---|
| 1 | **LinkedIn** | Very high | Very high | Largest recruiter presence in India; Easy Apply = 30-second applications; recruiters also do inbound search on your profile — the profile itself works while you sleep | Browser automation (logged in); URL filters: `f_AL=true`, `f_TPR=r604800`, `f_E=2` |
| 2 | **Naukri** | Very high | Very high | India's default hiring database — most Indian HR teams search Naukri FIRST, before posting anywhere. A fresh, keyword-rich profile gets recruiter calls without applying | Browser automation; keep profile updated every 3–4 days (recency boosts DB ranking) |
| 3 | **Instahyre** | High | Medium | Curated product-company roles; companies reach out to candidates (reverse apply); good salary transparency; strong for 1–3 yr engineers | Browser automation (logged in); complete profile = better matching |
| 4 | **Wellfound** (ex-AngelList) | High | Medium | Startup-native: salary + equity shown upfront, apply goes straight to founders/hiring managers, no recruiter middle layer | Plain HTTP fetch (public listings) — easiest platform for the agent to scrape |
| 5 | **Indeed India** | High | High | Broadest aggregator; server-rendered pages = most agent-friendly; `fromage=7` filter for freshness; catches SME/local jobs others miss | Plain HTTP fetch — the agent's proven workhorse (used for the current tracker) |

**Cut from Tier 1:** foundit (ex-Monster), Shine, TimesJobs — declining recruiter activity and high spam-posting ratios relative to the five above.

---

## Tier 2 — Low-Footfall, High-Worth (the hidden 5)

Fewer applicants per posting, higher reply rates, often direct founder contact. Check these EVERY run — one reply here is worth 50 Easy Applies.

| # | Platform | Footfall | Why it's a gem | Agent access |
|---|---|---|---|---|
| 1 | **Y Combinator — Work at a Startup** (workatastartup.com) | Very low from India | One profile → apply to hundreds of YC startups; founders read applications themselves; many US startups hire India-remote; AI startups heavily overrepresented — your AI-tooling edge lands here | HTTP fetch for listings; apply needs logged-in profile |
| 2 | **Peerlist Jobs** (peerlist.io) | Very low | Proof-of-work profile (projects > resume) — Stylio and your portfolio ARE the application; small dev-community job board, Indian-founded, very low competition | HTTP fetch; profile setup manual |
| 3 | **Hacker News — "Who is hiring?"** (monthly thread, 1st of each month) | Very low from India | Hiring managers post directly with email addresses; "remote" + "junior welcome" posts exist every month; a thoughtful email beats any ATS | HTTP fetch (hn.algolia.com API — trivially scrapeable); agent can filter by REMOTE + keywords |
| 4 | **Cutshort** | Low–medium | AI-matched Indian startup roles; fewer applicants than Naukri/LinkedIn; skill-assessment badges boost visibility for sub-2-yr candidates | HTTP fetch for public listings; browser for applying |
| 5 | **Hirect** | Low | Chat-directly-with-founder model (no ATS black hole); Indian startup focused; strong for immediate-joiner narratives | Mobile-first; manual or browser automation |

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

## Profile Prerequisites (one-time, before the agent's first full run)

- [ ] LinkedIn: headline + "Open to Work" (recruiters-only mode), skills section synced to knowledge_base.md
- [ ] Naukri: profile created/refreshed, resume uploaded, keywords from skills matrix
- [ ] Instahyre: profile completed (their matching depends on it)
- [ ] Wellfound: profile + salary/remote preferences set
- [ ] YC Work at a Startup: profile with Stylio + portfolio links
- [ ] Peerlist: proof-of-work profile (link GitHub projects)
- [ ] Cutshort: profile + 1–2 skill assessments (visibility boost)
