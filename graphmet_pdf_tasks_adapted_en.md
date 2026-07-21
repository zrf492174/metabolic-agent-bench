# graphMet Task Suite: Extracted from the PDF and Adapted to the Current Project

> Source: `media-1 (3).pdf`, primarily Table S13 (pages 16-22) and Table S14 (pages 22-25). Evaluation dimensions were adapted from Tables S1-S2 (pages 1-3).  
> Adaptation baseline: local graphMet repository as of 2026-07-21. This document is a task specification; it does not imply that every task is already implemented.

## 1. Executive Summary

The tasks in the source PDF are not software-development tickets. They are prompts designed to evaluate how well large language models handle genome-scale metabolic model (GEM/GSM) interpretation and analysis. The original tasks can be consolidated into six groups:

1. Model interpretation and domain knowledge;
2. FBA/FVA flux analysis;
3. Metabolic pathway gap filling and integration;
4. Metabolic flux optimization and knockout analysis;
5. Literature-driven case-study reproduction;
6. Model error detection.

The following changes were made for graphMet:

- Full JSON models are no longer embedded in prompts. Tasks use session uploads, attachment IDs, or session-scoped model paths to avoid context-window failures.
- General-purpose tasks default to the existing `models/ENGRO2_annotated.xml`; `models/e_coli_core.xml` may be used for fast tests.
- Organism-specific tasks involving iML1515, Yeast8/yeast850, iJR904, or iCW773 remain external-model tasks. They are not inappropriately replaced with a human model.
- Medium composition, oxygen conditions, carbon source, objective reaction, and product exchange must be explicit or confirmed. Reaction IDs must be discovered from the model rather than guessed.
- Model changes follow a controlled loop: candidate generation -> evidence review -> human confirmation -> modification of a session copy -> before/after validation.
- Computational outputs must include solver status, input and constraint audits, key numerical results, a reproducible notebook, structured tables, and provenance.

## 2. Current graphMet Capability Boundary

### 2.1 Capabilities that can be reused directly

- Agent orchestration: Supervisor, Reviewer, Retrieval, and Execution.
- GEM loading from SBML/XML and COBRA JSON.
- GEM tools: `load_gem_model`, `run_fba`, `run_fva`, `find_dead_end_metabolites`, `find_blocked_reactions`, `suggest_gap_fill`, and `gene_knockout_analysis`.
- Session-isolated custom analysis through `run_python`/Jupyter, with notebook and artifact output.
- Evidence retrieval from Neo4j, KEGG/external databases, PubMed abstracts, and available full text.
- Uploads for models, tables, and paper PDFs, plus human-in-the-loop requests when medium information or approval for a high-risk change is missing.

### 2.2 Capabilities requiring custom code or a dedicated wrapper

- pFBA, batch single/double knockout, yield calculation, growth-coupling classification, and FVSEOF.
- A unified model-quality report covering mass/charge balance, reference-model differences, and lost connectivity.
- Pathway insertion, export of modified SBML, and a standard before/after validation pipeline.
- A validated iBridge implementation with literature/reference-code consistency tests.

### 2.3 Inputs currently absent from the repository

- iML1515, Yeast8/yeast850, iJR904, and iCW773 models.
- The specific papers and reference implementations used by the PDF's yield, iBridge, and FVSEOF case studies.
- `excluded_metabolites.txt` and benchmark model variants containing known errors.

Task status is therefore defined as:

- **A - Natively executable:** a dedicated tool already exists; only light orchestration is needed.
- **B - Executable through the code sandbox:** `run_python` can perform the task, but Reviewer validation is required.
- **C - Conditional:** execution is blocked until a model, paper, reference implementation, or stable algorithm implementation is supplied.

## 3. Adapted Task Catalog

### A. Model Interpretation and Domain Knowledge

| ID | Task extracted from the PDF | graphMet-adapted task | Status | Ready-to-use graphMet prompt | Main deliverables |
|---|---|---|---|---|---|
| DK-01 | Summarize iML1515 and display its complete structure as a tree | Generate a model card for any session model, defaulting to ENGRO2. Report metadata, scale, compartments, objective, medium audit, and a readable compartment -> subsystem -> representative reaction hierarchy. Do not dump thousands of reactions into one tree. | A | **Prompt:** Read my uploaded GEM; if no model is attached, use `models/ENGRO2_annotated.xml`. Generate a model card containing model ID, organism, reaction/metabolite/gene/compartment counts, objective function, and medium audit. Produce a readable hierarchy organized as compartment -> subsystem -> representative reactions rather than expanding every reaction. Distinguish facts read directly from the model from inferences. Export Markdown and JSON. | Model-card Markdown/JSON and structural summary |
| DK-02 | List every reaction that uses acetyl-CoA as a substrate and summarize the pathways | Resolve the actual acetyl-CoA metabolite ID in each compartment, then select reactions with a negative stoichiometric coefficient. Add reversibility, subsystem, GPR, and provenance. | B | **Prompt:** Identify the acetyl-CoA metabolite IDs in every compartment of my uploaded GEM. List every reaction in which acetyl-CoA has a negative stoichiometric coefficient. Include reaction ID, name, equation, compartment, reversibility, subsystem, GPR, and coefficient. Summarize the reactions by pathway and explain their roles. Do not rely only on name matching, and do not include product-side occurrences as substrate reactions. Export CSV, a concise summary, and a reproducible notebook. | CSV, pathway summary, and notebook |
| DK-03 | Perform acetyl-CoA reaction identification using code | Merge the scientific task with DK-02 and treat executable code as a reproducibility criterion rather than a separate biological question. | B | **Prompt:** Use graphMet's code-execution environment to identify acetyl-CoA reactions. The code must discover compartment-specific acetyl-CoA metabolites from the model object rather than use a hard-coded reaction list. Select reactions in which acetyl-CoA is a substrate and export them to CSV. Run the code, check exceptions and empty results, report execution status and key findings first, and then provide the reproducible notebook and a short method explanation. | Executed notebook and CSV |
| DK-04 | Summarize reactions/pathways involving NAD+/NADH or NADP+/NADPH | Separate oxidized/reduced cofactors by compartment, report stoichiometry and subsystem, and avoid interpreting equation orientation as the sole physiological direction of reversible reactions. | B | **Prompt:** Analyze all reactions involving NAD+, NADH, NADP+, or NADPH in my uploaded GEM. Group them by compartment and subsystem. Report reaction ID, equation, reversibility, and the stoichiometric coefficients of all four cofactors. Summarize potential production and consumption pathways. For reversible reactions, do not treat the written equation direction as the only physiological direction. Export a reaction table, pathway statistics, and a notebook. | Cofactor reaction table and pathway statistics |
| DK-05 | Explain biomass-objective composition and compare WT versus core biomass | Parse the actual objective reaction in the selected model. If the model does not contain both WT and core biomass reactions, state that the comparison is not applicable and compare available objectives instead. | B | **Prompt:** Read the active objective of the current GEM, identify biomass-related reactions, and parse all biomass components, coefficients, and compartments. Explain the role of the biomass objective in FBA. Verify whether the model truly contains separate WT and core biomass reactions. If not, state that the comparison is not applicable; do not invent one. Instead compare the default biomass objective with any available maintenance or alternative core objective. Export a composition table and explanatory report. | Biomass composition table and report |
| DK-06 | Design an MVA pathway for limonene, select a host, and analyze bottlenecks | Treat this as evidence-driven pathway design. Retrieval gathers genes, enzymes, EC numbers, reactions, host evidence, and bottlenecks. Model computation begins only after a suitable microbial GEM is uploaded. | C | **Prompt:** Using literature and database evidence, design a pathway from acetyl-CoA to limonene through the mevalonate pathway. For each step, list substrates, products, reaction, enzyme, gene, EC number, source organism, and heterologous-expression requirement. Compare at least two suitable microbial hosts; analyze precursor, ATP/NADPH, toxicity, and regulatory bottlenecks; and propose yield-improvement strategies. Before model-based computation, verify that I have uploaded a suitable microbial GEM. Do not default to ENGRO2 or Human-GEM as a limonene production host. | Pathway table, host comparison, and evidence chain |
| DK-07 | Compare iML1515 and yeast850 using truncated JSON | Remove the truncated-JSON design. Require two complete uploaded models, extract statistics and pathway sets programmatically, and request the missing model if only one is available. | C | **Prompt:** Compare the two complete GEM files I uploaded. Programmatically extract model version, size, compartments, objective, medium, subsystem information, and reaction sets for each model. Compare shared and model-specific reactions/pathways and metabolic capabilities. If only one model is available, pause and request the second model. Do not reconstruct missing content from truncated JSON or memory. Export a comparison table, reproducible notebook, and limitations section. | Model-difference table and notebook |
| DK-08 | Compare iML1515 and yeast850 without model inputs | Keep this as a knowledge-only comparison, clearly separated from file-based analysis. Cite version-sensitive or quantitative claims. | A | **Prompt:** Without loading model files, compare iML1515 and Yeast8 from a domain-knowledge perspective. Cover organism, cellular compartments, central carbon metabolism, fermentation, amino-acid metabolism, lipid/sterol metabolism, and engineering applications. Clearly label this as a knowledge-based comparison rather than a model measurement. Cite version-dependent or quantitative claims and explain that they may change across releases. | Cited comparison report |
| DK-09 | Explain GSM applications, limitations, and future improvements | Use Retrieval to ground the discussion in literature, covering strain design, pathway optimization, high-value products, steady-state assumptions, missing regulation/kinetics, multi-omics, and enzyme-constrained models. | A | **Prompt:** Review the applications, limitations, and future development of genome-scale metabolic models in systems biology and metabolic engineering. Cover strain design, pathway optimization, and successful high-value chemical examples. Explain how steady-state assumptions, objective functions, network gaps, and missing regulation or kinetics affect predictions. Summarize multi-omics integration, enzyme-constrained models, regulatory/kinetic extensions, and AI-assisted refinement. Use reliable recent literature and separate established facts, opinions, and inferences. | Review-style response and references |
| DK-10 | Explain growth-coupling concepts, methods, relevance, limitations, and future directions | Provide operational definitions of strong and weak coupling and compare FVA, production envelopes, OptKnock, and related approaches. Keep concepts separate from model-specific calculations. | A | **Prompt:** Explain what it means for product formation to be growth-coupled and provide operational definitions of strong, weak, and uncoupled production. Compare FVA, production-envelope analysis, OptKnock, and related methods. Discuss engineering value, successful examples, discrepancies between model predictions and experiments, limitations, and future directions. Separate conceptual discussion from any model-specific result, and cite algorithmic and case-study claims. | Method-comparison table and cited explanation |

### B. Flux Prediction

| ID | Task extracted from the PDF | graphMet-adapted task | Status | Ready-to-use graphMet prompt | Acceptance focus |
|---|---|---|---|---|---|
| FP-01 | Run FBA on iML1515 and list the ten highest-flux reactions | Default to ENGRO2. Audit medium and objective first, ask for confirmation where necessary, rank by absolute flux while preserving signs, and separately flag boundary reactions. | A | **Prompt:** Run FBA on my uploaded GEM; if none is attached, use `models/ENGRO2_annotated.xml`. First show the default objective and medium audit. If the medium is empty or broadly open, pause and ask me to confirm it. After a successful solve, report solver status, objective value, and applied constraints. List the top ten reactions by absolute flux while preserving the sign, reaction name, and subsystem. Mark exchange, demand, and sink reactions separately. Export CSV, a notebook, and an audit JSON. | Optimal status plus objective, constraints, and top-ten results |
| FP-02 | Run FVA on iML1515 and find the ten redundant reactions with the largest range | Default to ENGRO2 and require an explicit `fraction_of_optimum`. Replace “redundant” with “most variable,” because a large range does not prove biological redundancy. | A | **Prompt:** Run FVA on my uploaded GEM; if none is attached, use `models/ENGRO2_annotated.xml`, with `fraction_of_optimum=0.9`. Confirm the objective and medium first. Separately identify blocked reactions, fixed reactions, and the ten reactions with the largest flux ranges. Report minimum, maximum, and range. Do not call high-range reactions redundant without additional evidence; interpret them using subsystem and alternative-pathway information. Export the full CSV, notebook, and constraint audit. | Correct minimum/maximum/range and non-misleading terminology |

### C. Metabolic Pathway Construction

| ID | Task extracted from the PDF | graphMet-adapted task | Status | Ready-to-use graphMet prompt | Acceptance focus |
|---|---|---|---|---|---|
| PC-01 | Identify gaps preventing limonene synthesis in iML1515 and suggest reactions/enzymes | Require an uploaded microbial host model. Check target, precursors, and product exchange first, then combine blocked/dead-end analysis, KG evidence, and literature. Do not write candidates into the model without review. | C | **Prompt:** Determine whether my uploaded microbial GEM can synthesize limonene. First verify the presence of limonene, IPP, DMAPP, GPP, and a product exchange reaction. Then run dead-end, blocked-reaction, and target-reachability analyses. Use the graphMet knowledge graph and literature to generate gap-fill candidates. For each candidate, provide equation, compartment, enzyme, gene, EC number, source, confidence, and risk. Propose candidates only; do not modify the model until I approve them. | Candidate equations, compartments, genes/enzymes, sources, and confidence |
| PC-02 | Compare limonene gaps, host suitability, and yield strategies in iML1515 and yeast850 | Require two complete models and harmonized media/objective definitions. Compute precursor reachability, theoretical yield, and bottlenecks before combining results with experimental evidence. | C | **Prompt:** Compare the two complete host GEMs I uploaded for limonene biosynthesis. Establish consistent, auditable media, carbon source, and product objectives for both models. Evaluate IPP/DMAPP/GPP reachability, missing reactions, theoretical yield, and bottlenecks in each model. Combine the computational results with experimental literature to assess host suitability and yield-improvement strategies. Separate model-derived results, literature evidence, and inference. If either model or essential condition is missing, pause and ask for it. | Comparable conditions and separation of model and literature conclusions |
| PC-03 | Insert the MVA-limonene pathway into iML1515 and run FBA | Work only on a session copy, discover existing IDs, validate mass/charge and stoichiometry, confirm the proposed changes, then export a new model and change manifest. | C | **Prompt:** Add the approved MVA-limonene pathway to a session copy of my uploaded model. First discover existing metabolite and reaction IDs programmatically to avoid duplicates. Validate compartments, elemental formulas, charges, directionality, and stoichiometry, then show the proposed additions and wait for my confirmation. After approval, add the product exchange/demand reaction, set the objective, and run before/after FBA. Export the new SBML/JSON, notebook, mass-balance report, and itemized change manifest. Never overwrite the original model. | Original preserved, no duplicate additions, and validated before/after FBA |

### D. Metabolic Flux Optimization

| ID | Task extracted from the PDF | graphMet-adapted task | Status | Ready-to-use graphMet prompt | Acceptance focus |
|---|---|---|---|---|---|
| FO-01 | Use FVSEOF on iML1515 to identify succinate up/down-regulation targets | Generalize FVSEOF to accept model, product reaction, medium, enforced production levels, and minimum growth. Run FVA at every level and measure robust trends in minimum and maximum flux. | B/C | **Prompt:** Run a general FVSEOF workflow on my uploaded GEM. Use `<product_reaction_id>` as the product reaction. Confirm the medium, substrate uptake, and minimum growth requirement first. Create enforced product levels from baseline to the maximum feasible value, run FVA at every level, and calculate correlations and slopes between enforced production and each reaction's minimum and maximum flux. Exclude blocked, constant, and boundary reactions. Report the top ten up- and down-regulation candidates, infeasible levels, statistics, and pathway interpretation. Save CSV, plots, and a notebook. | Filter blocked/constant reactions and report infeasible levels and statistics |
| FO-02 | Perform single knockout on iML1515_limonene while retaining 10% maximum biomass | Compute WT biomass in the same medium, set its 10% value as the lower bound, perform batch single-gene deletions, and report product flux, substrate-normalized yield, and growth changes. | B/C | **Prompt:** Perform genome-wide single-gene knockout analysis on my uploaded product-pathway model. First calculate the WT maximum biomass in the active medium, then constrain biomass to at least 10% of that value. Delete each gene and optimize `<product_reaction_id>`. Report status, biomass, product flux, change relative to WT, and molar yield normalized by actual substrate uptake. Filter infeasible and ineffective results and export ranked candidates, CSV, and a notebook. | The 10% constraint uses the same-medium WT; flux and yield remain distinct |
| FO-03 | Suggest and validate succinate knockouts in anaerobic iJR904 | Keep this as an external-model scenario. Discover and confirm oxygen, glucose, and succinate exchange IDs; then evaluate suggested knockouts under an auditable anaerobic configuration. | C | **Prompt:** Use my uploaded iJR904 model to analyze anaerobic succinate production. Programmatically identify the oxygen, glucose, and succinate exchange reactions and ask me to confirm them before applying an auditable anaerobic medium. Calculate succinate flux under biomass optimization, then optimize succinate while retaining at least 10% of WT biomass. Derive knockout candidates from literature or computation and test each one for biomass, product flux, and yield. Missing genes or reactions must produce explicit errors rather than being skipped silently. | Auditable anaerobic conditions and explicit handling of absent IDs |
| FO-04 | Determine whether limonene production in iML1515_limonene is growth-coupled | Use a production envelope over multiple biomass levels and classify strong, weak, or absent coupling. A single optimal solution is insufficient. | B/C | **Prompt:** Determine whether `<product_reaction_id>` is growth-coupled in my uploaded model. Confirm the medium, biomass reaction, and product reaction, then compute the maximum growth rate. Across multiple biomass levels from zero to maximum growth, constrain biomass and calculate minimum and maximum product flux to generate a production envelope. Use explicit thresholds to classify strong, weak, or absent coupling. Report the feasible region, numerical tolerances, and abnormal solver states. Do not base the conclusion on a single optimal solution. | Production-envelope plot, explicit thresholds, and feasible-region explanation |

### E. Literature-Driven Case-Study Validation

| ID | Task extracted from the PDF | graphMet-adapted task | Status | Ready-to-use graphMet prompt | Acceptance focus |
|---|---|---|---|---|---|
| CV-01 | Read and summarize a paper | Parse the attached PDF and report the research question, model, constraints, algorithm, results, limitations, and reproduction parameters. Cite the paper body rather than only the abstract. | A | **Prompt:** Read the paper PDF I uploaded. Produce a structured summary covering the research question, experimental/computational model, data, medium and constraints, algorithmic steps, key parameters, major findings, limitations, and reproducibility risks. Ground citations in the paper's main text, tables, or figures rather than only the abstract. Also produce a reproduction-parameter checklist and explicitly mark settings that are missing or ambiguous in the paper. | Structured summary, citations, and parameter checklist |
| CV-02 | Use a paper to calculate theoretical and maximum achievable L-lysine yield in E. coli | Require the paper and iML1515. Extract the paper's definitions of YT and YA and state whether normalization is molar, mass-based, or carbon-based. | C | **Prompt:** Using the paper and iML1515 model I uploaded, reproduce YT and YA for L-lysine production by E. coli from D-glucose under aerobic conditions. First extract the paper's YT/YA definitions, equations, units, medium, oxygen setting, and ATP-maintenance setting. If any definition is ambiguous, pause and ask me. Programmatically identify glucose, oxygen, lysine, and biomass reactions, run the calculation, and report paper values beside reproduced model values with differences and likely causes. Export formulas, audit JSON, CSV, and a notebook. | Explicit equations/units and side-by-side paper/model values |
| CV-03 | Calculate ethanol YT and YA for E. coli under aerobic, anaerobic, and microaerobic conditions | Extract the microaerobic oxygen setting from the paper. Keep the remaining medium identical across the three conditions and report biomass, product flux, substrate uptake, and yield. | C | **Prompt:** Using the paper and iML1515 model I uploaded, calculate ethanol YT and YA for E. coli under aerobic, anaerobic, and microaerobic conditions. Extract the exact microaerobic oxygen bound from the paper; do not invent one. Keep the medium and glucose setting identical across conditions except for oxygen. For each condition, report solver status, biomass, ethanol flux, actual glucose uptake, yield, equation, and units, then compare with the paper. Export a comparison table and notebook. | Comparable conditions; no guessed microaerobic value |
| CV-04 | Reproduce the same ethanol comparison in yeast850 | Apply the CV-03 design to Yeast8/850 while auditing model-specific medium, maintenance, and compartment assumptions. | C | **Prompt:** Using the paper and Yeast8/yeast850 model I uploaded, reproduce ethanol YT and YA for S. cerevisiae under aerobic, anaerobic, and microaerobic conditions. Extract and strictly apply the paper's oxygen settings. Audit model version, medium, maintenance, glucose/ethanol exchange reactions, and cytosolic/mitochondrial compartments. Report solver status, biomass, product flux, substrate uptake, yield, units, and differences from the paper for every condition. | Correct organism-specific settings |
| CV-05 | Add an R-mevalonate pathway to iCW773 and calculate theoretical yield | Require the model and pathway definition. Deduplicate internal metabolites, modify a copy, and record ATPM, glucose, oxygen, and other paper conditions as explicit experiment configuration. | C | **Prompt:** Use my uploaded iCW773 model and R-mevalonate pathway table to add the pathway to a session copy and calculate theoretical molar yield from D-glucose under aerobic conditions. Check existing internal metabolites and reactions first to avoid duplication, and validate compartment, element, charge, and stoichiometry. Record ATPM lower bound=0, glucose uptake=-10, O2=-1000, and all other settings as an explicit experiment configuration. Show and confirm the additions before modifying the copy. Export the new model, change manifest, FBA results, yield equation, and notebook. | No duplicate pathway entries, audited conditions, and reproducible molar yield |
| CV-06 | Implement iBridge using only the paper | Translate the paper into pseudocode, inputs/outputs, parameters, boundary cases, and stopping criteria before implementing a minimal version and testing it on a small model. graphMet has no native iBridge tool. | C | **Prompt:** Implement iBridge using only the paper I uploaded. First extract the algorithm definition, mathematical equations, inputs/outputs, exclusion rules, parameters, and stopping criteria into reviewable pseudocode. Clearly identify underspecified details and pause for confirmation. Then implement a minimal version in graphMet's code environment and test it with a small model and hand-constructed cases before using the uploaded target model. Export the implementation, tests, run log, candidate targets, and limitations. | Algorithm-fidelity tests, failure modes, and run log |
| CV-07 | Implement iBridge from the paper and GitHub code, applying excluded metabolites | Track the reference-code version or commit, compare it with the paper, parse exclusions, and validate that exclusions are actually applied. | C | **Prompt:** Implement and run iBridge using the paper, reference code, and `excluded_metabolites.txt` I uploaded. Record the reference-code version or commit. Compare the paper's description with the code's actual behavior, then validate excluded-metabolite ID mapping and filtering. On the same small model, compare graphMet and reference outputs under an explicit numerical tolerance. After consistency checks pass, analyze the target model and export provenance, tests, logs, and candidate tables. | Traceable sources and verified exclusion behavior |
| CV-08 | Implement FVSEOF using only the paper | Reuse the standard FO-01 interface after extracting enforced levels, objective, growth constraint, FVA settings, filters, and ranking rules from the paper. | B/C | **Prompt:** Reproduce FVSEOF using only the paper I uploaded. First extract the enforced product levels, objective function, minimum growth requirement, FVA settings, candidate filters, and ranking definition, and mark unreported parameters. After I confirm the reproduction configuration, run graphMet's standard FVSEOF workflow on the uploaded model. Report infeasible levels, reaction trends, candidate targets, comparison with paper results, and a reproducible notebook. | Complete reproduction parameters and repeatable output |
| CV-09 | Implement FVSEOF using the paper and GitHub code | Track the reference version and compare graphMet with the reference implementation on the same small model before running the target model. | C | **Prompt:** Reproduce FVSEOF using the paper and GitHub reference code I uploaded. Record the code version or commit and compare the paper and code with respect to enforced levels, FVA settings, correlation metrics, and ranking logic. Run graphMet and the reference implementation on the same small model, compare numerical values and rankings under explicit tolerances, explain discrepancies, and only then run the target model. Export configuration, comparison results, CSV, plots, and a notebook. | Numerical differences have explicit tolerances and explanations |

### F. Model Error Detection

Table S14 in the PDF uses several prompt strategies for the same underlying defects. For graphMet, these variants are consolidated into two reproducible benchmark tracks rather than treated as separate product features.

| ID | Original prompt variants | graphMet-adapted task | Status | Ready-to-use graphMet prompt | Passing criteria |
|---|---|---|---|---|---|
| QC-01 | Text-only analysis, generic code, COBRApy-specific code, text reference comparison, and code reference comparison for a stoichiometric sign error | Introduce a known sign error in a session copy of a benchmark model. Scan non-boundary/non-biomass reactions with `reaction.check_mass_balance()` and compare stoichiometric coefficients, direction, bounds, and GPR against a reference model. | B | **Prompt:** Compare the uploaded reference model and test model to detect stoichiometric-sign or mass/charge-balance errors. Analyze internal reactions only, excluding exchange, demand, sink, and biomass pseudo-reactions. Use COBRApy to scan mass and charge balance and compare metabolite coefficients, directionality, bounds, and GPR reaction by reaction. Report abnormal reaction ID, name, reference/test equations, coefficient differences, affected metabolites, and evidence type. Export a difference CSV and notebook. Do not modify either input model. | The altered reaction is correctly located, the benchmark remains unchanged, and false positives can be counted |
| QC-02 | Generic issue detection, missing-reaction question, generic code, guided glycolysis check, code reference comparison, and text reference comparison | Remove a known internal reaction from a benchmark copy. Compare reaction sets and connectivity, blocked reactions, and target reachability. Without a reference, use a pathway checklist and KG/literature as weaker evidence. | B | **Prompt:** Compare the uploaded reference model and test model to identify missing internal reactions or lost metabolic connectivity. Programmatically compare reaction sets, equations, bounds, and GPRs, then assess affected metabolites, blocked reactions, key-pathway completeness, and target reachability. Exclude exchange, demand, sink, and biomass pseudo-reactions. Report each missing reaction's ID, name, reference equation, affected metabolites, functional consequence, and evidence strength. If no reference model is supplied, clearly label the lower confidence and cross-check against a predefined pathway list, the knowledge graph, and literature. | Missing reaction ID/name/equation and affected metabolites are found, with the functional consequence explained |

The Table S14 prompt variants should remain as experimental dimensions:

1. No tools, generic prompt;
2. Code allowed, generic prompt;
3. COBRApy or a dedicated tool explicitly requested;
4. Structured analysis steps supplied;
5. Reference model supplied for programmatic comparison.

This design measures the improvement produced by better instructions and tools instead of treating five phrasings as five independent product capabilities.

## 4. Recommended Unified Task Template

Every graphMet task should use a structured input similar to the following:

```yaml
task_id: FP-01
model:
  source: session_attachment
  path_or_artifact_id: <required>
analysis:
  objective_reaction: <explicit-or-confirm>
  medium_constraints: <explicit-or-confirm>
  organism: <optional-but-recommended>
  solver_tolerance: 1e-9
outputs:
  - summary_markdown
  - result_table_csv
  - reproducible_notebook
  - audit_json
validation:
  require_optimal_status: true
  require_reviewer: true
```

Model-modification tasks should additionally require:

```yaml
modification_policy:
  edit_session_copy_only: true
  require_human_confirmation: true
  validate_mass_charge_balance: true
  compare_before_after: true
  export_change_manifest: true
```

## 5. Recommended Implementation Order

### Phase 1: Benchmarks that can be implemented immediately

- DK-01, DK-02, DK-04, and DK-05;
- FP-01 and FP-02;
- QC-01 and QC-02;
- CV-01.

These tasks cover model loading, code execution, Reviewer behavior, attachment parsing, notebooks, and structured artifacts. Most can use the existing ENGRO2 or e_coli_core models.

### Phase 2: Standardize reusable algorithms

- FO-01 FVSEOF;
- FO-02 batch single-gene deletion;
- FO-04 production-envelope/growth-coupling analysis;
- Unified yield calculation and model-quality reporting.

These algorithms should be promoted from ad hoc notebook code into tested graphMet tools, each with small-model unit tests.

### Phase 3: Prepare external benchmark assets

- Obtain version-pinned iML1515, Yeast8, iJR904, and iCW773 models.
- Record each model's source, download date, checksum, and license.
- Prepare target-reaction and medium configurations for limonene, succinate, ethanol, and L-lysine.
- Construct case1/case2 error models only as session copies of benchmark models.

### Phase 4: Reproduce literature algorithms

- CV-02 through CV-09;
- Prioritize FVSEOF before iBridge;
- Preserve paper parameters, reference-code version, numerical comparison, and failure cases for every implementation.

## 6. Unified Acceptance and Scoring Rules

The PDF uses separate 1-5 rubrics for domain and coding tasks. For graphMet, a unified 100-point rubric is recommended:

| Dimension | Weight | Highest-standard description |
|---|---:|---|
| Computational correctness | 25 | Solving, filtering, ranking, formulas, and units are correct; key results can be independently recomputed |
| Reproducibility | 20 | Input versions, constraints, random seeds, code, notebooks, and artifacts are complete |
| Biological validity | 15 | Organism, compartment, directionality, GPR, medium, and objective interpretation are correct |
| Completeness | 10 | Every sub-question and required output is covered |
| Robustness and error handling | 10 | Missing files, absent IDs, infeasible models, and non-optimal solver states are handled explicitly |
| Evidence and provenance | 10 | Model facts, KG records, literature evidence, and inference are clearly separated and traceable |
| Clarity and concision | 5 | The result is readable and table-oriented, without dumping unmanageable raw objects |
| Modification safety | 5 | Only a session copy is edited, risky steps require confirmation, and a change manifest is exported |

Recommended passing score: 80/100, with three hard-failure rules:

- A computational task that omits solver status fails automatically.
- A medium- or objective-dependent conclusion that does not record the applied constraints fails automatically.
- A task that modifies the original benchmark model or omits a change manifest fails automatically.

## 7. Additional Ready-to-Use graphMet Prompt Examples

### Example 1: FBA

> Run FBA on my uploaded GEM. First identify and display the default objective and medium audit. If the medium is empty or broadly open, pause and ask me to confirm it. After a successful solve, report objective value and the top ten reactions by absolute flux while preserving flux signs, reaction names, and subsystems. Save the complete result as CSV and a reproducible notebook.

### Example 2: FVA

> Run FVA on my uploaded GEM with `fraction_of_optimum=0.9`. Separately identify blocked, fixed, and the ten highest-range reactions. Do not equate a large range with biological redundancy. Interpret the result using subsystem and alternative-pathway evidence, and export CSV, notebook, and constraint audit.

### Example 3: Missing-reaction detection

> Compare my uploaded reference and test models, analyzing internal reactions only. Programmatically compare reaction sets, stoichiometric coefficients, bounds, and GPRs, then assess lost connectivity and blocked reactions. Report missing reaction ID, name, equation, affected metabolites, and functional consequences. Keep programmatic facts separate from biological inference.

### Example 4: Safe gap filling

> Analyze target-associated dead ends and blocked reactions in my uploaded model. Use the graphMet knowledge graph and literature to generate gap-fill candidates. First provide only a candidate table containing equation, compartment, enzyme/gene, source, confidence, and risk; do not alter the model without my approval. After confirmation, modify a session copy only and provide before/after FBA, mass/charge balance, target reachability, the modified SBML, and a change manifest.

## 8. Most Important Revisions Relative to the Source PDF

1. Replaced embedded full-model JSON with file/attachment-driven inputs compatible with graphMet sessions and artifacts.
2. Replaced “provide code” with “execute, validate, and save a notebook”; runnable analysis matters more than a syntactically complete code block.
3. Replaced ambiguous terms such as “top flux,” “redundant reaction,” and “theoretical yield” with testable definitions, sorting rules, formulas, and units.
4. Converted model modification and gap filling into reviewed, reversible, and auditable workflows.
5. Marked absent microbial models and algorithms as conditional tasks rather than misusing ENGRO2 or Human-GEM.
6. Consolidated Table S14's repeated prompt variants into two scientific tasks while retaining prompt/tool conditions as benchmark dimensions.
