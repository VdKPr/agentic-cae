# Problem Statement

| | |
|---|---|
| **Status** | **Frozen v1.0 — 14 Oct 2026** |
| **Frozen scope** | O1 (verified verdicts) + O2 (lightweighting), on structural brackets |
| **Planned extension** | O3 — patterned power-electronics substrates (not frozen; scoped separately) |
| **Author** | Varad Pawar |
| **Related** | [Pre-literature and technology review](00_pre_literature_review.md) · [DEVLOG](../DEVLOG.md) |

---

## 1. The business problem

Routine structural design work loses money in two places.

1. **Time.** A design engineer who needs a "will this part hold?" answer usually waits for an analyst, and most of the analyst's effort goes into setup — cleaning geometry, assigning materials, choosing boundary conditions, meshing — not into the physics.
2. **Material.** Because each check is slow, few lighter variants get explored, so parts ship heavier than they need to be.

LLM agents can now drive simulation tools, but running is not the same as being right. On FEABench, the best agent produced executable solver calls 88% of the time and still solved no problem completely and correctly. An agent a company could trust has to put verification first.

## 2. Problem statement

> Build a ReAct-style engineering agent that reduces the time and material cost of structural design checks. Given a part (STEP or mesh) and a load case in plain language, it returns a verified stress and deflection verdict, then iteratively proposes a lighter design that still meets the required safety factor. The language model plans, asks clarifying questions and explains; deterministic, verified tools do the geometry, meshing, solving and checking.

## 3. Objectives and success criteria

### O1 — Faster verified verdicts

| Metric | Definition | Target |
|---|---|---|
| Time-to-verdict | Wall-clock time from request to verified report, on a laptop | ≤ 10 min per load case at SimJEB mesh size |
| Displacement agreement | Same SimJEB mesh and element order as the reference: maximum displacement magnitude, and relative L2 error of the displacement field | within ±10% |
| Stress agreement | von Mises, 99th percentile, excluding a zone around bolt and load interfaces | within ±15% (stress peaks depend on mesh and load application) |
| Analytical benchmarks | Cantilever, plate with a hole (Kirsch), thick cylinder (Lamé), 1-D conduction | ≤ 2% / 3% / 1% / 0.5% |
| **Silent wrong answers** | The run completes and reports a verdict, but the error is outside tolerance | **0 — headline metric** |
| Baseline | The same tasks done manually in Ansys Student, timed | measured and reported side by side |

### O2 — Lightweighting without losing safety margin

| Metric | Definition | Target |
|---|---|---|
| Mass reduction | Mass saved at an equal or better minimum safety factor across all load cases, verified by full FEA | ≥ 10% on parametric bracket families (revisit after v0) |
| Constraint violations | Final proposed designs that fail full-FEA verification | 0 |
| Solver efficiency | Full FEA solves per optimisation | reported; reduced in v2 |
| Business value | ₹ saved = mass saved × material price (public source, dated) × assumed annual volume | reported, with every assumption written down |

### O3 — Planned extension (not frozen): patterned substrates

A copper-pattern drawing (DXF, or DWG via conversion) is turned into a layered metal–ceramic 3-D model, analysed thermo-mechanically, and summarised as a design-for-manufacturability (DFM) risk report: warpage, ceramic stress, copper imbalance and design-rule checks. It reuses the same agent core with different tools, and uses only synthetic patterns and public data. It will be scoped and frozen separately.

## 4. Scope

**In scope (v0–v2):** linear-elastic static analysis; steady conduction for the analytical checks; single parts (no assemblies); isotropic materials from a curated table; text input (voice optional later); an open-source Python stack.

**Out of scope:** certification or engineering sign-off; contact, large deformation, fatigue and creep (v0–v2); CFD; assemblies; any commercial use of SimJEB CAD; any proprietary or employer data.

## 5. Approach — a ReAct agent with verified tools

```
User: part + load case (text; voice later)
   │
   ▼
 ┌──────────────── ReAct loop (LLM) ─────────────────┐
 │ Thought:     what is being asked? what is missing? │
 │ Action:      call one tool, or ask the user        │
 │ Observation: numbers, logs and check results       │
 │ … repeat until a verified answer or a step limit   │
 └────────────────────────────────────────────────────┘
   │ tools are deterministic, tested Python functions
   ▼
 parse_spec → load_geometry → mesh → apply_loads_bcs → solve
   → verify (equilibrium, mesh convergence, analytical bounds)
   → modify_design / run_optimizer (O2) → report
```

**Guardrails**

1. The LLM never writes solver physics (weak forms, boundary-condition code). It chooses tools and fills a validated JSON spec.
2. Every number in a report comes from a tool, never from the LLM.
3. A verification gate must pass before any verdict is shown; failures go to repair or back to the user.
4. Step and retry budgets belong to the orchestrator, not the LLM.
5. Every Thought / Action / Observation trace is logged — an audit trail and an evaluation dataset.

**Planned stack:** Python · LangGraph (ReAct loop) · Pydantic (spec schema) · pint (units) · CadQuery (geometry) · gmsh (meshing) · FEniCSx/dolfinx (solver, run in WSL2 on Windows) · CalculiX (cross-check) · PyVista (plots) · scipy/Optuna (optimiser) · an API LLM first, a local model later.

## 6. Data and validation

| Source | What it provides | Use here | Licence / caveats |
|---|---|---|---|
| **SimJEB** (Whalen, Beyene & Mueller, *Computer Graphics Forum* 2021) | 381 simulated jet-engine bracket designs. Per design: clean CAD (.stp), OptiStruct deck (.fem), tetrahedral mesh (.vtk), surface mesh (.obj), and nodal results (.csv): displacement X/Y/Z, magnitude and von Mises for four load cases (vertical, horizontal, diagonal, torsional). Linear-static OptiStruct runs on first-order tetrahedra (~2 mm average size). Ti-6Al-4V, E = 113.8 GPa, ν = 0.342. Three 80/20 splits. | O1 reference: reproduce the linear-static results on the provided meshes. Realistic geometry for O2 | Data under ODC-By; CAD from the GrabCAD challenge is non-commercial. The authors note distorted elements in some meshes and limited stress accuracy from first-order elements and sharp corners |
| **DeepJEB** (Hong et al., 2024) | 2,138 bracket designs with structural simulation labels, second-order tetrahedra and field data | O2 trends; surrogate training in v2 | Check the licence before use |
| **Self-generated parametric brackets** | CadQuery families with design variables (thicknesses, ribs, hole patterns), solved by our own verified pipeline | The O2 optimisation loop; unlimited labels | Ours |
| **Analytical benchmarks** | Closed-form solutions | Verification gate and regression tests | — |
| **Manual baseline** | The same tasks timed in Ansys Student | The "before" number for O1 | Ansys Student is for educational use only |

**How the SimJEB reference will be reproduced.** In SimJEB, each bolt is a rigid RBE2 spider with a constraint at its centre, and each load is applied through an RBE3 spider over the interface-hole surfaces. Plan: read the exact node sets and load values from each `.fem` deck; clamp the bolt-hole surfaces; apply equivalent distributed tractions for forces and an equivalent traction distribution for the torque. This differs from RBE3 near the interfaces, which is why stress is compared outside an exclusion zone. Reproduction uses first-order elements to match the reference; a second-order run then shows how far the first-order reference is from a converged answer — itself a useful finding.

## 7. Roadmap and exit criteria

| Version | Adds | Exit criterion |
|---|---|---|
| **v0 — Verified verdict (O1)** | JSON spec; four analytical templates; SimJEB reproduction; ReAct loop with tools; report; manual-baseline timing; evaluation set v0 | Benchmarks pass; ≥ 20 SimJEB brackets × 4 load cases within O1 tolerances; 0 silent wrong answers |
| **v1 — Lightweighting loop (O2)** | Parametric bracket families; `modify_design` and `run_optimizer` tools; safety-factor constraint across all load cases; ₹ value report | ≥ 10% mass saved at equal safety factor on ≥ 3 families; 0 constraint violations |
| **v2 — Surrogate tool** | Surrogate trained on DeepJEB, SimJEB and self-generated data; out-of-distribution confidence gate; used inside the v1 loop | Fewer full FEA solves per optimisation, with 0 violations after the final full-FEA check |
| **v3 — Substrate extension (O3)** | Pattern drawing → layered model → thermo-mechanical analysis → DFM risk report; nonlinear copper plasticity accelerated by a residual-aware neural Newton warm start (from the author's M.Tech thesis) | Defined when O3 is frozen |

Voice input (faster-whisper) is optional after v0. Expected pace: part-time, roughly 15 hours a week.

## 8. Risks

| Risk | Mitigation |
|---|---|
| The LLM sets wrong boundary conditions or units | Pydantic schema, unit normalisation, named faces, clarifying questions, reaction-force equilibrium check |
| Distorted SimJEB elements; boundary-condition translation differs from OptiStruct | Mesh-quality filter; compare outside interface zones; document every mismatch |
| FEniCSx on Windows | Run in WSL2 (conda-forge) or Docker |
| Laptop limits (8 GB GPU; CPU solver) | Iterative solvers (CG with algebraic multigrid); first-order elements for reproduction |
| The optimiser exploits model gaps (thin walls, singular stress points) | Manufacturing bounds on design variables; stress evaluated away from singular corners; final full-FEA check |
| Over-claiming | Every result tied to a logged run; failures recorded in the DEVLOG |

## 9. Non-goals

- Not a replacement for commercial solvers or for an analyst's sign-off.
- Not affiliated with Ansys or Synopsys; Ansys appears only as the manual baseline tool.
- No proprietary data from any company in this repository, now or later.

## 10. Open questions (to resolve before v0 code)

1. The safety-factor definition for O2, and a cited public yield strength for Ti-6Al-4V (additively manufactured and wrought values differ).
2. Which SimJEB subset to reproduce first (filtered by mesh quality).
3. The LLM for development (API) and the local fallback.
4. Evaluation-set size and the paraphrase protocol.
5. Repository licence (decide before the first code commit).

## References

- Whalen, Beyene & Mueller. *SimJEB: Simulated Jet Engine Bracket Dataset.* Computer Graphics Forum 40 (2021) 9–17. arXiv:2105.03534. Data: https://simjeb.github.io
- Hong et al. *DeepJEB: 3D Deep Learning-based Synthetic Jet Engine Bracket Dataset.* arXiv:2406.09047 (2024). Data: https://www.narnia.ai/dataset
- Mudur et al. *FEABench.* arXiv:2504.06260 (2025).
- Yao et al. *ReAct: Synergizing Reasoning and Acting in Language Models.* ICLR 2023. arXiv:2210.03629.
- Upadhyay & Reinhart. *Constrained natural-language interface for FEniCS.* arXiv:2606.10928 (2026, preprint).
- CAX-Agent. arXiv:2605.15218 (2026, preprint).
