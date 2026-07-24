# JobLord

A nightly, AI-driven **job-hunt orchestrator**. JobLord fans out one subagent per job platform, each subagent searches → filters → scores listings against a single candidate profile, and only jobs scoring **≥ 70** come back — each with a tailored resume draft and a direct apply link. The orchestrator then dedupes, ranks, caps the list at 20, finalizes resumes, and writes a tracker plus a digest for the human to review.

**JobLord never auto-applies.** It prepares everything; the human reviews the tracker and applies.

This repo is a **specification / prompt project**, not runnable code. It defines the agent's behavior, filters, and candidate knowledge as a set of Markdown documents that are injected into an LLM agent (built for [Claude Code](https://claude.com/claude-code) / Cowork), plus a standalone HTML dashboard for tracking results.

---

## Architecture — Fan-Out / Fan-In

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

### Each platform subagent runs the same pipeline (cheap steps first)

```
SEARCH → FILTER → ENRICH → SCORE → GATE (≥70) → TAILOR → RETURN
```

- **SEARCH** — fetch listings using the platform brief (keywords + freshness filters).
- **FILTER** — apply `rules.md` mechanically (≤7 days, exp min ≤1.5y, full-time only, India/remote, title & stack exclusions). Deterministic, no LLM.
- **ENRICH** — fetch full JD text for survivors only.
- **SCORE** — against the candidate profile using the shared rubric.
- **GATE** — score < 70 is discarded silently.
- **TAILOR** — draft tailored resume content (JSON) for this JD.
- **RETURN** — result JSON, or `{"jobs": [], "error": "..."}` on failure. A platform failing is a digest note, never a run failure.

### Scoring rubric (identical across all subagents — required for fair ranking)

| Component | Weight | What earns points |
|---|---|---|
| Stack overlap | 40 | React / Next / RN / Node / Express / Django REST / Mongo / SQL named in JD |
| Experience-band fit | 25 | JD wants 0–1.5 yrs → full; unstated but junior-toned → partial |
| Edge match | 20 | AI tooling, GenAI product work, Expo, accessibility, design systems |
| Practicals | 15 | salary ≥ 6 LPA, remote / any-India, reasonable joining |

**70 is the gate**, tunable in one place (`rules.md`) if daily results run thin.

---

## Repository contents

| File | Role |
|---|---|
| [JobLord_agent.md](JobLord_agent.md) | The agent definition — drop into `.claude/agents/JobLord.md` or use as the scheduled-task prompt. Contains the run procedure, output schema, and boundaries. |
| [job_agent_orchestration.md](job_agent_orchestration.md) | Full orchestration design (v2): fan-out/fan-in, subagent contract, resume two-stage generation, build order. |
| [knowledge_base.md](knowledge_base.md) | **The candidate** — identity, positioning, resume-variant router, scoring reference, and resume-tailoring source. |
| [ways.md](ways.md) | **Platforms + access methods** — sliced per-platform into subagent briefs. |
| [rules.md](rules.md) | **Hard filters** — freshness, experience band, employment type, location logic, role keywords, stack exclusions, scoring adjustments, browser-platform conduct, and schedule. |
| [Job Hunt Tracker (standalone).html](Job%20Hunt%20Tracker%20(standalone).html) | Standalone dashboard that reads/writes `tracker.json` (tabs: Easy Apply · Direct/Company · Applied Log). |
| [Balaji_AI_Enhanced_Full_Stack_Resume_Updated.tex](Balaji_AI_Enhanced_Full_Stack_Resume_Updated.tex) | LaTeX resume template used to compile tailored PDFs. |

### The "three-doc brain"

The agent never hardcodes candidate or platform facts. It loads three documents at runtime:

1. `knowledge_base.md` — who the candidate is
2. `ways.md` — where and how to search
3. `rules.md` — how to filter, score, and behave

Edit these Markdown files to change behavior — never the agent logic.

---

## Outputs (per run)

```
job-hunt/
├── tracker.json          # cumulative — easy_apply · direct_company · applied_log
├── applied_log.json      # memory: every job ever surfaced/applied (dedupe source)
└── runs/<YYYY-MM-DD>/
    ├── digest.md         # "X new · Y ≥85 · top: <company — role (score)>" + skips + errors
    └── resumes/<Company>_<Role>_<Score>.pdf
```

Each `tracker.json` job entry uses a fixed schema (`key`, `company`, `role`, `platform`, `score`, `location`, `work_mode`, `experience_req`, `salary`, `posted_date`, `why_fit`, `watch_out`, `apply_link`, `apply_route`, `resume_file`, `status`, `surfaced_on`, `notes`) — the dashboard reads these keys verbatim. See [JobLord_agent.md](JobLord_agent.md) for the full table.

---

## Usage

1. Register the agent: copy [JobLord_agent.md](JobLord_agent.md) into `.claude/agents/JobLord.md` (Claude Code) or paste it as a scheduled-task prompt (Cowork).
2. Keep the three-doc brain current — especially `knowledge_base.md` (your profile) and `rules.md` (target companies, gate threshold).
3. Run on schedule (default **daily 21:30 IST**; Sunday additionally prunes rows > 7 days old and reports weekly stats).
4. Open `Job Hunt Tracker (standalone).html`, review surfaced jobs, open each `apply_link`, attach the generated resume, and mark `status: applied`.

---

## Boundaries (non-negotiable)

- **Never auto-submit** an application, Easy Apply, message, or connection request — the human applies, always.
- **Never invent resume content** not present in `knowledge_base.md`; tailoring may reorder and rephrase, never fabricate.
- Browser platforms (LinkedIn / Naukri / Instahyre / Hirect) are **read-only**, human-paced, and hard-capped per run; abort on any CAPTCHA, security warning, or forced re-login. See the "Browser-platform conduct" section of [rules.md](rules.md).
- A platform failing is a digest note, not a run failure.

---

**Owner:** Balaji Nagendra Naga · Orchestration finalized 2026-07-22
