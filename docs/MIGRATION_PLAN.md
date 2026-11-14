# Migration plan (2026-11-11)

1. **Tonight (NightCoder rounds 5-7, source repo):** clean module boundaries: public API facade (R19), node plug-in points (R20),
   domain-agnostic eval runner and board (R21, R31), domain packs (R18, R22, R23, R30, R32), embedding guide (R26),
   extraction plan with the import graph (R34).
2. **2026-11-12 morning (Claude):** move the research graph, domain filters, guardrail, State types, eval runner/board, eval
   cases, tests and design docs here **with commit history**; package as `langgraph_research_kit`; this repo alone must
   install, test and build the board.
3. **Then:** `langchain-rag-research-agent` depends on this kit (pinned commit) and keeps its old public names as
   re-exports, so existing code and tests keep working.

## Update 2026-11-14

Steps 2 and 3 are cancelled for this repository. The implementation (research graph, domain filters, eval runner and
board, model nodes) moves to a private repository instead. This public repository keeps the concept, the board design
and the first synthetic measurements as they were published.
