# Pre-Literature and Technology Review

*v0.1 · 11 Oct 2026 · snapshot taken before the project objective was frozen.*

> **Note (14 Oct 2026).** §7 below recommended starting with power-electronics substrates. After checking which public reference data exist, the objective was frozen on structural brackets first, because SimJEB and DeepJEB provide reference FEA to validate against. Substrates remain the planned extension. The decision and its reasoning are in [01_problem_statement.md](01_problem_statement.md).

The most defensible and buildable version of "Agentic CAE" is not an LLM that does FEA. It is a verified pipeline:

speech/text → LLM → validated JSON spec → deterministic CAD/mesh/solver templates (CadQuery, gmsh, FEniCSx/CalculiX) → automatic verification and repair → report.

Neural components (surrogates, and a neural warm start for nonlinear Newton solves) then speed it up, without ever being trusted blindly. The 2025–2026 literature and Ansys's own 2026 R1 release both converge on this design.

## TL;DR

- **Where the field is (Oct 2026).** LLM agents can now reliably *set up and run* simple simulations when an orchestration harness constrains them. Foam-Agent reaches 88.2% execution success on 110 Basic-tier OpenFOAM tasks and 62.5% on its out-of-distribution Advanced tier. CAX-Agent reaches 92.7% completion on 50 APDL tasks with model-driven recovery, versus 69.3% without. On harder, realistic benchmarks they still fail to fully solve problems: on FEABench, 88% of API calls were executable but no problem was solved end to end. Vendors (Ansys 2026 R1 Mesh Agent and Discovery Validation Agent; Navier; Neural Concept) are productising exactly this "agent wraps solver" layer.
- **What the Ansys SimAI e-book really shows.** Physics-agnostic surrogates work on narrow design families trained on roughly 10–2,250 prior simulations. The evidence is entirely vendor case studies, with small test sets and loosely defined error metrics. For this project, surrogates are a *downstream consumer* of the clean, automatically generated data an agent produces, so agentic setup complements SimAI rather than competing with it.
- **Open gaps one person can own:** independent physics verification (not self-validation), benchmarking of analysis-type selection, domain-specific agents, out-of-distribution gating between surrogate and full FEM, and neural solver acceleration deployed safely inside an agent.

---

## 1. Purpose, scope and evidence conventions

This document surveys the AI + simulation space to inform the problem statement for Agentic CAE. The idea: describe an engineering problem in plain language (later by voice); the AI creates the CAD, runs the analysis, presents the results, and helps choose the right analysis type, combining FEM and deep learning.

**Evidence labels used throughout:**

| Label | Meaning |
|---|---|
| **[PR]** | Peer-reviewed journal or conference paper |
| **[PP]** | Preprint (arXiv), not yet peer reviewed — treat numbers as provisional |
| **[V]** | Vendor marketing, vendor white paper or press release — claims not independently verified |
| **[N]** | Trade press, news or funding databases |

Background from an earlier feasibility study:
- Keep the LLM out of the numerics. FEABench showed that 88% executable API calls still produced zero fully solved problems.
- Recovery loops matter: CAX-Agent went from 69.3% to 92.7% completion with them.
- Ansys 2026 R1 shipped the portfolio's first agentic features.
- A neural initial guess for Newton's method is a natural accelerator for nonlinear solves. Close prior art: Neural-Initialized Newton (IJNME 2026) and a neural hybrid Newton solver (J. Comput. Phys. 2025).

---

## 2. Taxonomy of "AI + simulation"

### 2.1 Overview

| Branch | What it does | Maturity (Oct 2026) | Representative works / products | Typical data needs | Key limitations |
|---|---|---|---|---|---|
| **(a1) Physics-agnostic surrogates** | Learn field or scalar outputs directly from past solver runs on (often unparameterised) geometry | **Commercial**: SimAI, PhysicsX, Neural Concept, Monolith | Ansys SimAI (implicit neural representation), PhysicsX platform / "Large Physics Models", Neural Concept (geometric DL) | ~10–300 simulations per narrow design family (vendor claims); thousands for broad families | Interpolation only; silent failure out of distribution (OOD); error metrics often scalar or global; the cost of generating data moves upstream |
| **(a2) Physics-aware surrogates** | Embed PDE structure: physics loss terms (PINNs), operator learning (FNO, DeepONet), mesh-graph message passing (MeshGraphNets), attention on meshes (Transolver), NVIDIA DoMINO / X-MeshGraphNet | **Research → early product** (NVIDIA PhysicsNeMo; Luminary uses DoMINO/GeoTransolver) | PINNs; FNO; DeepONet; MeshGraphNets; Transolver; DoMINO; X-MeshGraphNet | PINNs: little or no data but expensive training; operators/GNNs: hundreds to tens of thousands of solves | PINNs struggle with stiff, high-order or multiscale PDEs; operators need regular grids or careful geometry encoding; generalisation across geometry and BC classes is still weak |
| **(b) LLM/agentic CAE workflow automation** | LLM plans and writes or fills solver inputs (APDL, COMSOL API, OpenFOAM dictionaries, FEniCS UFL), runs them, reads logs, repairs | **Research boom 2024–2026; first vendor features 2026** | FEABench, MechAgents, FeaGPT, Foam-Agent, MetaOpenFOAM, CAX-Agent, PDE-Agents, ALL-FEM, constrained FEniCS NL interface; Ansys Mesh Agent / Discovery Validation Agent; Navier | No training data needed for prompting; RAG over docs and tutorials; fine-tuning uses 1,000+ verified FEniCS scripts built from 503 expert-written seed scripts (ALL-FEM, CMAME 2026) | Hallucinated physics/BCs; executable ≠ correct; weak verification; benchmarks use simple geometries |
| **(c) Text-to-CAD / generative geometry** | Text, image or point cloud → parametric CAD (command sequences or CadQuery code) | **Research → early product** (Zoo, Leo AI, Ansys GeomAI, Neural Concept generative CAD) | Text2CAD, CAD-Recode, CADCodeVerify, Text-to-CadQuery, CAD-Coder | 100k–1M synthetic CAD programs (e.g., 170,000 CadQuery annotations in Text-to-CadQuery; 1M synthetic sequences for CAD-Recode) | Mostly sketch-and-extrude parts; weak dimensional accuracy and design-intent constraints; little attention to simulation-readiness (clean faces, named boundaries) |
| **(d) Hybrid numerical–neural solvers** | A neural net supplies an initial guess, preconditioner or correction; the classical solver guarantees the final answer | **Research (fast-growing 2025–2026)** | Neural-Initialized Newton (IJNME 2026), neural hybrid Newton (JCP 2025), Int-Deep, neural-operator warm starts, neural preconditioners | Tens to thousands of solves, or physics-informed training; small nets are often enough | Speed-up depends on the network's cost versus the Newton iterations saved; must fall back gracefully; few industrial-scale demonstrations |
| **(e) Documentation copilots** | RAG assistants over vendor docs and forums inside the GUI | **Commercial, mature** | AnsysGPT; Ansys Engineering Copilot (2025 R2, extended in 2026 R1); Siemens Industrial Copilot / Simcenter assistants | Vendor docs, tickets, training content | Advice, not execution; no guarantee on numerical settings |

### 2.2 Interpretation for this project

- The highest-value, least-crowded intersection is **(b) × (d) with verification**. Agentic setup is crowded at the "LLM writes code" level but thin at the "verified + domain-specific + neural-accelerated" level. The author's M.Tech thesis (RANG, a residual-aware neural initial guess for a nonlinear fourth-order FEM problem) already sits inside (d).
- **(a1)** is where the money is (PhysicsX at a ~$2.4B valuation; Neural Concept's $100M round), but one person cannot out-build those surrogates. The better move is to *produce clean training data automatically* (the agent) and *consume* surrogates behind an OOD gate.
- **(c)** is the weakest link for simulation, because generated CAD is rarely simulation-ready. A parametric CadQuery template library is far more reliable than free-form text-to-CAD. Use LLM text-to-CAD only as a fallback for non-catalogue geometry, the pattern the Penn State FEniCS paper uses for gmsh.

---

## 3. Critical review of the Ansys e-book "Combine AI with Simulation for Next-level Innovation" (© 2024)

### 3.1 What it claims [V]

- **Speed:** "up to 100X faster"; a "10–100X" performance boost; predictions in minutes (ship hull: ~8 s per prediction).
- **Product:** SimAI is a cloud-native, physics-agnostic SaaS. Models are trained on prior simulation data (Ansys or not) and use the geometry itself as input, with no parameterisation required.
- **Architecture (high level only):** An implicit neural representation learns a continuous field function. Regularisation lets it generalise to unparameterised geometries. A **confidence score** is the distance to the nearest known geometry in a learned high-dimensional space.
- **Security/infrastructure:** AWS eu-west-1; encrypted S3; TLS 1.3; Kubernetes on EKS; KMS AES-256; PySimAI SDK (Python ≥3.9).
- **Framing:** "physics-aware" ML (replace solver bottlenecks / add physics losses) versus "physics-agnostic" ML (learn latent physics from solver outputs — SimAI's approach).
- **Portfolio:** AI+ modules (Granta MI, optiSLang, CFD turbulence, SynMatrix, Structures thermal-strain and compute prediction, Missions, Electronics) and the AnsysGPT assistant.

### 3.2 Case studies and data volumes [V]

| Case | Training data | Claimed result | Evidence-quality notes |
|---|---|---|---|
| Ship hull | 288 CFD (244/20/20) | <5% resistance error; ~50 s vs 4 h on 32 cores; ~8 s/prediction | Best documented: separate white paper with field comparisons; still only a 20-case test set |
| Bumper crash | 98 designs × 21 states = 2,058 | "Most" within 10% of LS-DYNA | "Most" is unquantified; transient nonlinear — impressive if true |
| Chip thermal | 2,250 MAPDL | <0.5% temperature difference; <1 min vs 30 min (4 cores) | Largest dataset; "% temperature difference" in K vs °C is ambiguous |
| Jet-engine bracket | ~250 | — | No metric reported |
| SUV aero | ~50 Fluent | <0.5% drag error | Scalar drag only; small family |
| Hairpin wire forming | — | <5% shape difference | No data volume |
| Crystal plasticity | 50 microstructures × 10 strains | <3% | Metric undefined |
| Crane hook | 50 | <1% stress difference | Peak or field-averaged? Unstated |
| Rubber bush | ~10 designs | 5 min vs 1.3 h | Very small; likely a smooth design family |
| Battery thermal | ~10 CFD | — | No metric |
| Power-inverter cooling | ~30 Fluent | 30 s | Speed only |
| Centrifugal pump | 113 | — | — |
| Stirred tanks | 28 | — | — |
| Shell-and-tube HX | 250 CHT | 99% time reduction | Speed only |
| Permanent magnets | ≤100 Maxwell | 20 s vs 5 min | — |
| Traction motor | — | 20–50× faster | — |
| Optics in fog | 10 | 2% max illuminance error | Tiny dataset |

### 3.3 Evidence-quality rating

**Overall: low to moderate (vendor case studies).** Specific weaknesses:

1. **No peer review, no public datasets, no code.** None of the 17 cases can be reproduced.
2. **Tiny, in-distribution test sets.** For example, 20 test hulls drawn from the same parametric family. There is no OOD test, so this is interpolation performance.
3. **Unstated or inconsistent error metrics.** Relative scalar error, maximum field error, mean field error and "shape difference" are mixed; several cases report speed only.
4. **Speed comparisons are apples to oranges.** Inference time is compared with full solve time, while the cost of generating the 10–2,250 training simulations and training the model is left out. Break-even depends on how many predictions you make.
5. **Selection bias.** Only successes are shown.
6. **Architecture is described qualitatively.** "Implicit neural representation + regularisation" is a design family, not a method anyone can check.

Still, the e-book is **directionally consistent with peer-reviewed work**: MeshGraphNets, Transolver and DoMINO also show accurate in-distribution field prediction from hundreds to thousands of samples. Its real contribution is product packaging: geometry as input, no parameterisation, a confidence score, and enterprise security.

### 3.4 Implications for Agentic CAE

1. **Data budgets are small enough for one person.** 10–300 simulations per design family can be generated overnight on a laptop for simple linear models in FEniCSx or CalculiX. *An agent that generates, verifies and labels that dataset automatically is itself valuable*, and it is the bottleneck SimAI users describe.
2. **Copy the confidence-score idea.** A nearest-neighbour distance in a geometry/spec embedding decides whether to send a request to the surrogate (fast) or the full FEM (trusted). It is a cheap, explainable OOD gate.
3. **Agentic setup is complementary, not competing.** SimAI assumes someone already produced consistent, correctly set-up simulations, so it accelerates the *n*th evaluation. An agent automates the *first* evaluations: spec → geometry → mesh → BCs → solve → verify. Ansys itself now ships both: SimAI connectors in optiSLang for data generation, and the Mesh Agent / Discovery Validation Agent for setup.
4. **Report metrics properly from day one:** field L2 error, maximum error, error at hotspots, calibrated uncertainty, and cost including data generation. This is what the vendor material lacks.

---

## 4. E-book resources and 2025–2026 updates

### 4.1 Resources listed at the end of the e-book

| Type | Title | URL | Summary | Status |
|---|---|---|---|---|
| Webinar | Introducing Ansys AI: A New Chapter in Engineering Simulation | https://www.ansys.com/webinars/introducing-ansys-ai-a-new-chapter-in-engineering-simulation | Overview of the 2024 AI portfolio: SimAI, AnsysGPT and AI+ add-ons (starting with optiSLang AI+ and Granta MI AI+). Pitch: use prior simulation and measurement data to assess new designs "in minutes instead of hours". [V] | Found (on demand) |
| Webinar | Introducing Ansys SimAI: Cloud-native Generative AI for Simulation | https://ansys.synopsys.com/webinars/introducing-ansys-simai-cae-platform-powered-by-generative-ai | SimAI launch webinar: a cloud-native app for "all simulation engineers and designers, Ansys or not, in every physics and industry". [V] | Found |
| Webinar | Engineering in the AI Era | https://www.ansys.com/webinars/engineering-in-the-ai-era | Management-level talk on turning legacy data into assets; low technical depth. [V] | Found |
| Webinar series | Accelerate Your Simulation with AI (SimAI webinar series) | https://www.ansys.com/events/accelerate-your-simulation-webinar-series | Multi-episode on-demand series showing SimAI by physics. The most useful e-book resource for seeing real workflows. [V] | Found (title on the site differs slightly from the e-book) |
| Article | Explaining SimAI: How AI is Applied to Numerical Simulation In Practice | https://ansys.synopsys.com/blog/explaining-simai | The "upload data, train AI model, predict" workflow and the physics-agnostic approach; same source as the e-book's architecture section. [V] | Found |
| Article | From Chips to Ships: Optimize Design With the Ansys SimAI Platform | https://www.ansys.com/blog/chips-ships-optimize-design-ansys-simai-platform | Four scenarios: chip thermal (MAPDL), SUV aero (Fluent), ship hull (288 CFD results), plus one more. [V] | Found |
| Article | Maximize Simulations With Secure, Cloud-native AI | https://ansys.synopsys.com/blog/maximize-simulations-secure-cloud-native-ai | AWS architecture and security; source of the e-book's security claims. [V] | Found |
| Article | The Intersection of AI and Simulation Technology | https://ansys.synopsys.com/blog/simulation-and-ai | General essay (Mar 2024) on AI democratising simulation. [V] | Found |
| White paper | How to Improve Hull Designs with Ansys SimAI | https://www.nafems.org/downloads/dropbox/nologin/nrc24_americas/ansys/how_to_improve_hull_designs_with_ansys_simai.pdf | 288 CFD samples; validation against CFD on volume, surface, evolution curves and coefficients. **Read this one first** — the most quantitative. [V] | Found |
| White paper | Ansys SimAI Software Predicts Fully Transient Vehicle Crash Outcomes | https://ansys.synopsys.com/resource-center/white-paper/simai-predicts-transient-vehicle-crash | Bumper crash vs LS-DYNA: kinematics, deformation, wall forces. Gated (registration). [V] | Found (gated) |

### 4.2 Important 2025–2026 updates

| Date | Event | Source type | Implication |
|---|---|---|---|
| 26 Mar 2025 | **Siemens closes the Altair acquisition** (enterprise value about USD 10 billion) | [V] | Siemens now owns HyperWorks, PhysicsAI and romAI |
| 17 Jul 2025 | **Synopsys completes its acquisition of Ansys** | [V/regulatory] | Ansys is now "Ansys, part of Synopsys" |
| 29 Jul 2025 | **Ansys 2025 R2**: Engineering Copilot across Mechanical, Discovery, Fluent, HFSS and more; optiSLang–SimAI integration | [V] | Copilots (category e) are now standard |
| 6 Oct 2025 | **CoreWeave announces it will acquire Monolith AI** | [V]/[N] | AI-cloud players are buying simulation AI |
| 15 Dec 2025 | **Navier emerges with a $5.6M seed** (GV, HCVC, Y Combinator) | [N] | "Agent-Driven Engineering" between CAD and simulation — a direct comparable |
| 18 Dec 2025 | **Neural Concept $100M round** (Goldman Sachs Alternatives) | [V] | Generative CAD feature planned for 2026 |
| 11 Mar 2026 | **Ansys 2026 R1**: "the portfolio's first agentic capabilities" — **Mesh Agent** in Mechanical (debugs meshing failures), **Discovery Validation Agent** (flags setup issues; early evaluation), **GeomAI**, **SimAI Pro** (desktop) and **SimAI Premium** (cloud) | [V] | Validates the agentic-setup thesis; Ansys targets the same meshing and setup failures CAX-Agent catalogued |
| 8 Jun 2026 | **PhysicsX $300M Series C at ~$2.4B**, led by Temasek | [N] | Strong investment signal for physics AI |
| Ongoing | **PySimAI** (`ansys-simai-core`; docs at simai.docs.pyansys.com) | [V] | Shows the API shape of a commercial surrogate platform |
| Ongoing | **NVIDIA PhysicsNeMo** (formerly Modulus; GitHub NVIDIA/physicsnemo) | [V] | Open-source home of DoMINO, X-MeshGraphNet and Transolver-style models |

---

## 5. Key academic literature

### 5.1 Agentic FEA/CFD

| Work | Date / venue | Link | One-line contribution | Relevance |
|---|---|---|---|---|
| FEABench (Mudur et al.) | 2024 NeurIPS MATH-AI workshop; arXiv Apr 2025 [PR workshop / PP] | https://arxiv.org/abs/2504.06260 | 15 verified COMSOL problems; best agent produces 88% executable API calls but solves no problem completely and correctly | **Core motivation**: executable ≠ correct; evaluation template |
| MechAgents (Ni & Buehler) | 2023–24, Extreme Mechanics Letters [PR] | https://arxiv.org/abs/2311.08166 | Multi-agent LLM teams that write and self-correct FEniCS elasticity code | Early proof of concept |
| FeaGPT (Qi, Xu, Chu) | Oct 2025 [PP] | https://arxiv.org/abs/2510.21993 | Conversational Geometry–Mesh–Simulation–Analysis pipeline over CalculiX | Closest open comparable; verification is self-validation only |
| MetaOpenFOAM | 2024 [PP] | https://arxiv.org/abs/2407.21320 | RAG-based multi-agent OpenFOAM automation | CFD agent baseline |
| Foam-Agent / Foam-Agent 2.0 (Yue et al.) | 2025–2026 [PR / PP] | https://arxiv.org/abs/2505.04997 ; https://arxiv.org/abs/2509.18178 | Multi-agent OpenFOAM pipeline with hierarchical RAG and a reviewer loop; 88.2% execution success on 110 Basic-tier tasks | Structure and error correction dominate the gains |
| CAX-Agent | May 2026 [PP] | https://arxiv.org/abs/2605.15218 | MAPDL harness with a recovery ladder (rule patch → model regeneration → context enrichment → human); completion 0.69 / 0.77 / 0.93 | Orchestrator owns retries — copy this design |
| PDE-Agents | Jun 2026 [PP] | https://arxiv.org/abs/2606.07850 | Multi-agent FE simulation with knowledge-graph-augmented reasoning | A route to "which analysis to use" |
| Constrained NL interface for FEniCS (Upadhyay & Reinhart) | Jun 2026 [PP] | https://arxiv.org/abs/2606.10928 | LLM only parses into a closed JSON spec and writes gmsh code for non-catalogue geometry; five human-written FEniCS templates | **Blueprint for v0** |
| ALL-FEM (Deotale et al.) | 2026, CMAME [PR] | https://www.sciencedirect.com/science/article/abs/pii/S0045782526002586 | Fine-tunes open LLMs on 1,000+ verified FEniCS scripts | Fine-tuning data scale for FEM code |
| VFEAgent | May 2026 [PP] | https://arxiv.org/abs/2605.28978 | Multimodal (image + text) multi-agent FEA automation | Sketch/photo input |
| "What do CAE simulation agents really need beyond a generic harness?" | Sep 2026 [PP] | https://arxiv.org/abs/2609.03718 | Cross-benchmark analysis of CAE agents | Latest meta-analysis — read before finalising |
| Agentic AI for staged 3D FE tunnel modelling | Oct 2026 [PP] | https://arxiv.org/abs/2610.00056 | Four agents + 15 skills over MCP for PLAXIS 3D, with an independent audit gate | Template for a domain-specific agent with independent verification |

### 5.2 Text-to-CAD

| Work | Date / venue | Link | Contribution | Relevance |
|---|---|---|---|---|
| Text2CAD (Khan et al.) | NeurIPS 2024 [PR] | https://arxiv.org/abs/2409.17106 | Text → CAD command sequences on DeepCAD | Baseline |
| CAD-Recode (Rukhovich et al.) | ICCV 2025 [PR] | https://arxiv.org/abs/2412.14042 | Point cloud → CadQuery Python via a small LLM trained on 1M synthetic programs | Reverse engineering into editable code |
| CADCodeVerify | ICLR 2025 [PR] | https://arxiv.org/abs/2410.05340 | VLM generates CAD code and verifies it against renders | Verification loop for geometry |
| Text-to-CadQuery (Xie & Ju) | May 2025 [PP] | https://arxiv.org/abs/2505.06507 | 170,000 CadQuery annotations; fine-tuned models improve exact match and Chamfer distance | Justifies CadQuery as the geometry language |
| CAD-Coder | May 2025 [PP] | https://arxiv.org/abs/2505.19713 | SFT + GRPO with a Chamfer-distance reward | RL-with-geometric-reward option for later |

### 5.3 Surrogates

| Work | Date / venue | Link | Contribution | Relevance |
|---|---|---|---|---|
| DeepONet (Lu et al.) | Nature Machine Intelligence 2021 [PR] | https://arxiv.org/abs/1910.03193 | Branch–trunk operator networks | Parametric load/BC → field operators |
| FNO (Li et al.) | ICLR 2021 [PR] | https://arxiv.org/abs/2010.08895 | Spectral-convolution neural operator on regular grids | Fast warm-start generator on structured domains |
| MeshGraphNets (Pfaff et al.) | ICLR 2021 [PR] | https://arxiv.org/abs/2010.03409 | Message-passing GNN on simulation meshes | Native FEM-mesh surrogate |
| Transolver (Wu et al.) | ICML 2024 [PR] | https://arxiv.org/abs/2402.02366 | Physics-attention over learned slices of mesh points | Strong general mesh surrogate |
| DoMINO (NVIDIA) | Jan 2025 [PP] | https://arxiv.org/abs/2501.13350 | Multi-scale neural operator for large point clouds | Industrial scale |
| X-MeshGraphNet (NVIDIA) | Nov 2024 [PP] | https://arxiv.org/abs/2411.17164 | Scalable multi-scale MeshGraphNet | Scaling GNNs to large meshes |

### 5.4 Hybrid numerical–neural solvers

| Work | Date / venue | Link | Contribution | Relevance |
|---|---|---|---|---|
| Neural-Initialized Newton (Taghikhani et al.) | IJNME 2026 [PR] | https://onlinelibrary.wiley.com/doi/10.1002/nme.70380 | Physics-informed neural field gives the Newton initial guess | **Closest prior art** to a neural warm start |
| Fast neural hybrid Newton solver (Jin et al.) | J. Comput. Phys. 2025 [PR] | https://arxiv.org/abs/2407.03945 | Neural initial guesses for Newton in implicit time-stepping, including Allen–Cahn | Direct overlap with an Allen–Cahn setting |
| Int-Deep (Huang, Wang, Yang) | J. Comput. Phys. 2020 [PR] | https://arxiv.org/abs/1903.07601 | Deep-learning-initialised Newton-type methods with convergence analysis | Foundational precedent |
| Neural-operator warm starts (Eshaghi et al.) | Nov 2025 [PP] | https://arxiv.org/abs/2511.02481 | Warm-starting iterative solvers with neural operators | Benchmarks of the idea |
| Neural-operator preconditioned Newton (Lee et al.) | Nov 2025 [PP] | https://arxiv.org/abs/2511.08811 | Neural operator as a nonlinear preconditioner | The next step beyond initial guesses |
| Pretrain Finite Element Method (PFEM) | Jan 2026 [PP] | https://arxiv.org/abs/2601.03086 | Neural-operator pretraining + FEM warm start at the final load step | Competing framework |
| Interface-aware neural Newton preconditioning for cohesive zones | 2026 [PP] | https://arxiv.org/abs/2606.31921 | Neural Newton preconditioning for delamination | Relevant to metal–ceramic interfaces |

**Interpretation.** "A neural initial guess for Newton" is no longer new; NiN, Jin et al. and Int-Deep establish it. Still-open contributions: (i) fourth-order PDE settings; (ii) residual-aware training or selection of the guess; (iii) **deployment as a safe accelerator inside an agentic pipeline, with a fallback**. Point (iii) is new.

---

## 6. Industry landscape

| Company | What they build | Recent news |
|---|---|---|
| **Ansys (part of Synopsys)** | SimAI Pro/Premium, GeomAI, Mesh Agent, Discovery Validation Agent, Engineering Copilot, PyAnsys | 2026 R1 shipped first agentic features |
| **Siemens (incl. Altair)** | Simcenter, HyperWorks/PhysicsAI, romAI, Industrial Copilot | Altair acquisition closed Mar 2025 |
| **Dassault Systèmes** | SIMULIA (Abaqus, PowerFLOW), 3DEXPERIENCE virtual-twin AI | Not verified in this review |
| **Autodesk** | Fusion generative design; Autodesk Research text/image-to-3D | Not verified in this review |
| **NVIDIA** | PhysicsNeMo, Omniverse, DoMINO, CUDA-X solvers | Open-source physics-ML stack |
| **PhysicsX** | Physics-AI platform, "Large Physics Models", forward-deployed engineering | $300M Series C (Jun 2026) |
| **Neural Concept** | Geometric-DL surrogates, AI workflows, generative CAD | $100M round (Dec 2025) |
| **Navier** | Agents between CAD and simulation | $5.6M seed (Dec 2025) |
| **Luminary Cloud** | GPU-native cloud CFD + physics AI | $72M Series B (Sep 2025) |
| **SimScale** | Browser-based CAE, AI-assisted physics | — |
| **Synera** | Engineering-process automation + agents | — |
| **Leo AI** | Engineering copilot / text-to-CAD assistant | — |
| **Zoo (KittyCAD)** | Text-to-CAD API, KCL CAD language | — |
| **CoreWeave / Monolith** | AI cloud; Monolith's test-data ML for engineers | Acquisition announced Oct 2025 |

**What the field values.** The recurring technical bundle is PyTorch + 3D geometric deep learning (point clouds, meshes, graphs) + CAE domain knowledge (CFD/FEA) + data pipelines + customer-facing delivery, with LLM/RAG/agent skills now appearing. A credible project must therefore show **agent orchestration with verification** and, later, **3D mesh/point-cloud ML** — not just prompting.

---

## 7. Open gaps and candidate problem statements

### 7.1 Gaps identified

1. **Verification gap.** Most agents (FeaGPT, Foam-Agent) check executability or self-validate. Few use independent physics checks: energy balance, mesh convergence, analytical bounds, reaction-force equilibrium.
2. **Analysis-selection gap.** "Which analysis to use" (steady vs transient, linear vs nonlinear, thermo-mechanical coupling, fatigue) is mostly hard-coded or implicit; there is no benchmark for it.
3. **Domain gap.** No published agent targets power-electronics packaging (DBC/AMB substrates: thermal resistance, CTE-mismatch stress, warpage, metal–ceramic delamination).
4. **Trust gap between agents and surrogates.** Nobody routes agent requests between surrogate and FEM with a calibrated OOD score the way SimAI's confidence score suggests.
5. **Hybrid-solver deployment gap.** Neural Newton warm starts are demonstrated in isolation, never as an automatically invoked accelerator with a fallback inside an agentic workflow.

### 7.2 Candidate problem statements (as of 11 Oct 2026)

**P1 — SubstrateAgent: verified natural-language thermo-mechanical analysis of DBC/AMB substrates**
- *Objective:* request → validated JSON spec → parametric layer-stack geometry → gmsh with named physical groups → FEniCSx thermo-elastic solve (CalculiX cross-check) → independent verification (mesh convergence, energy balance, 1-D thermal-resistance bound, bimaterial-strip stress bound) → report.
- *Novelty:* a domain-specific agent for power-electronics substrates; independent physics verification; analysis-type selection evaluated on a benchmark.
- *v0 feasibility:* high (2-D/2.5-D or 3-D layer stack, linear thermo-elasticity, a handful of templates).
- *Thesis link:* an elastoplastic copper template (nonlinear Newton) where a neural warm start cuts iterations.
- *Industry relevance:* the open-source equivalent of vendor setup/validation agents, for a real industry.

**P2 — Agent-generated datasets + OOD-gated surrogate ("mini-SimAI")**
- *Objective:* the agent generates 50–500 verified simulations; train a Transolver/MeshGraphNet surrogate; add a nearest-neighbour confidence score that routes queries to surrogate or FEM.
- *Novelty:* closes the loop between agentic setup and surrogates; publishes the honest metrics vendor material lacks.
- *v0 feasibility:* medium-high; 8 GB VRAM suffices for small meshes.
- *Thesis link:* the surrogate's output can serve as the Newton initial guess.
- *Industry relevance:* matches physics-ML surrogate work at PhysicsX, Neural Concept, Luminary and NVIDIA.

**P3 — Safe neural Newton acceleration as an agent tool**
- *Objective:* package a residual-aware warm start as a FEniCSx tool that the agent invokes for nonlinear templates when the OOD score is low, with a guaranteed fallback and reported iteration savings.
- *Novelty:* fourth-order PDEs, residual-aware selection, first integration into an agentic workflow with a safety gate.
- *v0 feasibility:* high (reuses thesis code).
- *Industry relevance:* research-flavoured; strongest combined with P1.

**P4 — "Which analysis?" benchmark and planner**
- *Objective:* a 100–200 case benchmark of natural-language engineering questions labelled with the correct analysis type, coupling, BCs and expected result range; evaluate LLM planners with and without a knowledge graph or rules.
- *Novelty:* analysis selection is unbenchmarked.
- *v0 feasibility:* high (mostly curation).
- *Industry relevance:* LLM/agent evaluation work; benchmarks get cited.

**P5 — Interface-aware nonlinear substrate reliability with neural preconditioning**
- *Objective:* cohesive-zone delamination of copper/ceramic under thermal cycling, accelerated by a neural Newton preconditioner or warm start.
- *v0 feasibility:* medium-low (cohesive-zone convergence is hard).
- *Industry relevance:* simulation R&D; a long timeline; better as a later phase.

**Recommendation at the time:** P1 as the spine, P3 as the technical differentiator, P2 as the ML showcase.

> **Decision (14 Oct 2026):** the objective was frozen on *structural brackets first* — verified verdicts plus a lightweighting loop — because SimJEB and DeepJEB provide public reference FEA to validate against, while no public substrate data exist. P1-style substrates became the planned extension, and P2/P3 became later roadmap stages. See [01_problem_statement.md](01_problem_statement.md).

---

## 8. Recommendations at the time (superseded by the problem-statement roadmap)

1. Freeze a JSON schema; build three CadQuery/gmsh/FEniCSx templates; write verification checks first.
2. Add an LLM parser (closed schema, function calling) plus a CAX-Agent-style recovery ladder; build a benchmark with analytical bounds.
3. Auto-generate ~200 verified runs; train a small Transolver or MeshGraphNet; implement the nearest-neighbour confidence gate.
4. Plug in the neural warm start for an elastoplastic template; report Newton-iteration savings with a fallback. Publish the repo, a short demo video and a write-up.

---

## 9. Caveats

- Many agentic-CAE results are **preprints** from 2025–2026 with small, simple benchmarks; numbers may change on review.
- All Ansys e-book and SimAI numbers are **vendor claims** without public data.
- Funding figures come from press releases and news; where databases conflict, the company-announced figure is used.
- Several company details (Dassault, Autodesk, SimScale, Synera, Leo AI, Zoo) could not be verified in this review.
- The arXiv 2609.03718 meta-analysis could not be read in full.

---

## 10. Prioritised reading / watching list

1. Constrained NL interface for FEniCS (2026) — https://arxiv.org/abs/2606.10928 — the v0 blueprint.
2. CAX-Agent (2026) — https://arxiv.org/abs/2605.15218 — recovery-ladder design.
3. FEABench (2025) — https://arxiv.org/abs/2504.06260 — evaluation methodology and failure modes.
4. Neural-Initialized Newton (IJNME 2026) — https://onlinelibrary.wiley.com/doi/10.1002/nme.70380
5. Jin et al., neural hybrid Newton (JCP 2025) — https://arxiv.org/abs/2407.03945
6. Ansys white paper: How to Improve Hull Designs with Ansys SimAI — https://www.nafems.org/downloads/dropbox/nologin/nrc24_americas/ansys/how_to_improve_hull_designs_with_ansys_simai.pdf
7. Explaining SimAI (blog) — https://ansys.synopsys.com/blog/explaining-simai
8. Ansys 2026 R1 press release — https://news.synopsys.com/2026-03-11-Synopsys-Launches-Ansys-2026-R1-to-Re-Engineer-Engineering-with-Joint-Solutions-and-AI-Powered-Products
9. Foam-Agent (2025) — https://arxiv.org/abs/2505.04997
10. FeaGPT (2025) — https://arxiv.org/abs/2510.21993
11. NVIDIA PhysicsNeMo docs — https://docs.nvidia.com/physicsnemo/latest/ (GitHub: https://github.com/NVIDIA/physicsnemo)
12. Transolver (ICML 2024) — https://arxiv.org/abs/2402.02366
13. MeshGraphNets (ICLR 2021) — https://arxiv.org/abs/2010.03409
14. Accelerate Your Simulation with AI — SimAI webinar series — https://www.ansys.com/events/accelerate-your-simulation-webinar-series
15. Introducing Ansys SimAI webinar — https://ansys.synopsys.com/webinars/introducing-ansys-simai-cae-platform-powered-by-generative-ai
16. Text-to-CadQuery — https://arxiv.org/abs/2505.06507 ; CADCodeVerify — https://arxiv.org/abs/2410.05340
17. PySimAI docs — https://simai.docs.pyansys.com
18. From Chips to Ships (blog) — https://www.ansys.com/blog/chips-ships-optimize-design-ansys-simai-platform
19. Maximize Simulations With Secure, Cloud-native AI — https://ansys.synopsys.com/blog/maximize-simulations-secure-cloud-native-ai
20. The Intersection of AI and Simulation Technology — https://ansys.synopsys.com/blog/simulation-and-ai
21. FNO — https://arxiv.org/abs/2010.08895 ; DeepONet — https://arxiv.org/abs/1910.03193 ; DoMINO — https://arxiv.org/abs/2501.13350 ; X-MeshGraphNet — https://arxiv.org/abs/2411.17164
22. Int-Deep — https://arxiv.org/abs/1903.07601 ; neural-operator preconditioned Newton — https://arxiv.org/abs/2511.08811
23. FEniCSx tutorial (Dokken) — https://jsdokken.com/dolfinx-tutorial/ ; CadQuery docs — https://cadquery.readthedocs.io ; gmsh docs — https://gmsh.info/doc/texinfo/gmsh.html ; PyMAPDL — https://mapdl.docs.pyansys.com
