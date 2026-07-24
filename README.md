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
