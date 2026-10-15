# Agentic CAE *(working title)*

**Status: objective frozen (14 Oct 2026) · no code yet.** This repository documents the project from the idea stage onwards.

A ReAct-style engineering agent that turns a part plus a load case described in plain language into a **verified** FEA verdict, and then proposes a lighter design that still meets the required safety factor. The language model plans, asks and explains; deterministic, tested tools do the geometry, meshing, solving and checking.

## Why

- Routine "will this part hold?" checks wait in an analyst's queue, and most analyst time goes into setup rather than physics.
- Parts are over-designed because exploring lighter variants is slow.
- LLM agents can already drive simulation tools, but *running* is not *right*: on FEABench, 88% of the best agent's solver calls were executable, yet it fully solved no problem. This project puts verification first.

## Objective (frozen)

| | Objective | Headline targets |
|---|---|---|
| **O1** | Verified verdicts, fast | ≤ 10 min per load case · within ±10% (displacement) of SimJEB reference FEA · **0 silent wrong answers** |
| **O2** | Lightweighting with safety margin | ≥ 10% mass saved at equal safety factor · 0 constraint violations · ₹ value reported with stated assumptions |
| O3 | *Planned extension:* patterned power-electronics substrates — copper-pattern drawing → 3-D model → thermo-mechanical analysis → manufacturability risk report | not frozen |

Full detail: **[docs/01_problem_statement.md](docs/01_problem_statement.md)**

## How it will work

```
plain-language request
      │
      ▼
 ReAct loop (LLM):  Thought → Action (call a tool) → Observation → …
      │
      ▼
 verified tools:  spec → geometry → mesh → loads/BCs → solve
                  → verify (equilibrium, convergence, analytical bounds)
                  → modify design / optimise → report
```

The LLM never writes solver physics, and every number in a report comes from a tool.

## Roadmap

| Version | Adds | Status |
|---|---|---|
| v0 | Verified verdict: analytical benchmarks, SimJEB reproduction, ReAct agent, report | next |
| v1 | Lightweighting loop on parametric brackets | planned |
| v2 | Surrogate model with an out-of-distribution confidence gate inside the loop | planned |
| v3 | Patterned-substrate extension; neural Newton warm start for nonlinear copper plasticity | planned |

## Documentation

- [Pre-literature and technology review](docs/00_pre_literature_review.md) — the landscape, the evidence, and the gaps
- [Problem statement](docs/01_problem_statement.md) — objectives, metrics, data, roadmap, risks
- [DEVLOG](DEVLOG.md) — every decision and result, including the failures

## Data

SimJEB, DeepJEB, self-generated parametric brackets and analytical benchmarks. No data is committed; see [data/README.md](data/README.md).

## Planned stack

Python · LangGraph · Pydantic · CadQuery · gmsh · FEniCSx (dolfinx) · CalculiX · PyVista · scipy/Optuna — on Windows with WSL2 and an RTX 4060 (8 GB).

## Author

**Varad Pawar** — M.Tech, Mechanical Engineering (Manufacturing), IIT Bombay · [LinkedIn](https://www.linkedin.com/in/varadkpawar) · [GitHub](https://github.com/VdKPr)

*Not affiliated with Ansys or Synopsys.*
