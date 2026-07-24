# Job Hunt Agent — Orchestration v2 (Fan-Out / Fan-In)

Owner: Balaji Nagendra Naga · Finalized: 2026-07-22
Foundation docs: `knowledge_base.md` · `ways.md` · `rules.md`

---

## The Design in One Sentence

The Orchestrator loads the three docs, fans out one subagent per platform — each receiving
`(knowledge_base.md + its own platform slice of ways.md + rules.md)` — and each subagent returns
only jobs scoring **≥ 70**, with a **tailored resume** and an **apply link**; the Orchestrator
fans in, dedupes, ranks, and delivers.

```
                            ┌──────────────────────────────┐
              INPUTS ─────► │        ORCHESTRATOR          │
   knowledge_base.md        │  1. load 3 docs              │
   ways.md                  │  2. slice ways.md by platform│
   rules.md                 │  3. spawn subagents (parallel)
                            └───────────────┬──────────────┘
                                            │ fan-out (parallel)
     ┌──────────┬──────────┬──────────┬─────┴────┬──────────┬───────────┐
     ▼          ▼          ▼          ▼          ▼          ▼           ▼
┌─────────┐┌─────────┐┌──────────┐┌─────────┐┌─────────┐┌─────────┐┌──────────┐
│LinkedIn ││ Naukri  ││Instahyre ││Wellfound││ Indeed  ││ Tier 2  ││ Career   │
│subagent ││subagent ││subagent  ││subagent ││subagent ││(YC, HN, ││ Portals  │
│         ││         ││          ││         ││         ││Peerlist…││ (ATS)    │
└────┬────┘└────┬────┘└────┬─────┘└────┬────┘└────┬────┘└────┬────┘└────┬─────┘
     │          │          │           │          │          │          │
     │   EACH SUBAGENT RECEIVES:  knowledge_base.md + rules.md          │
     │                          + ONLY its platform section of ways.md  │
     │   EACH SUBAGENT RETURNS:  jobs with score ≥ 70 ONLY              │
     │                          + tailored resume + apply link          │
     └──────────┴──────────┴─────┬─────┴──────────┴──────────┴──────────┘
                                 │ fan-in
                    ┌────────────▼─────────────┐
                    │   ORCHESTRATOR (merge)   │
                    │  dedupe → rank → cap 20  │
                    │  resume finalize (PDF)   │
                    │  tracker + digest        │
                    └────────────┬─────────────┘
                                 ▼
                  OUTPUTS: tracker.json · resumes/ · digest
                                 ▼
                  HUMAN: review → approve → apply
                                 ▼
                  applied_log.json (memory for next run)
```

---

## 1. Orchestrator (main agent)

**Receives:** `knowledge_base.md`, `ways.md`, `rules.md`

**Responsibilities:**
1. Parse ways.md → produce one **platform brief** per platform (that platform's row: access method, URL filters, tips — nothing about other platforms, keeps each subagent's context small and focused)
2. Spawn all platform subagents **in parallel**, each with its 3-part payload
3. Enforce a per-subagent timeout (e.g., 5 min) — a slow/dead platform never blocks the run
4. **Fan-in:** collect result JSONs, then:
   - dedupe (`lowercase(company)+lowercase(title)`; keep best route: `company_site > easy_apply > external`)
   - dedupe against `applied_log.json` (never resurface)
   - re-rank merged list by score, apply rules.md bonuses (+5 Tier 2/company_site, +5 <48h, +5 AI-tooling JD), cap at 20
   - **finalize resumes** for survivors only (see §4 — draft is made in the subagent, PDF compiled here)
5. Write outputs: tracker.json, `resumes/` folder, digest message
6. Append every surfaced job to `applied_log.json` with status `surfaced`

## 2. Platform Subagent (spec — identical logic, different platform brief)

**Receives:**
| Input | Purpose |
|---|---|
| `knowledge_base.md` | scoring reference + resume tailoring source |
| platform brief (its slice of ways.md) | where and how to search |
| `rules.md` | hard filters + keywords + scoring adjustments |

**Internal pipeline (strictly this order — cheap steps first):**
```
SEARCH   fetch listings using platform brief (keywords from rules.md, freshness filter in URL when possible)
   ▼
FILTER   apply rules.md mechanically: ≤7 days · exp min ≤1.5y · full-time only ·
         India/remote · title & stack exclusions        [deterministic, no LLM]
   ▼
ENRICH   fetch full JD text for survivors only          [don't fetch JDs you'll drop]
   ▼
SCORE    vs knowledge_base.md: stack overlap 40% · exp-band fit 25% ·
         edge match (AI tooling/Expo/a11y) 20% · practicals 15%
   ▼
GATE     score < 70 → discard. score ≥ 70 → continue    [THE quality bar]
   ▼
TAILOR   draft tailored resume content for this JD (see §4) — markdown draft, not PDF
   ▼
RETURN   result JSON (schema below)
```

**Output contract (per job — only score ≥ 70 jobs may appear):**
```json
{
  "platform": "linkedin",
  "company": "", "title": "", "location": "", "work_mode": "remote|hybrid|onsite",
  "experience_req": "", "salary": "", "posted_date": "",
  "apply_link": "",                        // direct, deep link — not a search page
  "apply_route": "easy_apply|external|company_site",
  "score": 84,
  "score_breakdown": {"stack": 36, "exp_fit": 22, "edge": 16, "practicals": 10},
  "why_fit": "one line",
  "watch_out": "one line (bond/shift/relocation catch) or null",
  "jd_summary": "3-line JD digest",
  "resume_draft": {
    "base_variant": "MERN | AI-Enhanced | Django",   // from KB §3 router
    "summary_rewrite": "JD-aligned 3-sentence summary",
    "skills_reorder": ["skill1", "skill2"],          // JD keywords first
    "project_order": ["Stylio", "Aerchain"],       // most relevant first
    "bullet_tweaks": [{"project": "", "original": "", "tailored": ""}]
  },
  "error": null
}
```
**Failure policy:** subagent errors → return `{"jobs": [], "error": "<what broke>"}`. Never crash the run.

## 3. Scoring Rubric (uniform across all subagents — critical for fair fan-in ranking)

| Component | Weight | What earns points |
|---|---|---|
| Stack overlap | 40 | React/Next/RN/Node/Express/Django REST/Mongo/SQL named in JD |
| Experience-band fit | 25 | JD wants 0–1.5 yrs → full points; unstated but junior-toned → partial |
| Edge match | 20 | AI tooling, GenAI product work, Expo, accessibility, Storybook/design systems |
| Practicals | 15 | salary ≥ 6 LPA visible, remote/any-metro, immediate-ish start OK |

**70 = the gate.** All subagents use the same rubric with the same weights, or the merged ranking is meaningless. Rubric text lives in this doc and is injected verbatim into every subagent prompt.

## 4. Resume Generation (two-stage — the correction to the original plan)

Original plan: subagent outputs a full generated resume. **Refinement:** split into draft (subagent) and finalize (orchestrator), because:
- Duplicates across platforms would waste full generations — dedupe hasn't happened yet at subagent level
- Only the capped top-20 need real files; drafts are cheap, PDFs are not

**Stage A — subagent (draft):** picks base variant via KB §3 router, rewrites summary against the JD, reorders skills/projects, tweaks bullets. Structured JSON, ~30 lines.

**Stage B — orchestrator (finalize, post-dedupe, survivors only):** applies the draft to the LaTeX template (from `Balaji_AI_Enhanced_Full_Stack_Resume_Updated.tex`), compiles PDF → `resumes/<Company>_<Role>_<date>.pdf`, links it in the tracker.

**Hard rule — truthfulness:** tailoring may reorder, rephrase, and emphasize; it may NEVER invent experience, skills, or numbers not present in knowledge_base.md. Every generated resume is reviewed by the human before any application.

## 5. Outputs (per run)

1. **tracker.json** — top-level arrays: `easy_apply` · `direct_company` · `applied_log`; each entry: score, company, role, location, mode, exp, salary, why_fit, watch_out, apply_link, resume_file, status
2. **resumes/** — one tailored PDF per surfaced job
3. **Digest:** `"7 new · 3 above 85 · top: <company — role (score)>"`

## 6. Human-in-the-Loop + Memory

- Agent **never auto-applies**. You review tracker → open apply_link → attach the generated resume → mark status `applied` in the tracker
- `applied_log.json`: every job ever surfaced/applied, with date + resume version. Read by orchestrator at fan-in (dedupe) and — via rules.md — feeds the target-company auto-grow rule
- Schedule: **daily 21:30 IST**; Sunday run additionally prunes >7-day-old rows and reports weekly stats

## 7. Notes on the Threshold (alignment fix)

rules.md previously kept Medium tier (40–69). **This design supersedes it: the gate is 70.**
If daily results feel thin (<5 jobs/run for several days), lower the gate in ONE place — rules.md —
to 60. No agent code changes.

## 8. Build Order

1. Subagent prompt template (shared skeleton + platform-brief injection) — build once, reuse 7×
2. Prove end-to-end on **Indeed** (no login): search → filter → score → draft → JSON
3. Add Wellfound + HN + ATS portals (all plain fetch)
4. Add browser-auth platforms: LinkedIn, Naukri, Instahyre
5. Orchestrator fan-in: dedupe, rank, resume finalize (LaTeX→PDF), tracker writer
6. Memory (applied_log.json) + 21:30 IST schedule + digest
