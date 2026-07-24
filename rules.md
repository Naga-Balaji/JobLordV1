# rules.md — Job Hunt Agent (hard filters)
<!-- Machine-readable. Phase 2 applies these mechanically. Edit here, never in agent code. -->
<!-- Finalized: 2026-07-22, from Balaji's answers. Platform detail lives in ways.md. -->

## Platforms (priority order — see ways.md for access methods)
### Tier 1 — volume
1. LinkedIn        — browser (logged in) — Easy Apply preferred; tag easy_apply vs external
2. Naukri          — browser (logged in) — also keep profile fresh (recruiter DB ranking)
3. Instahyre       — browser (logged in)
4. Wellfound       — http fetch
5. Indeed          — http fetch — reliable fallback
### Tier 2 — low footfall (check every run)
6. YC Work at a Startup — http fetch
7. Peerlist Jobs        — http fetch
8. HN "Who is hiring"   — http fetch (hn.algolia.com API), filter REMOTE + junior-friendly
9. Cutshort             — http fetch
10. Hirect              — browser/manual
### Always — zero footfall
11. Company career portals / ATS APIs (Greenhouse, Lever, Ashby) — target list below + X-ray discovery

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

## Browser-platform conduct (LinkedIn / Naukri / Instahyre / Hirect — cautious agent rules)
<!-- These platforms run through the user's REAL logged-in browser session. Protect the account above all. -->
### Hard boundaries (never, regardless of anything)
- READ-ONLY by default: search listings, open postings, read JDs. Nothing else.
- NEVER submit an application, Easy Apply, or any form — human applies, always
- NEVER send messages, connection requests, InMails, or founder chats
- NEVER change account settings, visibility, preferences, or notification options
- NEVER edit profile content (exception below), upload files, or delete anything
- NEVER interact with posts: no likes, comments, follows, endorsements
### Pacing (look human, stay under radar)
- Max 1 browser platform per run session; 3-6 seconds between page loads
- Max ~15 search-result pages and ~10 job-detail opens per platform per run
- One run per day per platform — never re-poll within the same day
- Use the platform's own URL filters (date/experience/Easy-Apply) instead of clicking through UI repeatedly
### Abort conditions (stop immediately, log, report in digest — do NOT retry)
- CAPTCHA or verification challenge appears -> abort platform, flag "manual login check needed"
- Any security warning, unusual-activity notice, or forced re-login -> abort ALL browser platforms this run
- Logged-out state -> skip platform, note in digest; never attempt to log in or handle credentials
### Single exception (explicit approval each time)
- Naukri profile "refresh" (recency boost) is allowed ONLY when the user approves it in that session; agent proposes, human confirms
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
