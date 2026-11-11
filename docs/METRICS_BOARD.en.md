# Metrics board design: four panels and an event schema

> Required design item. Beyond "the agent runs", the board measures internal state changes, loop counts, token cost and hallucination blocks in real time, so 3–5 domain pipelines (legal, medical, research, …) can be tuned from data (Data-Driven Harness Tuning). One event per run, keyed by `run_id`, across the Spring Boot — FastAPI — LangGraph stack.

## 1. The four panels

| Panel | Metrics | How it drives tuning |
|---|---|---|
| 1. Workflow trace | Path retrieve→grade→rewrite…; end type END / give_up / cutoff | See which questions loop in rewrite; tune retrieval and grading |
| 2. Cost control | Calls per model (SLM judge vs flagship writer), input/output tokens, cost per run (KRW) | Watch the cost target (e.g. 1.5 KRW); set the Fast/Deep Track routing ratio |
| 3. Guardrail audit | RelevanceGrade drop rate, GroundedGrade rejection reasons | Diagnose number/article errors per domain; catch over-strict graders |
| 4. Latency | Per-node ms (retrieve, abatch grading, generate), TTFT | Find the bottleneck; parallelise or swap in a lighter model |

## 2. Event schema (1 run = 1 `run` + N `node` events)

The JSON example in [METRICS_BOARD.ko.md](METRICS_BOARD.ko.md) applies as is (field names are English).

- Tokens are split into input/output; cost is converted with a price table (`pricing.json`). Rule-based nodes report shadow tokens, flagged `estimated: true`.
- `guard[].code` reuses the filter codes (`UNSUPPORTED_NUMBER`, `GAP:<metric>`, …), so the board reads any domain filter with one schema.
- Spring Boot ties request and events by `run_id`; FastAPI collects events from LangGraph node callbacks into JSONL (today: `runs.jsonl` → HTML board).

## 3. Build order
1. Collect `nodes[]` and `path` via node callbacks (extend today's `result.path/cost/log`).
2. Add end type, loop counts, call counts and cost fields.
3. Render the four panels (generalise `research_board.py` across domains).
4. Targets (cost 1.5 KRW, latency 6 s, …) in a config file; show a warning when exceeded.
