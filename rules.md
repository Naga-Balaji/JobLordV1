# rules.md — Job Hunt Agent (hard filters)
<!-- Machine-readable. Phase 2 applies these mechanically. Edit here, never in agent code. -->
<!-- Finalized: 2026-07-22, from Balaji's answers. Platform detail lives in ways.md. -->

## Platforms (priority order — see ways.md "Access ladder" for full detail)
<!-- Policy: try DIRECT (http fetch / public API) first; fall back to Apify only if direct fails. No browser login. -->
### Tier 1 — volume
1. LinkedIn        — direct fetch → Apify fallback `cheap_scraper/linkedin-job-scraper` (then `curious_coder/linkedin-jobs-scraper`) — Easy Apply preferred; tag easy_apply vs external
2. Naukri          — direct fetch → Apify fallback `muhammetakkurtt/naukri-job-scraper`
3. Instahyre       — no direct listing, no actor → skip (note "skipped (no access)"); optional `lenient_grove/Daily-Job-Pulse`
4. Wellfound       — direct http fetch → Apify fallback `crawlerbros/wellfound-scraper`
5. Indeed          — direct http fetch (workhorse) → Apify fallback `borderline/indeed-scraper`
### Tier 2 — low footfall (check every run)
6. YC Work at a Startup — direct http fetch → Apify fallback `parsebird/yc-jobs-scraper`
7. Peerlist Jobs        — http fetch (direct)
8. HN "Who is hiring"   — http fetch (hn.algolia.com API, direct), filter REMOTE + junior-friendly
9. Cutshort             — http fetch (direct)
10. Hirect              — no actor / mobile-only → out of scope (manual only)
### Always — zero footfall
11. Company career portals / ATS APIs (Greenhouse, Lever, Ashby) — http fetch (direct) — target list below + X-ray discovery

## Freshness
- posted_within_days: 7          # hard drop if older
- prefer_within_days: 2          # scoring bonus for very fresh postings

## Experience (candidate profile: ~1 yr professional)
- target_band: 0 to 1.5 years total (internships count toward the total)
- keep_if: stated minimum <= 1.5 yrs (so "0-1", "1 yr", "1-2 yrs", "fresher" all pass)
- drop_if: stated minimum >= 2 yrs (e.g., "2+ yrs", "2-4 yrs", "3+ yrs")
- ambiguous/unstated experience: KEEP, let Phase 3 scoring judge from responsibilities
- titles auto-drop: senior, sr., lead, principal, staff, architect, manager

## Employment type
- accept: full-time permanent ONLY
- drop: internships (paid or not), contract, freelance, part-time, training-fee/bond programs, unpaid anything

## Location logic
- work_mode == remote (India-eligible): ACCEPT always
- on-site / hybrid: ACCEPT anywhere in India (relocation OK)
- drop: roles requiring immediate on-site presence outside India, night-shift-only roles

## Role keywords (must match at least one)
- full stack developer / engineer
- MERN stack developer
- react developer / react.js developer / frontend developer (react)
- next.js developer
- react native developer (secondary priority)
- software engineer / SDE-1 (react/node context)
- AI product engineer / GenAI engineer / AI full stack (priority — leverage AI-tooling edge)

## Stack exclusions (drop if the role is PURELY these, no JS overlap)
- pure Java / Spring, pure .NET, pure PHP, pure Angular, pure Flutter/native iOS-Android

## Salary
- policy: FLAG below 6 LPA as "below target" — never drop on salary
- target band for answers/negotiation: 6-10 LPA (from knowledge_base.md)

## Company blocklist
- none (no exclusions; decide manually from the tracker)

## Target companies (career-portal agent; grow this list every run)
- Aerchain, 1Acre.in, Zoho, Freshworks, Razorpay, Postman, Hasura, Chargebee,
  Zerodha, Groww, CRED, Meesho, Swiggy, Zepto, Rapido, Darwinbox, Keka
- rule: any company appearing twice in High tier across runs → auto-add here

## Scoring adjustments (input to Phase 3)
- +5 Tier 2 platform or company_site source (low footfall)
- +5 posted within 48 hours
- +5 explicit "AI tools / GenAI / Cursor / Copilot" mention in JD (edge match)
- -5 service/consulting body-shop indicators ("client deployment", "bench")

## Data-access conduct (direct-first, Apify fallback — no browser login)
<!-- Try direct http/API first. Apify Actors (managed proxies) are the fallback. No accounts/cookies/sessions touched. -->
### Access order (per platform, every run)
- TRY DIRECT FIRST: plain http fetch of public listing pages / public APIs (ATS JSON, hn.algolia.com). Free, no per-result cost.
- FALL BACK TO APIFY only when direct is blocked (anti-bot / JS-gated), returns empty, or errors.
- Never invoke Apify for a platform whose direct fetch already succeeded.
### Hard boundaries (never, regardless of anything)
- READ-ONLY: fetch listings + JDs. Nothing else.
- NEVER submit an application, Easy Apply, message, connection request, or any form — human applies, always
- NEVER use browser login, or run an Actor that logs into / posts to / writes to any platform or the candidate's accounts
- NEVER store or pass the candidate's platform credentials to an Actor
### Run limits (keep cost + noise bounded)
- Direct fetch: reasonable page/JD caps per platform per run (~15 result pages, ~10 JD opens)
- Apify fallback: bound every Actor (maxJobs / maxRows / maxItems ~50–100 per platform); prefer pay-per-result; abort runaway runs
- One run per day per platform — never re-poll within the same day
- Use each source's own filters (date/experience/work-mode/Easy-Apply) — don't over-fetch then filter
### Failure handling (a platform failing is a digest note, never a run failure)
- Direct fetch blocked/empty -> try the Apify fallback (if one exists)
- Apify fallback errors / times out / returns empty -> return empty list + error note; continue other platforms
- No actor and no direct access (Instahyre, Hirect) -> skip platform, note "skipped (no access)" in digest
- Apify connector not authorized / no token -> use direct only; skip platforms that require the fallback, note in digest; never attempt a browser-login fallback
### Escalation
- Anything not covered here defaults to: don't do it, ask the human in the digest

## Dedupe
- key: lowercase(company) + lowercase(title)
- keep best apply_route: company_site > easy_apply > external
- check applied_log.json — never resurface an already-applied or previously-surfaced job

## Output
- max_jobs_per_run: 20
- tiers: keep High (>=70) and Medium (40-69); DROP Low (<40)
- sort: score desc
- tabs: [Easy Apply, Direct/Company, Applied Log]

## Schedule
- run: daily 21:30 IST (9:30 PM)
- digest format: "X new today - Y High tier - top pick: <company/role>" + link to updated sheet
- weekly (Sunday): prune active tabs of postings older than 7 days; report application stats
