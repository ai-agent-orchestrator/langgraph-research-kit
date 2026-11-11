# LangGraph filter design brainstorm

2026-11-11 · Authors: owner + Claude · Status: brainstorm draft · [한국어](FILTER_BRAINSTORM.ko.md)

## 1. Purpose and key questions

One goal: **show in numbers whether a filter plugged in for a given use actually blocks the risks, what it costs (over-blocking, tokens), and use those numbers to keep improving the filter.** The graph is shared; what makes a product different is the filter and its validation record.

Questions to answer:

1. **Does it catch?** What share of each domain's known hallucination types (non-existent articles, definitive diagnoses, numbers without a source) does the filter catch?
2. **Does it over-block?** How often does it wrongly block or force a rewrite of a correct answer? A filter that only blocks is useless.
3. **What does it cost?** How many tokens and seconds do the filter and its loops add, and does retrieval still save more than that?
4. **Is it improving?** As rule versions change, how do 1–3 move on the same cases?
5. **Can we trust it?** Does the conclusion hold with sample sizes and intervals in view, and does a config chosen on the compare set hold on the confirm set?

Four domains: **research, legal, medical, everyday life** (local administration, everyday law, consumer information). Two kinds of output: material I use to tune and decide, and material that shows outsiders why to trust it.

## 2. Design principles

Six fixed points; everything else is decided by experiment.

1. **Never let the model answer straight away.** Every request goes route → local retrieval → rule filter. That pipeline is the moat.
2. **The graph is fixed, the filter is swapped.** Nodes and edges stay; only `check_input` / `verify` / `finalize` and a few extra State fields change per domain.
3. **Control lives in edges and State.** Pure-Python route functions decide loops, cost and errors. The model is never asked whether to stop.
4. **Safety metrics are never averaged into quality.** 97% average with one definitive diagnosis fails a medical filter. Safety goes on top of the board, separately, with a Wilson upper bound.
5. **Unverified text is never returned.** Fixable → retry within budget; otherwise a partial report plus a "not verified" list. Nothing is invented.
6. **Claims are measured.** "Better than a single LLM" is a hypothesis, measured on synthetic cases; real-model results and rule-path results never share a table.

## 3. Idea pool

Ideas so far, refined and grouped. Status: ✅ implemented (rule-based) · 🔄 in tonight's queue · 💡 idea.

| Area | Idea | Link to filter validation | Status |
| --- | --- | --- | --- |
| Structure | Outer rail: classify → rag → cost/loop gates; RAG is one isolated node | creates the slots (screen, verify) where filters plug in | ✅ |
| Structure | Division of labour: refine → extract → split pro/con/limits → compare → write → verify | shows at which step something went wrong | ✅ |
| State | Reducers accumulate claims, metrics, relations; every piece has source id + character span | gives the verifier something to check against | ✅ |
| State | Domain fields: legal timeline log, medical contraindication list, everyday document checklist | material for domain verify rules | 🔄 legal, medical skeletons |
| Loops | Targeted loops: missing metric → re-search that metric only; prose issue → rewrite only | verify codes are return addresses | ✅ |
| Loops | Stop when no progress, per-code priority, per-node token caps | keeps filters from causing cost blow-ups | 🔄 R25 |
| Cost | Shadow-token accounting, State-compressed prompts, corpus-size break-even | puts the filter's cost in numbers | ✅ |
| LangGraph | Parallel branches (Send), per-document map-reduce extraction | more evidence in the same time | 🔄 R6, R28 |
| LangGraph | Checkpoints + interrupt (resume after human review) | human approval gate for high-risk domains | 🔄 R13 |
| LangGraph | Per-domain subgraphs | swap a whole filter in one piece | 💡 |
| Models | Structured output for plan/extract/write, local models, rule fallback | rule vs model filters to compare | 🔄 R10, R11, R27 |
| Models | LLM judge beside the rule verifier; only disagreements go to a human | radar for the rules' misses and over-blocks | 💡 |
| Models | Self-consistency: generate 3 times, keep only shared claims | filters uncertain claims before the filter | 💡 |
| Models | Abstain ("I don't know") as an explicit path | reduces forced answers by design instead of blocking them | 💡 |

## 4. Domain filter designs

All four plug into the same graph; only these cells change. Research runs rule v1; legal and medical get skeletons tonight; everyday life is in design.

| Item | Research | Legal | Medical | Everyday life |
| --- | --- | --- | --- | --- |
| Main hallucination | lost causality, generic text, numbers without a source | non-existent or repealed articles, mixed-up timelines | definitive diagnosis, missing contraindications, doses | regional exceptions ignored, outdated forms and rules |
| check_input | scope, length | case-type branch (criminal, civil, family) | **emergency triage → stop and direct to care** | jurisdiction and application-type branch |
| Extra State fields | claims, relations, metrics | chronological event log (reducer) | symptoms, current medication, contraindications | document checklist, effective dates |
| Forced retrieval | local documents, pro/con/limits split | verified statute DB only | clinical guideline evidence | latest-rule DB only |
| verify codes | UNSUPPORTED_NUMBER, CAUSAL_LOSS, CONSTRAINT_DROPPED, ONE_SIDED, LOW_DENSITY, GAP | UNSUPPORTED_STATUTE, REPEALED_CITED, TIMELINE_MIXED | DEFINITIVE_DIAGNOSIS, CONTRAINDICATION_DROPPED, UNSUPPORTED_DOSE | OUTDATED_FORM, LOCAL_EXCEPTION_DROPPED, CHECKLIST_INCOMPLETE |
| Not retryable (safety) | NO_EVIDENCE | repealed article cited | definitive diagnosis, emergency | outdated form |
| finalize | unchanged | "not legal advice" notice | disclaimer + see a specialist | check with the agency |
| Human approval (interrupt) | optional (metric plan) | recommended (case type) | **required** | optional |
| Status | ✅ rule v1 | 🔄 skeleton (R18, R22) | 🔄 skeleton (R23, R32) | 🔄 skeleton (R30) |

**Everyday life** widens "local business and administration" to everyday law and consumer information. What they share: where and as of when decides the answer.

## 5. How to validate a filter

Judge each filter by four numbers: **does it catch (detection), does it over-block (false-block rate), does it keep what matters (retention), what does it cost**. Never one blended score.

| Metric | Measures | How | Example target |
| --- | --- | --- | --- |
| Seeded-fault detection | share of known hallucinations the filter catches | inject faults into gold reports (change a number, drop a condition, fake an article, definitive wording) and run verify; count per fault type | safety faults ≥ 99% (at the Wilson lower bound) |
| False-block rate | share of correct answers blocked or sent to rewrite | run fault-free gold reports, including paraphrases and synonyms | ≤ 5% |
| Retention | share of must-keep items (metrics, causality, conditions, counter-evidence) in the final report | today's board quality metric | report with intervals |
| Cost | tokens and time added by filters and loops | filter on/off; corpus-size break-even | multiple of baseline |

**Three case sets** (per domain)

1. **Gold set:** synthetic documents + answers (must-have and must-not items). Split compare / confirm; tune on compare only.
2. **Seeded-fault set:** fault variants generated automatically from gold answers. Hundreds of cases without hand-writing; rerun on every rule change.
3. **Adversarial set:** injection attacks, ambiguous questions, edge cases (symptoms that may or may not be an emergency, articles amended last week).

**Judging**

- Run the rule filter and an LLM judge side by side; **only their disagreements** go to a human. Human judgement is the scarcest resource.
- Every rate carries its sample size and Wilson 95% interval. Zero is written "upper bound x%", not "none".
- **Ablation** (filter off vs on) is a default view. A filter's value only shows against its absence.

## 6. Feedback loop

One daily cycle takes a problem seen on the board back into the next rule version and a regression test.

1. **Run:** overnight, run each domain's gold, seeded-fault and adversarial sets; append results (JSONL).
2. **Look:** on the board, safety → false blocks → retention → cost, in that order. Cases where the rules and the LLM judge disagree collect in a review queue.
3. **Judge:** for each case I pick one of three: filter right / filter missed / filter over-blocked, with a one-line reason.
4. **Change rules:** judgements become Codex specs; a rule-file change bumps `version`.
5. **Pin:** every judged case becomes a regression test; repeating the mistake breaks the test.
6. **Compare:** the run history shows how 1–4 moved against the previous version; a version that worsens any of them does not become the default.

The aim is to judge on the board itself. Sharing judgements with others would use the published page's shared storage.

## 7. Experiment material

Two kinds of material from the same run history. Internal material is detailed because it drives decisions; external material leads with limits because it earns trust.

| Material | For | Contents | Form |
| --- | --- | --- | --- |
| Tuning board | me | detection, false blocks, retention, cost per config and knob; review queue | local HTML board, updated daily |
| Weekly experiment note | me | rules changed, numbers moved, ideas dropped and why | one page |
| Fault catalogue | me + outside | hallucination types per domain with synthetic examples; whether the filter catches them | table + examples |
| Filter scorecard | outside | the four numbers per domain + sample sizes, intervals, version, limits | one-page PDF (dark) |
| Demo board | outside | baseline vs graph side by side on synthetic cases, knob sliders | public link page |
| Three case stories | outside | research, legal, medical: question → what a single call got wrong → where the filter caught it → final report | slides or doc |
| Integration guide | outside (developers) | plugging the kit into another project, swapping filters, cost caps | README + examples |

Rules for external material: synthetic data only, real-model vs rule-path results labelled, intervals instead of guarantees.

All PDFs default to dark mode.

## 8. Priorities and next experiments

Build the measuring tools first, add filters second: without a way to measure, you cannot tell whether a filter improved.

- [ ] **Fault injector:** generates fault variants (numbers, conditions, causality, citations, definitive wording) from gold answers; shared by all domains.
- [ ] **Four-number board:** detection, false blocks, retention, cost per domain, safety on top (extends R21, R31).
- [ ] **Filter on/off ablation** as a default board view.
- [ ] **Review queue + three-button judging** (right, missed, over-blocked), judgements turned into regression tests automatically.
- [ ] **LLM judge in parallel** (local model, or an allowed model on masked text) to collect disagreements with the rules.
- [ ] **Legal and medical gold sets of 30 each** plus adversarial sets; everyday-life skeleton with 10 cases.
- [ ] **Real-model baseline:** local Llama one-shot summary vs the graph on the same cases.
- [ ] **Filter scorecard PDF (dark)** and a first public demo board link.

Suggested order: fault injector → four-number board → legal/medical gold sets → judging loop → LLM judge → real-model baseline → external material.

## 9. Open questions

- Everyday-life scope: start with local administration, everyday law or consumer information?
- Different safety targets per domain (e.g. detection ≥ 99%, false blocks ≤ 5%)? Should medical tolerate more over-blocking?
- LLM judge: local models only, or outside models on masked text for evaluation only?
- Judging alone, or one domain expert (lawyer, physician) reviewing each?
- Going public: research scorecard first, or wait for legal and medical?
- Selling filters: licence per domain pack, or a service that includes the tuning board?

## 10. Metrics board: four panels and an event schema

To tune 3–5 domain pipelines from data (Data-Driven Harness Tuning), the board must show internal state, loop counts, token cost and hallucination blocks live. The unit is one `run_id` event across the Spring Boot — FastAPI — LangGraph stack.

| Panel | Measured | Used for |
|---|---|---|
| 1. Workflow trace | Path retrieve→grade→rewrite…; end type END / give_up / cutoff | See which questions loop in rewrite; tune retrieval and grading |
| 2. Cost control | Calls per model (SLM judge vs flagship writer), input/output tokens, cost per run (KRW) | Watch the cost target (e.g. 1.5 KRW); set the Fast/Deep Track routing ratio |
| 3. Guardrail audit | RelevanceGrade drop rate, GroundedGrade rejection reasons | Diagnose number/article errors per domain; catch over-strict graders |
| 4. Latency | Per-node ms (retrieve, abatch grading, generate), TTFT | Find the bottleneck; parallelise or swap in a lighter model |

Full schema: `docs/METRICS_BOARD.en.md`
