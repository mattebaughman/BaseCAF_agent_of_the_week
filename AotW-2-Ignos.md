<p align="center">
  <img src="images/CAF-AotW-banner.svg" width="100%" alt="CAF AotW banner">
</p>

# 02/16/2026 &mdash; AotW#2: Ignos Fusion Foundation Model

---

## Science Story

Fusion energy research requires navigating thousands of published papers and running complex multiscale simulations across leadership-class machines &mdash; a cycle of literature review, simulation configuration, HPC execution, and result interpretation that can take weeks per research question when performed manually. The Fusion Agent is a closed-loop agentic system, built on the Academy framework, that automates this entire pipeline. It performs distributed literature search, synthesizes findings, parameterizes simulations, and executes them through Ignos &mdash; the Fusion Foundation Model &mdash; which routes workloads to MATEY on Frontier or Helios on Aurora via Globus Compute, compressing question-to-insight time from weeks to hours.

---

## Agentic Motivation

- **Distributed parallel literature search.** Multiple `SearcherAgent` instances each analyze a different paper from the fusion literature corpus in parallel, extracting question-relevant insights simultaneously across the search space.
- **Autonomous literature-to-simulation pipeline.** A `SynthesizerAgent` consolidates searcher outputs into unified context, which feeds directly into simulation parameterization &mdash; eliminating manual translation from papers to executable configurations.
- **Cross-facility HPC execution via Ignos.** Ignos routes workloads to MATEY on Frontier (OLCF) or Helios on Aurora (ALCF) through Globus Compute, targeting seamless exascale execution across DOE facilities.
- **Closed-loop iterative refinement.** The runner orchestrates multiple iterations of the full search-synthesize-simulate cycle, evaluating results against the research question and re-entering the loop with refined parameters.

---

## Implementation

### Academy Agent Architecture

The Fusion Agent is built on [Academy](https://github.com/academy-agents/academy) (`academy-py`), an open-source Python middleware for deploying stateful, autonomous agents across federated research infrastructure. Each agent in the pipeline is a Python class inheriting from Academy's base `Agent` type, with remotely invocable methods marked by the `@action` decorator. Agent lifecycle hooks (`agent_on_startup`, `agent_on_shutdown`) handle initialization and teardown. A `Manager` created via `Manager.from_exchange_factory()` manages agent lifecycles, and `manager.launch(AgentClass, kwargs={...})` spawns each agent, returning a typed `Handle[AgentType]` &mdash; a client-side proxy that translates method calls into asynchronous action requests.

An early prototype (accessible within the Ignos git repo) demonstrates the core flow with three agent classes and a runner script that orchestrates the iterative loop:

### Agent Classes

**`SearcherAgent`** &mdash; A lightweight LM-backed agent replicated *N* times (configurable via `--num-readers`). Each instance retrieves relevant articles from the fusion arXiv abstracts corpus. Its single `@action`, `analyze_abstract(abstract, question)`, prompts an LLM to produce a one-sentence synthesis of the abstract in the context of the research question. All *N* searcher handles are invoked concurrently via `asyncio.gather()`, implementing a fan-out parallel RAG pattern.

**`SynthesizerAgent`** &mdash; A single instance that receives the list of all searcher outputs. Its `@action`, `synthesize(question, searcher_outputs)`, prompts the LLM to consolidate the individual analyses into a comprehensive answer &mdash; resolving contradictions, identifying consensus, and highlighting the most relevant insights. This is the fan-in complement to the searcher fan-out.

**`SimulationAgent`** &mdash; Receives the synthesized insights and manages simulation execution. Exposes two `@action` methods: `decide_parameters(insights)` selects simulation configuration (turbulence model, resolution, kinetic model), and `run_simulation(params)` executes the run. The current prototype reads pre-existing Gkeyll two-stream instability output (`rt_vlasov_twostream_p1-stat.json`) and returns real simulation metrics (RK3 time, species RHS time, update counts, stage-2 failures). Fully connecting insights to simulation parameterization is in development.

### Runner Loop

The `runner.py` script orchestrates the full pipeline across multiple iterations:

1. **Abstract analysis** &mdash; launch *N* `SearcherAgent` instances and one `SynthesizerAgent` via the manager; fan out searcher tasks with `asyncio.gather()`; pass outputs to the synthesizer.
2. **Simulation** &mdash; launch a `SimulationAgent`; call `decide_parameters()` with the synthesized insights; call `run_simulation()` with the resulting configuration.
3. **Iterate** &mdash; repeat for a configurable number of iterations, collecting results into a summary table of Gkeyll metrics across runs.

All agents communicate through Academy's exchange layer (currently `LocalExchangeFactory` for single-node execution).

<p align="center">
  <img src="images/2-Ignos/agent_architecure.png" width="60%" alt="Fusion Agent architecture: iterative loop between RAG + Synthesis and Simulation + FM agents">
</p>
<p align="center"><em>Fusion Agent architecture: an input query drives an iterative loop between distributed RAG + Synthesis (right) and Simulation + FM execution (left), coordinated through Academy agents and sub-agents.</em></p>

### Target Architecture: Ignos and Cross-Facility Execution

The production system extends this prototype by replacing the local simulation stub with **Ignos**, the Fusion Foundation Model. Ignos is a client harness that performs content-based routing to the appropriate inference backend:

- **MATEY** on Frontier (AMD MI250X, OLCF) &mdash; spatiotemporal physics: CFD, PDE solvers, plasma dynamics
- **Helios** on Aurora (Intel Data Center Max, ALCF) &mdash; tokamak diagnostics: BES, ECE, ECEI
- **Traditional simulation codes** &mdash; SOLPS-ITER, XGC, Gkeyll, Navier-Stokes solvers

Cross-facility remote execution of the Fusion Foundation Model is managed through Globus Compute.

<p align="center">
  <img src="images/2-Ignos/ffm.png" width="60%" alt="Ignos Fusion Foundation Model: reasoning model, model selector routing to MATEY and Helios, and tool stack">
</p>
<p align="center"><em>Ignos (Fusion Foundation Model): a reasoning model selects between MATEY and Helios predictive and interpretive sub-models via a model selector, with a tool stack providing access to Gkeyll, BEAST/Transp, surrogates, and other simulation codes.</em></p>


---

## To Know More

### Source Code
- **Ignos (Fusion Foundation Model):** https://github.com/AI-ModCon/fusionFM
- **MATEY:** https://github.com/ORNL/MATEY
- **Helios:** https://github.com/Sande33p/HELIOS
- **Academy Framework:** https://github.com/academy-agents/academy
- **Flowcept:** https://github.com/ORNL/flowcept

### Additional Resources
- **MATEY Paper:** [arXiv:2412.20601](https://arxiv.org/abs/2412.20601)
- **Academy Paper:** [arXiv:2505.05428](https://arxiv.org/abs/2505.05428)
- **Academy Documentation:** https://docs.academy-agents.org/stable/
- **Globus Compute:** https://www.globus.org/compute
- **Frontier (OLCF):** https://www.olcf.ornl.gov/frontier/
- **Aurora (ALCF):** https://www.alcf.anl.gov/aurora
- **Contact:** Matt Baughman, mbaughman@pppl.gov

---

*Last Updated: 02/16/2026*
*Contributed by: Matt Baughman, mbaughman@pppl.gov*
