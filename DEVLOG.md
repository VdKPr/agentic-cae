# DEVLOG

Every decision, result and failure, in order. Newest entries at the bottom.

---

## Entry 0 — The idea (12 Oct 2026)

**Idea.** An "AI agent for ANSYS": describe the problem in plain language (eventually by voice), and the agent creates the CAD, applies boundary conditions, runs the analysis and presents the results — following the usual CAE workflow from geometry to report. It should combine FEM with deep learning, including deciding which analysis to use.

**Approach.** Start with the simplest version that works end to end (brute force), then add complexity one step at a time, recording each step here.

**Open question at this point:** is this doable for one person, and where exactly does AI help versus hurt?

---

## Entry 1 — Feasibility study and literature review (12 Oct 2026)

**What I checked.** Published LLM agents for FEA and CFD, text-to-CAD work, ML surrogates, hybrid neural–numerical solvers, vendor products, and the Ansys SimAI e-book with every resource it lists.

**Key findings**
- End-to-end "prompt → simulation" already exists in research. The open problem is *reliability*: on FEABench, 88% of the best agent's solver calls were executable, but it fully solved no problem.
- The systems that work keep the LLM out of the numerics: LLM → validated JSON spec → human-verified solver templates → automatic verification.
- Recovery loops matter more than prompting: CAX-Agent's completion rose from 69.3% without recovery to 92.7% with it.
- Vendor surrogate claims (SimAI case studies) rest on small, in-distribution test sets; useful as direction, not as evidence.
- Ansys Student is for educational use only, so the core stack must be open source.

**Decisions**
1. The LLM never writes solver physics.
2. Open-source core: CadQuery, gmsh, FEniCSx, CalculiX.

Full review: [docs/00_pre_literature_review.md](docs/00_pre_literature_review.md)

---

## Entry 2 — Business objective analysis and freeze (14 Oct 2026)

**Question.** Which business problem should the agent solve first, given that the data to validate it must actually be available?

**Candidates considered.** Faster verified verdicts (time); lightweighting (material cost); substrate warpage screening (scrap); a trusted surrogate (compute cost); faster quoting (sales).

**Data check**
- **SimJEB:** 381 bracket designs with CAD, meshes and reference FEA results — public.
- **DeepJEB:** 2,138 designs with simulation labels — public.
- **Substrate warpage:** no public data; it would have to be generated.
- **Manufacturing quotes:** no public ground truth at all, so quoting was dropped.

**Decision: freeze O1 + O2 on structural brackets.** They have public reference FEA to check against, they form a genuine ReAct loop (the agent decides each next design change), and they lead naturally to a surrogate (v2) and a substrate extension (v3). The earlier review recommended substrates first; public data availability changed that order.

**Agent framework:** ReAct (Thought → Action → Observation), with every action a verified tool.

**Something learned while checking the reference data:** SimJEB used first-order tetrahedra, with bolts and loads applied through rigid RBE2 and distributing RBE3 elements. So reproduction must use matching element order, and stress must be compared away from the bolt and load interfaces.

Problem statement: [docs/01_problem_statement.md](docs/01_problem_statement.md)

**Next:** environment setup (FEniCSx in WSL2), the four analytical benchmarks, reproducing one SimJEB bracket, and timing the same task manually in Ansys Student as the baseline.
