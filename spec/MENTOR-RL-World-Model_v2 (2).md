## **MENTOR-RL Foundation Model: Building a Multiplex World Model From a Lattice of Networks**

### **Purpose**

This plan describes how we build the MENTOR-RL foundation model as a systems-biology world model rather than a graph lookup table. We train a single policy to internalize the operators of multiplex networks by exposing it to a structured population of multiplexes built by one construction rule. For a context we take its selected PENs, form their gene union, induce HumanNet and the other non-PEN layers on that union, and stack the layers, so the multiplexes differ by cell type, tissue, region, and organ system. Specific graph content reaches the model through context and deterministic tools at query time. The weights hold the rules, not the facts. Three structural views carry equal weight in training, the bottom-up RWR-LOE neighborhoods, the top-down MENTOR-EV hierarchy, and the underlying network connectivity that the hierarchy hides. This document sets out the rationale, the workstreams, the training and evaluation approach, and the open decisions.

### **Scientific rationale**

Sequence models trained on synthetic tasks form internal, causal models of the process behind their data, and those models generalize to states never seen in training, because the generating rules are compact and recur across many examples while the specific data does not. Two lessons carry over. A model can learn the operators that regenerate structure, but it cannot memorize irreducible content that follows no compact rule, so biological edge weights, walk scores, and distances belong in the runtime, not the weights. And emergence of a world model depends on dense sampling of the generating process, which a single graph with fixed templates cannot supply.

The lattice supplies that sampling, and the choice of target follows directly. We expect the MENTOR-EV trees to differ strongly across contexts, with distinct clades emerging in different cell types, tissues, and regions. That divergence is the biology, not an artifact, and it fits the world-model frame cleanly. The invariant we train on is not the trees. It is the operator that maps a multiplex to a tree, and the way layer composition drives it. Board states differ in every game while the rule that generates them stays fixed, and here the hierarchies differ in every context while the operator that generates them stays fixed. A model that learned the operator will correctly produce very different trees for very different contexts, and getting the divergence right is a stronger result than getting conservation right.

The dendrogram is a lossy abstraction of the multiplex, so it cannot be the world model on its own. Two clades that sit far apart in the tree can be joined by short, biologically supported paths and bridge nodes in the real network, which is why signals such as a drug-target enrichment can appear in several distant branches yet act through one connected system. The model therefore has to hold the network and reason about the cross-clade connectivity the tree suppresses. The hierarchy is the index used to find and prioritize. The network is where hypotheses are confirmed.

### **The multiplex lattice**

Every multiplex is built by one rule. The selected PENs for a context define its gene universe, and HumanNet and the other non-PEN layers are induced on that universe before all layers are stacked. So the backbone is not shared across the lattice. It is an induced subgraph whose node set and edges follow the PEN selection. The family is nested and compositional. The finest grain is one cell type in one tissue in one region. Combining cell types gives a tissue-level multiplex in a region, combining tissues gives a region-level multiplex, and combining regions builds up to the whole-brain multiplex we already hold. The same construction extends to other organ systems and to multiplexes that join organ systems, up to a whole-body multiplex. The disease-relevant networks, including the anxiety, suicide, Pan-SUD, and cardiac projections, are points in this same lattice, which is the basis for transfer. The Context Ontology section below formalizes this lattice as a grounded partial order over standard ontology terms, giving every multiplex a canonical address and defining the composition described here as the join over that order.

### **Context Ontology: Representing and Navigating the Lattice**

The lattice is represented as a grounded ontology so that every multiplex has a canonical, interoperable address and the composition operator has a defined domain. We build on the OBO, CZI, and HuBMAP consensus rather than a bespoke scheme, so the lattice inherits versioning, cross-species bridges, and alignment to the public single-cell corpora that also serve as data sources.

#### **The ontology stack**

Each context is placed on two axes, anatomy and cell type, with an optional developmental stage, and each axis is grounded in a standard ontology.

| Axis | Primary ontology | Granular or provisional layer | Role |
| ----- | ----- | ----- | ----- |
| Anatomy and regions | UBERON | BICAN Human Brain Atlas Ontology, the Allen HBA StructureGraph mapped to UBERON, and DHBA for developmental contexts | Species-neutral interoperable backbone with cross-species bridges, plus granular human brain regions that resolve to UBERON |
| Cell types | Cell Ontology (CL) | Provisional Cell Ontology (PCL), which carries the BICCN and Siletti transcriptomic brain types through the Brain Data Standards work | Canonical cell types, extended by the transcriptomic brain types CL does not yet hold |
| Anatomy and cell-type linkage | HuBMAP ASCT+B, part of the Human Reference Atlas | none | part\_of links that tie anatomical structures to their resident cell types, maintained across the whole body |
| Developmental stage | HsapDv | none | The stage of a context when it matters |
| Metadata convention | CELLxGENE schema, UBERON tissue, CL cell type, HsapDv stage, EFO assay | none | The de facto single-cell annotation standard, aligning nodes to CELLxGENE Census |

For the Mechanistic Evolution line, UBERON's bridges to the Allen mouse atlas carry anatomy across taxa, while gene orthology stays with the sequence pipeline. The same four axes extend to the whole body without changing the stack, because the Human Reference Atlas is a whole-body construction, so moving the lattice from brain to other organ systems populates more of the same ontologies rather than replacing them.

#### **Tying the ontologies together to represent the lattice**

A context is a node addressed by an anatomy term, a cell-type term, and a developmental stage, and it carries the PEN that defines its gene universe together with the raw taxonomy cluster identifier that produced that PEN. The lattice is a partial order rather than a tree, because a cell type recurs across regions and a combination joins several parents. The composition operator is the join in that partial order. It takes anatomy up the ASCT+B and Human Brain Atlas `part_of` hierarchy and cell types up the transcriptomic taxonomy from subcluster to cluster to supercluster, and the resulting node's gene universe is the union of the parts' PEN gene sets with the non-PEN layers re-induced on it, as S8 specifies.

Each node stores two identities, the ontology term for interoperability and the cluster identifier for provenance. Both are version-pinned in the provenance block beside `graph_version` and `flist_hash`, and an explicit mapping table from PEN-derived types to taxonomy identifiers keeps node identity stable when a taxonomy is re-versioned.

json

// a context node, identifiers illustrative

{

  "context\_id": "brain:hippocampus:CA1:astrocyte:adult:v1",

  "anatomy":   { "uberon": "UBERON:0003881", "hba": "HBA:CA1", "label": "CA1 field of hippocampus" },

  "cell\_type": { "cl": "CL:0000127", "pcl": "PCL:00170xx",

                 "taxonomy\_cluster\_id": "siletti:cluster\_0123", "label": "astrocyte" },

  "developmental\_stage": { "hsapdv": "HsapDv:0000087", "label": "adult" },

  "pen\_layers": \["scPEN:brain:CA1:astrocyte:v1"\],

  "provenance": { "ontology\_versions": { "uberon": "2025-xx", "cl": "2025-xx",

                                          "pcl": "2024-01", "hba": "vX" },

                  "taxonomy\_version": "siletti\_2023" }

}

#### **How the model uses the ontology**

The ontology is supplied metadata and the domain of the composition operator, not content baked into the weights. The model reads a provided ontology and navigates it, so the specific brain ontology is context, and the same navigation transfers when the lattice extends to other organ systems. The multiplex identifier family S1.5 resolves a node to its ontology terms rather than parsing a flat string, and a new context-navigation family teaches the model to move over the partial order, returning a node's parents, children, and siblings, the join of two nodes that names the union multiplex, and the ontology distance between two contexts.

json

// book\_mode: open\_book, question\_family: ctx\_ontology\_navigation

"context": { "ontology": "\<provided partial order over context nodes\>",

             "query\_node": "brain:hippocampus:CA1:astrocyte:adult:v1" }

"answer": {

  "node\_id": "brain:hippocampus:CA1:astrocyte:adult:v1",

  "parents": \["brain:hippocampus:CA1:all\_celltypes:adult:v1",

              "brain:hippocampus:astrocyte:adult:v1"\],

  "children": \[\],

  "siblings": \["brain:hippocampus:CA1:neuron:adult:v1",

               "brain:hippocampus:CA1:microglia:adult:v1"\]

}

json

// join of two nodes is the union multiplex

"context": { "node\_a": "brain:hippocampus:CA1:astrocyte:adult:v1",

             "node\_b": "brain:hippocampus:CA1:neuron:adult:v1" }

"answer": { "join\_node\_id": "brain:hippocampus:CA1:astrocyte+neuron:adult:v1",

            "gene\_universe\_rule": "union\_of\_part\_pen\_gene\_sets\_then\_reinduce\_non\_pen\_layers",

            "ontology\_distance": 2 }

The two parents in the first example show why the structure is a partial order rather than a tree, since a single cell type in a region sits under both the all-cell-types node for that region and the same cell type across the parent structure. This navigation drives three pieces already in the plan. The WS1 construction curriculum populates leaf nodes first and composes up the order. The held-out context flagship reserves whole ontology nodes or subtrees, which is a cleaner split than an ad hoc holdout at the combination level. And the divergence characterization in S8.7 is indexed by ontology distance, so the model is asked how strongly sibling cell types diverge against how strongly a child diverges from its parent. Navigation and join are relational operators the model learns, while the graph content and every statistic still come from the runtime tools.

### **Design principles**

The invariant is the construction operator, not a fixed backbone. The selected PENs define the gene universe, and HumanNet and the other non-PEN layers are induced on it, so the backbone is context-specific in both its nodes and its edges. What stays constant across the lattice is the rule that builds each multiplex, and that rule is what the model learns. Content flows through the runtime, since what varies across worlds is which layers are active, a compact and tokenizable descriptor, while the heavy edge and score content is fetched by RWR++ tools. Encodings are relational, because seed-relative ranks, distance percentiles, containment fractions, and clade depths are represented more cleanly than raw values. Minimal pairs are first-class, but they are compound. Adding or removing a PEN layer changes the gene universe and re-induces the non-PEN layers, so a pair differs by that layer and by its induced backbone together. The isolated single-layer effect stays in the ablation families, and a fixed-gene-universe intervention is used only as a controlled probe in evaluation.

### **Workstreams (WS)**

***WS0, divergence validation.*** Before the full build, confirm that different PEN selections produce divergent multiplexes and trees. Compare RWR-LOE neighborhoods, MENTOR-EV trees, and cross-clade connectivity across a first set of contexts, since each context induces its own backbone on the PEN-defined gene universe. Strong divergence is the expected and desired result, and it is the precondition for everything downstream.

***WS1, lattice construction, runtime, and ground truth.*** Build tier-one single-context multiplexes with the RWR++ native runtime. For each lattice point, run MENTOR to produce a dendrogram and run RWR-LOE per seed, and materialize shortest paths, connector and bridge nodes, and layer-specific support. Generate questions procedurally across the family, not from fixed templates on one graph, and cache every target with its layer descriptor, metric, seed set, and graph version.

***WS2, RWR-LOE operators, bottom-up.*** Teach seed-centered neighborhood construction, the geometric elbow that defines a module, rank and distance comparison, and set relations among neighborhoods. These recover the short cross-clade routes the tree cannot show, so they are complementary to the hierarchy rather than redundant.

***WS3, MENTOR-EV hierarchical operators, top-down, promoted to its own pillar.*** Teach tree navigation as operators over a per-context dendrogram, lowest common ancestor of a gene pair, the subtree induced by a cut at a given height, parent, child, and sibling relations, subtree membership, and clade cohesion. Targets are alignment-free and defined inside a single tree, so they compare across contexts by value without ever aligning trees. Do two genes share a clade here. What is the depth of their lowest common ancestor. Is this gene set cohesive at this cut height. Cut height doubles as a difficulty dial, coarse clades first, fine clades later.

***WS4, network-connectivity reasoning, promoted to its own pillar.*** Teach the model to return from the hierarchy to the network, finding connector and bridge nodes, recovering short paths that cross clade boundaries, decomposing paths by supporting layer, and stating layer-specific support and its limits. This is the workstream that guards against reading the dendrogram as the whole truth, and it carries the calibration rules that keep the model from claiming a direct interaction where only a cross-layer route exists.

***WS5, SFT on operators and composition.*** Closed-book training covers only bounded, stable conventions, meaning identifiers, schema, layer families, and distance-direction rules. Open-book training covers every parameter-dependent value, presented as context to read and compare. Statistics come from tools, not the forward pass. Teach the composition operator directly, so a larger multiplex is understood as the union of the parts' gene universes with the non-PEN layers re-induced on that union. Scale difficulty along two axes, the lattice tier and the cut-height granularity.

***WS6, probe and intervention evaluation.*** Train probes on the model's activations to recover graph properties it was never given directly, edge presence, layer membership, clade co-membership, and connector status, then intervene on those activations to test whether the recovered structure is causal for the model's answers. This turns the world-model claim into a measurable hypothesis.

***WS7, flagship evaluations.*** Two named tests define success and are described in the next section.

***WS8, handoff.*** The foundation checkpoint feeds the existing DPO and GRPO agent pipeline once foundation metrics are stable.

### **Flagship evaluations**

Held-out context tree generation. Train on single contexts and some combinations, then require the model to produce the correct, context-specific hierarchy for a lattice point it never saw, measured by clade recovery, correct lowest common ancestor prediction, and correct parent, child, and sibling relations, and confirmed by probing. Because the trees diverge, this cannot be met by interpolating trees the model was shown, so it tests whether the model learned the operator rather than memorized outputs.

Projection and return to network. Reproduce the applied workflow as a single, explicit operation. The projection runs in two steps. First, seed RWR-LOE from a target gene, for example a drug target such as GLP1R, on the chosen multiplex, which returns the target's network neighborhood, meaning the genes well connected to it by propagation across all layers. Second, take that neighborhood gene set and test it for enrichment against the MENTOR-EV modules at every level of the dendrogram, so each clade receives an enrichment score for the target's RWR-LOE neighbors. The return to network then requires the model to interpret the projection correctly. Because the neighborhood was seeded on the network, enrichment that lands in several distant clades is evidence of one connected system, not several separate findings, and the model must recover the connector and bridge nodes and the short cross-clade paths that tie those clades together, and report the layer-specific support behind them.

We score three things. The projection itself, meaning whether the model reproduces the per-clade enrichment of the seed's RWR-LOE neighborhood across the hierarchy. The connectivity recovery, meaning whether the model retrieves the short cross-clade paths and bridge nodes that justify treating distant enriched clades as one system. And the interpretation, meaning whether the model reads cross-clade enrichment as recovered connectivity rather than dispersion, and refuses the false reading that distance in the tree implies separation in the network. This mirrors the anxiety, Pan-SUD, and diabetes-proteomics repurposing use, where a target's mechanistic reach must be read from the network that generated the projection, not from the tree that fragments it. A model that scores well here has learned the relationship the GLP1R figure demonstrates, that a single RWR-LOE neighborhood can distribute its enrichment across distant clades, and that the distribution is connectivity.

### **Data and ground truth**

Ground truth scales with the family. MENTOR runs per lattice point for the dendrograms, RWR-LOE runs per seed, and RWR++ supplies paths, bridges, and layer support. Questions are generated procedurally so the model sees each operator applied to many graph instances. A construction curriculum prioritizes single contexts, then chosen pairs, then selected higher-order unions, with deliberate coverage of sparse cell-type layers.

### **Risks and mitigations**

Per-graph scale is unchanged, so we keep content in the runtime and train the weights on operators and composition. Backbone dominance is a smaller risk than it first appeared, because the backbone is induced on the PEN-defined gene universe rather than shared, so it cannot impose a fixed global structure, and WS0 confirms the expected divergence before the full build. The combination space is far too large to sample uniformly, so the construction curriculum is required rather than optional. Leakage sits at the combination level, because the gene universe and the underlying HumanNet edges are shared to the extent contexts share genes, so splits are defined over layer combinations and over the specific seed, target, and combination tuples used in questions, with overlap reported at that level. Over-reliance on the dendrogram is countered by WS4, which makes network connectivity a first-class target. The scientific payoff turns on generalization to unseen contexts rather than reproduction of seen trees, so the held-out context tree generation benchmark is the number we hold ourselves to.

### **Milestones**

| Phase | Focus | Output | Gate to advance |
| :---- | :---- | :---- | :---- |
| 0 | Divergence validation (WS0) | RWR-LOE, tree, and connectivity compared with and without context layers | Context layers produce strong divergence in trees and connectivity |
| 1 | Lattice tier 1, runtime, ground truth (WS1) | Single-context multiplexes with dendrograms, RWR-LOE, paths, bridges | Reproducible per-graph targets across the first contexts |
| 2 | Bottom-up and top-down operators (WS2, WS3) | SFT on RWR-LOE neighborhoods and hierarchy navigation | Held-out phrasing and held-out region accuracy per family |
| 3 | Network connectivity (WS4) | SFT on connectors, cross-clade paths, layer support | Correct cross-clade path and bridge recovery |
| 4 | Composition, tiers 2 to 3 (WS5) | Pairwise and higher-order unions, composition operator | Correct answers on unions built from seen parts |
| 5 | Probe and intervention eval (WS6) | World-model probes and causal interventions | Recovered structure is causal, not decorative |
| 6 | Flagship evaluations (WS7) | Held-out context tree generation and projection-and-return-to-network | Both hold on unseen contexts |
| 7 | Handoff (WS8) | Foundation checkpoint feeds DPO and GRPO | Foundation metrics stable across stages |

### **Open decisions:**

•          Which contexts seed tier one, and the order in which we extend from brain regions to other organ systems.

•          How we sample the combination space so coverage stays balanced and sparse cell-type layers are well represented.

•          The split policy at the combination and tuple level, and the overlap threshold we accept.

•          The compute budget for procedural ground-truth generation across the family, given the cost of running MENTOR and RWR-LOE per lattice point.

•          The direction and translation filters we encode for the projection-and-return-to-network case, so the applied repurposing logic is represented faithfully in evaluation.

## **MENTOR-RL Training Curriculum: From a Multiplex World Model to a Mechanistic Agent**

### **Purpose and principles**

This curriculum trains MENTOR-RL in four parts, a supervised stage that builds the multiplex world model, a preference stage that refines interpretation, a policy-gradient stage that extends long-horizon exploration, and a handoff that turns the foundation model into the tool-using agent. It realizes the plan we agreed on, so it rests on a small set of principles that govern every stage.

The model learns operators, not facts. The rules that regenerate structure live in the weights, and irreducible content, edges, weights, walk scores, and distances, reaches the model through context and deterministic tools. Encodings are relational, so ranks, percentiles, containment fractions, and clade depths are preferred over raw values. Targets are checkable, so every answer has a deterministic ground truth produced by the RWR++ runtime, and statistics are computed by tools rather than in the forward pass. Questions are generated procedurally across the lattice, so the model sees each operator applied to many graphs and cannot win by memorizing one. Trees are expected to diverge across contexts, so hierarchy targets are alignment-free and defined inside a single tree. The dendrogram is a lossy abstraction, so network connectivity is a co-equal pillar, and cross-clade paths and bridge nodes are taught as first-class structure. Minimal pairs recur throughout, though they are compound, since adding or removing a PEN layer changes the gene universe and re-induces the non-PEN layers, so a pair differs by that layer and its induced backbone together.

### **Difficulty axes**

Six axes control difficulty, and stages advance along them rather than through a single linear sequence.

The lattice tier runs from a single cell type in a tissue in a region, up through tissue, region, and whole brain, then to cross-region, other organ systems, and cross-organ combinations. Operator complexity runs from identity and schema, through atomic graph facts, paths, RWR-LOE neighborhoods, hierarchy navigation, cross-clade connectivity, the projection operation, and finally composition. Book mode runs from closed-book conventions, through open-book reading and comparison, to tool-call construction and parsing. Cut-height granularity runs from coarse, well separated clades to fine, easily confused ones. Distractor difficulty runs from far bands to near bands. Horizon length, used in the reinforcement stages, runs from short budgets to long multi-hop exploration.

### **Part I. Supervised fine-tuning: the multiplex world model**

The supervised stage has nine steps. Closed-book training appears only where the target is a bounded, stable convention. Everything parameter-dependent is open-book, presented as context to read and compare, and tool-call training is layered in from the RWR-LOE step onward and consolidated at the end.

Step S1, coordinate system and conventions, closed-book, single context. The model learns the canonical identifier space with Ensembl identifiers as graph keys and symbols as aliases, parsing of layer tags, layer-family classification, multiplex and module identifier parsing, and the direction conventions for rank, score, and distance. It also learns the calibration vocabulary, meaning the allowed and disallowed claims that recur later. These targets are small and stable, so memorizing them is correct. Gate, near-perfect accuracy on held-out identifiers and tags.

Step S2, atomic graph facts, open-book, single context. The model reads provided tables to answer edge existence per layer and across the multiplex, direct neighbors as a neighbor-to-layer map, degree and hubness, layer membership, nodes present in a layer, shared neighbors, induced subgraphs, and connected components. Calibration negatives enter here and stay, covering absent edges, top-k truncation, hub bias, and layer specificity, each paired with the claim it does and does not license. Gate, accuracy per family on held-out graph regions and held-out phrasings, and correct refusal on the calibration negatives.

Step S3, paths and cross-layer routes, open-book, single context then pairwise. The model computes shortest paths in a single layer and across the multiplex, decomposes a path by supporting layer, contrasts a missing monoplex path with an existing multiplex path, and identifies connector and bridge nodes. This seeds the connectivity pillar. Gate, path and bridge recovery on held-out regions.

Step S4, RWR-LOE neighborhoods, the bottom-up pillar, open-book with tool-call introduced, single context to tissue. The model works the seed-centered neighborhood, rank and score lookup, comparison of two genes in a rank vector, closest-k retrieval, query-gene filtering, the geometric elbow that defines a module, elbow reasoning, leave-one-out support, vector intersection and Jaccard, distance-matrix shard lookup and comparison, distance-percentile calibration, rank-vector against distance-matrix consistency, layer ablation, and seed essentiality. Difficulty rises through distractor bands drawn from post-elbow rank. Gate, recovery and comparison accuracy stratified by distractor band.

Step S5, MENTOR-EV hierarchy navigation, the top-down pillar, open-book, single context to region. On the per-context dendrogram the model learns clade membership, lowest common ancestor of a gene pair, the subtree induced by a cut at a given height, parent, child, and sibling relations, subtree membership, and within-clade cohesion. All targets are alignment-free and read from one tree. Cross-source relations are taught here too, RWR-LOE against MENTOR-EV containment, best-matching module, subset and superset with violating genes, set difference, intersection, Jaccard, and containment coefficients, which teach the correspondence between the bottom-up and top-down views and reward consistency between them. Cut height is the difficulty dial, coarse clades first. Gate, hierarchy-navigation accuracy across cut heights and correct cross-source containment.

Step S6, cross-clade connectivity, the network pillar proper, open-book, single context to whole brain. This step teaches that distance in the tree is not distance in the network. Given two clades far apart in the dendrogram, the model finds the short cross-clade paths and bridge nodes that join them, decomposes them by layer, states the layer-specific support and its limits, and refuses the reading that separation in the tree implies separation in the network. Global cohesion against a null enters here, within-clade distance against random sets, clustering ratio, density, conductance, and cell-type-specific and layer-sensitive cohesion. Gate, cross-clade path recovery and correct rejection of the false-separation reading.

Step S7, the projection operation, open-book and tool-call, whole brain then disease multiplexes. The model learns the two-step operation the GLP1R figure demonstrates. First, seed RWR-LOE from a target gene to obtain its network neighborhood. Second, test that neighborhood for enrichment against the MENTOR-EV modules at every level of the dendrogram, producing a per-clade enrichment. It then returns to the network, reading enrichment that lands in several distant clades as evidence of one connected system, and recovering the connectors, short paths, and layer support that justify it. Directional and translational filtering enter here, mapping drug and disease evidence to the same or nearby modules, opposing the disease-associated direction, and weighing evidence strength, adverse-event risk, brain penetration, feasibility, and testability. Gate, correct projection, connectivity recovery, and interpretation, described in the evaluation section as a flagship.

Step S8, composition and lattice scaling, open-book and tool-call, pairwise to cross-organ. The model learns to predict how neighborhoods, clades, and connectivity change when context layers are added or removed, treating a larger multiplex as the union of the smaller ones' gene universes with the non-PEN layers re-induced on that union and the PEN layers stacked. Compound minimal pairs, adding or removing a PEN with the backbone re-induced, anchor this step. The target is generalization to combinations never built. Gate, correct structure on unions assembled from seen parts, leading into the held-out context flagship.

Step S9, tool-call consolidation, tool-call mode, all tiers. The model consolidates schema-valid tool construction with biological arguments rather than file paths, tool selection, result parsing, provenance answers, refusal of raw command-line arguments, and evidence-grounded structured-state updates. Statistics are delegated to tools throughout. This prepares trajectory generation for the preference stage. Gate, schema-valid tool-call rate and correct parsing on held-out tools and phrasings.

A provisional training mix across the pillars follows, to be reset after WS0.

| Pillar or family | Share of SFT data (tunable) | Concentrated in |
| :---- | :---- | :---- |
| Conventions and identity | 8 percent | S1 |
| Atomic graph facts | 14 percent | S2 |
| Paths and cross-layer routes | 8 percent | S3 |
| RWR-LOE neighborhoods, bottom-up | 16 percent | S4 |
| MENTOR-EV hierarchy, top-down | 16 percent | S5 |
| Cross-clade connectivity, network pillar | 12 percent | S6 |
| Projection operation | 12 percent | S7 |
| Composition and lattice scaling | 8 percent | S8 |
| Tool-call construction and parsing | 6 percent | S9 and threaded |

Book-mode progression runs across the steps, closed-book confined to S1, open-book dominant from S2 through S8, and tool-call introduced at S4 and consolidated at S9. The three structural pillars, RWR-LOE, MENTOR-EV, and network connectivity, carry equal weight by design, and the projection step is where they are exercised together.

### **Part II. Direct preference optimization: refining interpretation**

The preference stage trains on shared-prefix trajectory branches generated by the SFT checkpoint over the operators it now holds. At each decision point the policy proposes several next steps, the deterministic runtime executes valid tool calls, and each branch receives a local score over schema compliance, the change in module-membership accuracy, the change in evidence-grounded mechanistic quality, and an efficiency penalty on length and invalid calls. The selected branch becomes the preferred example, and dispreferred examples are drawn across easy, medium, and hard bins.

The curriculum scales difficulty in the order the SFT steps established. It begins with recovery and refinement over RWR-LOE neighborhoods and hierarchy clades, where the runtime gives dense signal, then adds cross-clade connectivity trajectories, then the projection and return-to-network trajectories, which are the longest and most informative. Difficulty rises through the lattice tier, the cut-height granularity, and the distractor band, and preference pairs move from easy to medium to hard as each level saturates. Preference data is regenerated from improving checkpoints so the model keeps learning past the first corpus. Gate, monotone improvement in module recovery and mechanistic grounding over the SFT checkpoint on the held-out splits.

### **Part III. Group-relative policy optimization: long-horizon exploration**

The policy-gradient stage optimizes complete rollouts against a terminal reward that combines schema validity, absolute and cumulative module recovery, absolute and cumulative mechanistic grounding, and the efficiency penalty. It needs no annotated trajectories, so it promotes exploration beyond the logged behaviors.

The curriculum scales the horizon budget from short to long, the realized module size from small to large, the per-task difficulty that jointly controls dropped and added genes and the distractor band, and the lattice tier from single contexts to cross-organ combinations. Emphasis falls on the trajectories that need many hops, the projection and return-to-network case and the generation of a context-specific hierarchy. Reward-hacking monitors run throughout, tracking tool-use entropy and trajectory diversity, with a specific guard against the single-pass RWR shortcut, and adversarial modules or a diversity bonus introduced if shortcut behavior appears. Gate, improvement over the preference checkpoint on the flagship evaluations, with the monitors clear.

### **Part IV. Handoff to the mechanistic agent**

The reinforced policy becomes the base for the tool-using agent, running the actor and verifier modes over the four task cases, completion of a partial group, explanation of a coherent group, refinement of a noisy group, and recognition of no shared mechanism. The agent curriculum culminates in the applied case, projecting a drug target or disease signal onto the fixed hierarchy and returning to the network for connectors, paths, and layer support, with directional and translational filtering, which is the repurposing workflow the anxiety, Pan-SUD, and diabetes-proteomics work depends on. Gate, expert-preferred interpretations at a rate that rises across the base, SFT, preference, and policy checkpoints, measured by the blind comparative protocol.

### **Cross-cutting evaluation gates**

Probing and intervention run after the structure steps S5 and S6 and again after each later stage, training probes on the model's activations to recover edge presence, layer membership, clade co-membership, and connector status, then intervening to test that the recovered structure is causal. No stage advances on accuracy alone if the probe test shows the structure is decorative rather than causal.

Two flagship evaluations gate the transitions into the reinforcement stages and the agent handoff. Held-out context tree generation requires the model to produce the correct, context-specific hierarchy for a lattice point it never saw, scored by clade recovery and correct ancestry, and it cannot be met by interpolating seen trees because the trees diverge. Projection and return to network reproduces the two-step operation, seed RWR-LOE from a target gene, enrich its neighborhood against the MENTOR-EV modules at every level of the dendrogram, then recover the connectors and short cross-clade paths that show distant enriched clades are one connected system, scoring the projection, the connectivity recovery, and the interpretation separately.

### **Data-generation dependencies**

Each step names what WS1 must produce first. S1 needs the identifier, tag, and module-id conventions. S2 and S3 need per-graph edge, neighbor, path, and component tables with provenance. S4 needs RWR-LOE rank vectors and distance shards per seed. S5 needs a MENTOR dendrogram per lattice point with clade identifiers. S6 needs cross-clade path, bridge, and cohesion statistics, and the null distributions. S7 needs the projection outputs, seed-neighborhood enrichment against every level of the tree, plus drug and disease evidence sets for the applied case. S8 needs paired multiplexes that differ by one layer and assembled unions. All targets are cached with layer descriptor, metric, seed set, and graph version. Because the curriculum drives generation, WS1 builds these in step order rather than all at once.

### **Parameters to fix after WS0**

The mix weights above, the number and boundaries of the cut-height and distractor bins, the promotion thresholds per gate, the horizon schedule for the policy stage, and the sampling rates across lattice tiers all remain provisional until WS0 shows how strongly the trees and connectivity diverge across contexts. WS0 also decides whether the PEN-driven contexts diverge enough to justify the finer tiers, which sets how deep into cross-organ composition the curriculum should reach in its first cycle.

### **System Prompts for the Training Pipeline**

The pipeline uses one stable core prompt plus thin overlays rather than a separate prompt per step. The core states the invariants that hold everywhere. A book-mode overlay, chosen per example from its book\_mode tag, grants or withholds capability. The later regimes, preference optimization, policy gradient, and the deployed agent, add their own framing on the same core. The per-step operator lives in the task framing of the user turn, not in the system prompt, so behavior keys to the task rather than to prompt wording. One switch in the core, the bootstrap toggle, controls whether ontology resolution and context navigation are in scope, since those exist only once the lattice is built.

#### **Core, shared by every stage**

*You are MENTOR-RL, a model for systems biology reasoning over multiplex biological networks. These rules hold in every stage.*

*Identifiers. Ensembl gene identifiers are the canonical graph keys. Gene symbols are display aliases, never keys. Resolve a symbol to its canonical Ensembl identifier before use. If it maps to several identifiers, return status ambiguous and stop. If it does not resolve, return status not\_found.*

*Content and facts. You learn operators, not graph content. Edges, weights, ranks, scores, distances, module membership, and clade structure are content that reaches you through context or tools. You never invent them, and you do not rely on memory for them.*

*Statistics. Jaccard, containment, cohesion, density, percentiles, p-values, and composite scores are computed by the runtime, never asserted by you. When a statistic is required and not provided, call the tool that computes it.*

*Encoding. Prefer relational quantities, ranks, percentiles, containment fractions, and clade depths, over raw values.*

*Ontology. A context identifier resolves to registered ontology terms, UBERON and the Human Brain Atlas for anatomy, Cell Ontology and the Provisional Cell Ontology for cell type, and HsapDv for stage. Resolve only from the registered mapping. Do not fabricate ontology terms, and do not claim that a finding in one context generalizes to another.*

*Output. Return one schema-valid JSON object with the requested keys and no text outside it. Use only the enumerated allowed claims, and never assert a disallowed claim. When support is missing or ambiguous, return the correct status and withhold the claim.*

*\[Bootstrap toggle. Before the lattice exists, ontology resolution and context navigation are out of scope. Treat the current graph as the only context and do not claim cross-context relationships.\]*

#### **Closed-book overlay, S1**

*This stage is closed book. You have no tools and no graph content. Answer only from the bounded conventions, identifiers, layer tags and families, direction rules, and identifier formats. Do not produce or guess any per-graph value or any module or clade membership.*

#### **Open-book overlay, S2 through S8**

*This stage is open book. The context contains the structured data your answer is read from. Retrieve, compare, filter, and navigate over that context, and copy continuous values exactly as given. Do not compute statistics yourself, and do not use any value that is not present in the provided context.*

#### **Tool-call overlay, from S4 and consolidated at S9**

*You may call the RWR++ tools. Construct schema-valid calls with biological arguments, gene and layer identifiers, never file paths. Parse the returned objects and read their values rather than restating them from memory. Delegate every statistic to the tool that computes it, and record the tool, graph version, and layer scope as provenance. Never assert a numeric value you did not receive from a tool.*

#### **Preference stage, DPO**

*You are producing one step of a mechanistic exploration trajectory. Given the query, the working interpretation, and the structured state, emit a reasoning step with an optional tool action, or a verifier update to the interpretation and state with a continuation decision of continue, revise, or stop. Keep every update schema valid and evidence-backed. Your step will be compared against alternatives, so prefer the move best grounded in the evidence rather than the one that merely looks plausible.*

#### **Policy stage, GRPO**

*You are exploring a multiplex to build and verify a mechanistic hypothesis over many steps. Use the tools to gather evidence, update the interpretation and the structured state as you go, and stop when the evidence supports a conclusion or the budget is spent. Recover the group or mechanism through genuine multi-step exploration rather than a single-pass expansion from the seeds. Shorter, well grounded trajectories are preferred over long or padded ones.*

#### **Deployed agent, actor and verifier**

*You are MENTOR-RL interpreting a multiplex for a scientist. In actor mode, propose a reasoning step and, when useful, a tool call. In verifier mode, update the human-readable interpretation and the schema-valid structured state, and emit continue, revise, or stop. Ground every claim in retrieved evidence, state uncertainty, and abstain when support is weak. Read the dendrogram as an index and confirm hypotheses on the network. Return the interpretation for the scientist and the structured state for the record.*

## **MENTOR-RL Data Catalog and Validator Spec**

### **Purpose and how to use**

This document defines, for every question family in the first three curriculum steps, the input the generator supplies, the exact answer schema, a worked example, the validator that checks the target, and the metric used at evaluation. A family is buildable when its schema and validator are frozen here.

One point on mechanics. Supervised fine-tuning optimizes next-token cross-entropy on the target string. The validators below do two jobs, they gate an example into the corpus only if its target validates, and they reappear at evaluation to score model output. They do not shape the loss.

### **Shared conventions**

Every question carries a metadata block so the model learns which graph, layer scope, and namespace it is operating in.

{

"schema\_version": "multiplex\_sft\_v2",

"book\_mode": "closed\_book | open\_book | tool\_call",

"step": "S1 | S2 | S3",

"question\_family": "\<family\_id\>",

"multiplex\_id": "brain:region:tissue:celltype:v1 | full\_brain\_multiplex\_v1",

"layer\_scope": "single\_layer | layer\_subset | all\_layers",

"layer\_ids": \["HumanNetV3:string\_ppi:global:all:v3"\],

"entity\_namespace": "ensembl\_gene\_id\_primary",

"answer\_format": "json"

}

Ensembl identifiers are the canonical graph keys, and gene symbols are display aliases only. Every answer is a single JSON object. Open-book questions carry the structured context the answer is read from, under a context field, so continuous values are retrieved rather than recalled. Closed-book questions carry no graph content and target only stable conventions. Counts over a provided short list are allowed as integers. Continuous values, weights, scores, distances, and percentiles, are copied verbatim from the provided context and never computed in the forward pass. Statistics that require computation are produced by the runtime and either provided in context or fetched by a tool.

### **Validator primitives**

Each answer field is checked by one primitive. Family validators compose these.

| Primitive | Rule | Eval metric |
| :---- | :---- | :---- |
| EXACT\_ID | String equality to canonical Ensembl id after normalization | Accuracy |
| SYMBOL | Case-insensitive equality, not accepted as a graph key | Accuracy |
| ID\_SET | Order-independent set equality over Ensembl ids | Jaccard for partial credit |
| RANKED\_LIST | Ordered equality of ids, ties resolved by the runtime's stable order | Top-k accuracy and Kendall tau |
| ENUM | Membership in a fixed vocabulary, exact | Accuracy |
| INT | Exact integer equality, used only for counts over provided items | Accuracy |
| FLOAT\_COPY | Equality to a value present in the provided context, tolerance 1e-6 | Accuracy |
| BOOL | Exact boolean | Accuracy |
| PATH | Node sequence is a valid walk in the stated scope and its length equals the runtime shortest-path length, any optimal path accepted | Validity rate and optimal-length rate |
| CLAIM\_GATE | Selected claim matches one of the enumerated allowed claims, and no disallowed claim is asserted | Allowed-claim accuracy and disallowed-claim leak rate |
| PROVENANCE | Required provenance fields present and consistent with the metadata block | Consistency rate |
| DERIVED\_RANK | Model predicts an integer cutoff, checked exactly against the runtime value, distinct from a copied count | Accuracy and mean absolute rank error |
| PARTITION | Two or more ID\_SET blocks checked for agreement up to reordering | Adjusted Rand index |
| MAP | An object keyed by clade or gene id, keys checked by ID\_SET and each value by its own primitive | Key Jaccard and per-value accuracy |
| TOOL\_CALL | tool\_name ENUM against the tool vocabulary plus arguments checked against the tool input schema, raw file paths rejected | Selection accuracy and schema validity |
| STATE\_UPDATE | Structured-state fields typed, relationship\_status ENUM, predicted\_groups ID\_SET per group, continuation\_state ENUM | Group Jaccard and field accuracy |

Data QC is all-or-nothing, an example enters the corpus only if every required field validates. Evaluation reports per-field pass rate, family exact-match rate, and the graded metrics above.

### **Step S1. Coordinate system and conventions, closed-book**

S1 teaches bounded, stable conventions, so targets are memorized and no graph content is supplied.

##### **Tokenization decision, identifiers**

Identifiers use atomic tokenization. Each gene takes one token for the ENSG prefix and one token for the accession body, each gene symbol takes one token, and module identifiers are tokenized compositionally, with atomic tokens for the source and dendrogram and for the digits of the clade index rather than one token per module, so parsing survives and the scheme scales to the full clade space. The typed-wrapper scheme, which marks entities with type tags, is rejected, because it gave no overall gain and regressed cross-context alignment.

An evaluation on a hard question subset motivates this. Atomic tokenization raised overall accuracy from 45.6 to 85.9 percent and solved the identifier mapping outright, while the typed wrapper matched control overall and dropped cross-context alignment from 96.9 to 78.1 percent.

| Family | Control | Typed wrapper | Atomic |
| ----- | ----- | ----- | ----- |
| Overall | 45.6% | 45.6% | 85.9% |
| Symbol to Ensembl | 8.3% | 8.3% | 100% |
| Ensembl to symbol | 5.6% | 8.3% | 100% |
| Module parsing | 8.3% | 11.1% | 91.7% |
| Ambiguous symbols | 5.6% | 16.7% | 16.7% |
| Cross-context alignment | 96.9% | 78.1% | 93.8% |
| Direction, layer-family, layer-tag | 100% | 100% | 100% |

The residual error after atomic tokenization is almost entirely ambiguous symbols, thirty of the thirty-five remaining misses, which tokenization does not address because it is a deferral and reasoning behavior rather than a lookup. That family is a separate training target, more ambiguous examples and explicit candidate-set supervision under the claim gate, not a tokenization change.

Two constraints hold. Atomic tokens are confined to the identifier namespace, the bounded convention we memorize, and are never used to store graph content, which still arrives as context, and the disjoint token spaces give type separation for free, which is what the wrapper was reaching for. Before the full-scale commit, re-run the evaluation at close to the full gene and symbol vocabulary, on the order of forty thousand added tokens, with held-out and low-frequency entities in the test, since single-token association can look perfect on a closed set while a rare or unseen entity has an undertrained embedding, and confirm the compositional module scheme parses at scale. Read single-example family moves as noise given the small counts.

#### **S1.1 Entity normalization, symbol to Ensembl**

Teaches that symbols resolve to a canonical key before any lookup. Input, a symbol and a multiplex id. Answer.

{ "status": "resolved", "gene\_id": "ENSG00000112164", "gene\_symbol": "GLP1R",

"canonical\_entity": "\<GENE:ENSG00000112164|GLP1R\>", "multiplex\_id": "full\_brain\_multiplex\_v1" }

Validator, status ENUM in {resolved, not\_found, ambiguous}, gene\_id EXACT\_ID, gene\_symbol SYMBOL, canonical\_entity EXACT\_ID pattern, multiplex\_id ENUM. Metric, resolution accuracy. Negative variant, status not\_found with an allowed\_claim CLAIM\_GATE stating no lookup proceeds until resolution.

#### **S1.2 Entity normalization, Ensembl to symbol**

Input, an Ensembl id. Answer carries gene\_symbol and canonical\_entity. Validator, gene\_symbol SYMBOL, gene\_id EXACT\_ID, canonical\_entity EXACT\_ID pattern. Metric, accuracy.

#### **S1.3 Ambiguous symbol resolution**

Teaches deferral under ambiguity. Answer.

{ "status": "ambiguous", "candidate\_gene\_ids": \["ENSG...A1","ENSG...A2"\],

"action": "ask\_for\_disambiguation\_or\_use\_context",

"allowed\_claim": "no\_graph\_lookup\_until\_canonical\_id\_resolved" }

Validator, status ENUM, candidate\_gene\_ids ID\_SET, action ENUM, allowed\_claim CLAIM\_GATE. Metric, accuracy plus disallowed-claim leak rate.

#### **S1.4 Cross-context entity alignment**

Teaches that a symbol in one artifact and an id in another are the same key. Answer carries same\_entity BOOL, gene\_id EXACT\_ID, gene\_symbol SYMBOL, reason CLAIM\_GATE. Metric, accuracy.

##### **S1.5 Multiplex identifier resolution**

Teaches that a context identifier resolves to grounded ontology terms rather than to a parsed flat string. Input, a context id. Answer, the anatomy term as UBERON with the finer HBA term where present, the cell-type term as CL with the PCL term where the type is transcriptomic, the developmental stage as HsapDv, the PEN layers that define the gene universe, and the taxonomy cluster id that produced them. Validator, `context_id` EXACT\_ID, each ontology term EXACT\_ID against its ontology id space, `developmental_stage` ENUM, `pen_layers` ID\_SET, and `taxonomy_cluster_id` EXACT\_ID. Metric, per-field resolution accuracy. This family carries the lattice addressing, so its ontology terms are the source of truth for the whole build.

json

// book\_mode: closed\_book, identifiers illustrative

"answer": {

  "context\_id": "brain:hippocampus:CA1:astrocyte:adult:v1",

  "anatomy":   { "uberon": "UBERON:0003881", "hba": "HBA:CA1", "label": "CA1 field of hippocampus" },

  "cell\_type": { "cl": "CL:0000127", "pcl": "PCL:00170xx",

                 "taxonomy\_cluster\_id": "siletti:cluster\_0123", "label": "astrocyte" },

  "developmental\_stage": { "hsapdv": "HsapDv:0000087", "label": "adult" },

  "pen\_layers": \["scPEN:brain:CA1:astrocyte:v1"\]

}

#### **S1.6 Layer tag parsing**

Input, a layer tag. Answer decomposes source, modality, context, cell type or region, version, and layer family.

{ "layer\_id": "bulkPEN:GTEx\_v9-brain-hippocampus:v1", "source": "GTEx\_v9",

"modality": "bulk\_pen", "context": "brain", "cell\_type\_or\_region": "hippocampus",

"version": "v1", "layer\_family": "bulk\_pen" }

Validator, source and modality and layer\_family ENUM, others by pattern. Metric, per-field accuracy.

#### **S1.7 Layer family classification**

Input, a layer tag. Answer, layer\_family ENUM in {ppi, coexpression, tf\_target, bulk\_pen, scpen, other}. Validator, ENUM. Metric, accuracy and confusion matrix across families.

#### **S1.8 Direction conventions**

Teaches the ordering rules the later steps depend on. Input, a metric name. Answer.

{ "metric": "rwr\_loe\_rank", "lower\_is\_closer": true,

"rule": "lower\_rank\_stronger\_proximity" }

Validator, lower\_is\_closer BOOL, rule CLAIM\_GATE against the enumerated convention set covering rank, score, distance, and percentile. Metric, accuracy. This family removes direction ambiguity before any comparison question appears.

#### **S1.9 Module identifier parsing**

Input, a module id. Answer parses source and construction.

{ "module\_id": "rwr\_loe:full\_brain\_multiplex\_v1:seed\_ENSG00000112164:geometric\_elbow\_v1",

"module\_source": "rwr\_loe", "seed\_gene\_id": "ENSG00000112164",

"construction\_rule": "seed\_centered\_rwr\_loe\_geometric\_elbow" }

For a MENTOR-EV id the source is mentor\_ev and the parse returns the dendrogram and clade identifiers. Validator, module\_source ENUM, seed\_gene\_id EXACT\_ID where present, construction\_rule CLAIM\_GATE. Metric, per-field accuracy.

The module identifier is tokenized compositionally, atomic tokens for the source and dendrogram and digit tokens for the clade index rather than one token per module, so the parse stays well defined and scales to the full clade space, following the tokenization decision above.

### **Step S2. Atomic graph facts, open-book**

S2 supplies the relevant tables under context and trains retrieval, counting, set operations, and claim-gating. Continuous values are copied from context.

#### **S2.1 Layer-specific edge existence**

Context, the edge record or its absence for the pair in the stated layer. Answer.

{ "edge\_exists": true, "source\_gene\_id": "ENSG\_A", "target\_gene\_id": "ENSG\_B",

"layer\_id": "HumanNetV3:string\_ppi:global:all:v3", "weight": 0.84 }

Validator, edge\_exists BOOL, ids EXACT\_ID, layer\_id ENUM, weight FLOAT\_COPY. Negative variant sets edge\_exists false, drops weight, and carries allowed\_claim CLAIM\_GATE stating no edge is recorded in this layer and version. Metric, existence accuracy, weight-copy accuracy, disallowed-claim leak rate.

#### **S2.2 Multiplex edge existence**

Context, per-layer edge presence for the pair. Answer carries edge\_exists\_any\_layer BOOL, supporting\_layers ID\_SET over layer ids, supporting\_layer\_count INT. Validator as typed. Metric, existence accuracy and supporting-layer Jaccard. Teaches that an edge can exist in some layers and not others.

#### **S2.3 Layer-specific direct neighbors**

Context, the neighbor table for the query gene in the layer. Answer.

{ "query\_gene\_id": "ENSG\_A", "layer\_id": "...", "neighbor\_count": 3,

"neighbors": \[ {"gene\_id":"ENSG\_B","weight":0.91}, {"gene\_id":"ENSG\_C","weight":0.73} \] }

Validator, query\_gene\_id EXACT\_ID, layer\_id ENUM, neighbor\_count INT, neighbors as ID\_SET on gene\_id with each weight FLOAT\_COPY. Metric, neighbor-set Jaccard and count accuracy. Open-book variant provides a redundant table and asks for deduplication.

#### **S2.4 Multiplex direct neighbors**

Answer is a neighbor-to-layer map, each neighbor carrying supporting\_layers ID\_SET and supporting\_layer\_count INT, plus unique\_neighbor\_count INT. Validator as typed. Metric, neighbor Jaccard and per-neighbor layer Jaccard. The answer must be a map, not a flat list.

#### **S2.5 Gene layer membership**

Context, the layer list for the gene. Answer, gene\_id EXACT\_ID, layer\_count INT, layers ID\_SET over layer ids. Metric, layer Jaccard.

#### **S2.6 Nodes by layer**

Context, presence flags for a gene set in a layer. Answer carries present\_gene\_ids and absent\_gene\_ids as ID\_SET, with counts INT. Metric, set Jaccard on both partitions.

#### **S2.7 Shared neighbors**

Context, neighbor sets for two genes in a layer. Answer, shared\_neighbor\_count INT, shared\_neighbors ID\_SET. Metric, Jaccard. Teaches local topology beyond a single edge.

#### **S2.8 Degree and hubness**

Context supplies the gene degree and the runtime-computed degree percentile against the sampled distribution, so no arithmetic is required. Answer.

{ "gene\_id": "ENSG\_A", "scope": "all\_layers", "degree": 1842,

"degree\_percentile": 99.2, "hub\_like": true,

"caveat": "high\_degree\_inflates\_apparent\_proximity" }

Validator, degree INT copied, degree\_percentile FLOAT\_COPY, hub\_like BOOL against a fixed percentile threshold stated in context, caveat CLAIM\_GATE. Metric, hub classification accuracy and caveat presence.

#### **S2.9 Induced subgraph**

Context, the intra-set edges in the layer. Answer, present\_gene\_ids and missing\_gene\_ids ID\_SET, edge\_count INT, edges as a set of typed triples with weight FLOAT\_COPY. Validator, edges as ID\_SET over unordered id pairs with weight copy. Metric, edge-set Jaccard. A companion field combined\_edge\_count across all layers is checked as INT.

#### **S2.10 Connected components**

Context, the intra-set adjacency in the layer. Answer, single\_component BOOL, component\_count INT, components as a set of ID\_SET partitions. Validator, partition equality up to reordering. Metric, partition agreement by adjusted Rand index. Supports the later insufficient-support behavior.

#### **S2.11 Calibration, no edge and no path**

Teaches the boundary of a negative result. No graph content beyond the stated absence. Answer.

{ "allowed\_claim": "no\_edge\_or\_path\_recorded\_in\_this\_version\_and\_scope",

"disallowed\_claims": \[ "genes\_biologically\_unrelated\_in\_all\_contexts",

"no\_interaction\_exists\_in\_biology",

"relationship\_experimentally\_disproven" \] }

Validator, allowed\_claim CLAIM\_GATE, disallowed\_claims checked against the enumerated forbidden set. Metric, allowed-claim accuracy and disallowed-claim leak rate. This family recurs at high frequency to suppress overclaiming.

#### **S2.12 Calibration, gene absent from layer**

Answer carries status gene\_absent\_from\_layer, allowed\_action and disallowed\_action CLAIM\_GATE. Metric, action accuracy.

#### **S2.13 Calibration, layer specificity**

Teaches that a coexpression edge does not license a physical interaction claim. Answer carries allowed\_claim and disallowed\_claim CLAIM\_GATE keyed to the layer family of the supporting edge. Metric, claim accuracy stratified by layer family.

### **Step S3. Paths and cross-layer routes, open-book**

S3 supplies the path context and trains path validity, layer decomposition, and the first connectivity reasoning. Paths are validated for validity and optimal length, and any optimal path is accepted.

#### **S3.1 Shortest path, monoplex**

Context, the layer adjacency along the region between source and target. Answer.

{ "path\_exists": true, "layer\_id": "...", "source\_gene\_id": "ENSG\_A",

"target\_gene\_id": "ENSG\_B", "path\_gene\_ids": \["ENSG\_A","ENSG\_X","ENSG\_B"\],

"path\_edges": \[\["ENSG\_A","ENSG\_X"\],\["ENSG\_X","ENSG\_B"\]\], "hop\_count": 2 }

Validator, path\_exists BOOL, path\_gene\_ids and path\_edges checked by PATH in the single layer, hop\_count INT equal to the optimal length. Negative variant sets path\_exists false with allowed\_claim CLAIM\_GATE. Metric, path validity rate, optimal-length rate, and existence accuracy.

#### **S3.2 Shortest path, multiplex**

Context, the aggregate adjacency with per-edge supporting layers. Answer carries path\_gene\_ids, hop\_count INT, and path\_edges where each edge lists supporting\_layers ID\_SET. Validator, PATH in the aggregate scope, supporting\_layers ID\_SET per edge. Metric, validity, optimal length, and per-edge layer Jaccard.

#### **S3.3 Path layer decomposition**

Context, a path with its per-edge layers. Answer, path\_edge\_count INT and layer\_edge\_counts as a map from layer to INT. Validator, INT per layer with the counts summing to path\_edge\_count. Metric, exact map agreement. Since counting provided edges is the only operation, no arithmetic beyond counting is required.

#### **S3.4 Compare monoplex and multiplex paths**

Teaches the meaning of a route that exists only across layers. Answer.

{ "monoplex\_path\_exists": false, "multiplex\_path\_exists": true,

"allowed\_claim": "cross\_layer\_route\_recorded\_not\_single\_layer",

"disallowed\_claim": "direct\_physical\_interaction\_from\_multiplex\_path" }

Validator, two BOOL fields against context, allowed\_claim and disallowed\_claim CLAIM\_GATE. Metric, classification accuracy and disallowed-claim leak rate. This is the first family that teaches the dendrogram-versus-network lesson at the level of a single route.

#### **S3.5 Connector and bridge node identification**

Context, a small two-cluster subgraph with the edges that join the clusters. Answer, bridge\_node\_ids ID\_SET, connector\_edges as a set of unordered id pairs, and cluster\_assignment as two ID\_SET partitions. Validator, ID\_SET on bridges, edge-set equality on connectors, partition agreement on clusters. Metric, bridge Jaccard, connector edge Jaccard, and partition agreement. This family is the atomic unit the S6 cross-clade connectivity step and the projection flagship build on, since a connector between two clusters here becomes a connector between two distant clades there.

One rule carries over and governs both steps. Continuous quantities, scores, distances, cohesion means, containment fractions, and Jaccard values, are FLOAT\_COPY from the provided context, which the runtime computed. The elbow cutoff is the single derived quantity the model is trained to predict rather than copy, checked by DERIVED\_RANK against the runtime elbow, because locating the elbow is an operator we want in the weights. Set intersections and unions are returned as sets and counted as integers, while their ratios are copied, so no ratio is computed in the forward pass.

### **Step S4. RWR-LOE neighborhoods, the bottom-up pillar, open-book with tool-call introduced**

S4 supplies the seed's rank vector or distance shard under context and trains lookup, comparison, filtering, the elbow that defines a module, and the stability checks. Difficulty rises through distractor bands drawn from post-elbow rank.

##### **Rank-Vector Task-Context Block**

One block governs how RWR rank vectors are presented to the model across the S4 families and wherever a rank vector appears. It has four parts, the generative story and legend that stays in the preamble, the rendering with its default and its ablation, the per-example anchor and linked-view fields, and the context fade schedule.

###### **Generative story and legend, for the preamble**

Include this whenever rank vectors appear, in the open-book overlay, so the reading rules and the origin of the object are always present.

*Rank vector legend and origin. A rank vector is the output of random walk with restart from the seed genes, propagated across all layers of the multiplex as independent lines of evidence and ranked by proximity to the seed. The geometric elbow on the score against rank curve separates the seed's module from its periphery. Read the fields as follows. rank is proximity order, lower is closer. score, where a rendering carries it, is normalized to the seed self-score of 1.0, higher is stronger, and it is copied from context, never computed. band is core, periphery, or background. in\_module is true at or above the elbow rank cutoff. percentile is the score's place in the global distribution, lower is closer than typical. Proximity here means topological closeness under RWR in this context and these parameters. It does not imply a direct edge or a physical interaction, and it does not carry to another context.*

###### **Rendering, default and ablation**

Rank vectors are rendered in a hybrid form by default, an ordered sequence in which position carries rank, with an explicit elbow marker, band grouping into core, periphery, and background, and calibration anchors and quantiles, and no per-entry raw scores. This keeps the ordering signal a sequence model reads well while preserving the elbow that defines the module and the scale needed for calibrated statements. Magnitude is expressed through the bands and anchors rather than through floats in the stream, and explicit closer-than statements are reserved for the S4.2 comparison answers rather than used to represent the whole vector.

seed ENSG00000112164

core, in module: ENSG\_B, ENSG\_C, ENSG\_D, ENSG\_E, ENSG\_F, ENSG\_G, ENSG\_H, ENSG\_I

\[elbow at rank 8\]

periphery: ENSG\_J, ENSG\_K, ENSG\_L ... (to rank 50\)

background: 19542 genes below, score quantiles p50 0.0004, p90 0.0012, p99 0.006

Because the rendering is a deterministic view of one runtime output, it is an ablation arm rather than a fixed choice. Three renderings are generated from the same vector and compared on the S4 interpretation, comparison, and recovery families under the probe condition and the same metrics. The structured table gives every entry a rank, a band, and a copied score with its linked view. The pure ordinal sequence gives only the ordered gene ids with the elbow marker. The hybrid, the default shown above, sits between them. The winning rendering is fixed after the ablation, exemplars for it are drawn only from the train split, and the field order is held constant so the comparison stays clean.

###### **Per-example anchor and linked-view fields, for the context**

Each rank-vector example carries anchors that make the numbers relative and a linked view that ties the neighborhood to the hierarchy. The anchors are the seed self-score at the top of the scale, a background gene near the floor, and the score quantiles. The linked view gives, for each top gene, its clade and its ancestral depth to the seed, so the model sees the RWR neighborhood and the MENTOR-EV position together. The structured-table rendering uses these fields directly, and the hybrid rendering above is generated from them.

json

{

  "seed\_gene\_ids": \["ENSG00000112164"\],

  "context\_id": "brain:hippocampus:CA1:astrocyte:adult:v1",

  "metric": "rwr\_loe", "params\_version": "rwrpp\_v3", "layer\_scope": "all\_layers",

  "elbow\_rank\_cutoff": 8,

  "score\_scale": "normalized\_to\_seed\_self\_1.0",

  "anchors": {

    "seed\_self":      { "rank": 0, "score": 1.0 },

    "background\_ref": { "gene\_id": "ENSG\_Z", "rank": 9987, "score": 0.0003 },

    "score\_quantiles": { "p50": 0.0004, "p90": 0.0012, "p99": 0.006 }

  },

  "ranked\_top": \[

    { "rank": 1, "gene\_id": "ENSG\_B", "score": 0.62, "band": "core", "in\_module": true,

      "linked\_view": { "clade\_id": "clade\_0001842", "lca\_depth\_to\_seed": 2 } },

    { "rank": 9, "gene\_id": "ENSG\_J", "score": 0.03, "band": "periphery", "in\_module": false,

      "linked\_view": { "clade\_id": "clade\_0002050", "lca\_depth\_to\_seed": 6 } }

  \],

  "tail\_summary": { "remaining\_genes": 19542, "band": "background" }

}

###### **Context fade schedule**

Context richness starts high and thins toward the deployment condition, so the model learns fast early and internalizes the operator rather than depending on the scaffold. The fade is also monotone within a stage, richest at the start and leaner as accuracy holds.

| Stage | Legend | Anchors | Linked views | Few-shot exemplars |
| ----- | ----- | ----- | ----- | ----- |
| S1 to S2, and the start of any stage | full | full, self, background, and quantiles | included | one to three, train split only |
| S3 to S6 | full | full | on demand for the family | none |
| S7 to S8 | condensed to a one-line reference | quantiles only | only when the task needs them | none |
| S9, DPO, GRPO, deployed agent | one-line reference | fetched by tool if needed | fetched by tool if needed | none |

Three rules keep the fade honest. Match the lean endpoint to what the deployed agent will actually carry, so training and serving agree. Draw any few-shot exemplars only from the train split, so the scaffold does not leak test answers. And in the probe and intervention evaluation, test interpretation under the lean condition, since a model that reads a vector correctly only with the full legend present has not yet moved the operator into its weights.

#### **S4.1 Rank lookup**

Context, the seed's rank vector rows for the target. Answer.

{ "seed\_gene\_ids": \["ENSG00000112164"\], "target\_gene\_id": "ENSG\_B",

"rank": 12, "score": 0.0041, "distance": 0.021,

"rule": "lower\_rank\_and\_distance\_closer\_higher\_score\_stronger" }

Validator, seed\_gene\_ids ID\_SET, target\_gene\_id EXACT\_ID, rank INT copied, score and distance FLOAT\_COPY, rule CLAIM\_GATE against the S1.8 convention set. Metric, field accuracy.

#### **S4.2 Compare two genes in one rank vector**

Context, the two rows. Answer, closer\_gene\_id EXACT\_ID, a comparison list carrying each gene's rank INT, score and distance FLOAT\_COPY, and rule CLAIM\_GATE. Validator as typed. Metric, comparison accuracy. Teaches ordering under the direction convention.

#### **S4.3 Closest-k from rank vector**

Context, the ranked rows with the seed excluded. Answer, top\_k INT, exclude\_seed\_genes BOOL, closest\_genes as a RANKED\_LIST of gene ids each with rank INT and score FLOAT\_COPY. Validator, RANKED\_LIST with ties resolved by the runtime stable order. Metric, top-k accuracy and Kendall tau.

#### **S4.4 Query gene filtering**

Context, ranks for a query set against a seed set. Answer, seed\_gene\_ids and query\_gene\_ids ID\_SET, ranked\_query\_genes RANKED\_LIST with rank INT and score FLOAT\_COPY. Metric, ranking accuracy over the query subset.

#### **S4.5 Elbow cutoff membership**

Teaches the module definition once the cutoff is given. Context supplies the runtime elbow\_rank\_cutoff and the ranked rows. Answer.

{ "seed\_gene\_id": "ENSG\_A", "elbow\_rank\_cutoff": 8,

"membership\_rule": "rank \< elbow\_rank\_cutoff",

"retained\_gene\_ids": \["ENSG\_B","ENSG\_C","ENSG\_D"\],

"excluded\_gene\_ids": \["ENSG\_E","ENSG\_F"\] }

Validator, elbow\_rank\_cutoff INT copied, membership\_rule CLAIM\_GATE, retained and excluded ID\_SET consistent with the rule and the boundary set to strict less-than to match the methods proposal. Metric, membership Jaccard on both partitions.

#### **S4.6 Elbow reasoning**

Teaches the model to locate the cutoff itself. Context supplies only the ordered rank and score curve. Answer carries elbow\_rank\_cutoff, and high\_score\_side\_gene\_ids and low\_score\_side\_gene\_ids as ID\_SET, with membership\_rule CLAIM\_GATE. Validator, elbow\_rank\_cutoff by DERIVED\_RANK against the runtime geometric elbow, side partitions PARTITION. Metric, cutoff accuracy, mean absolute rank error, and partition agreement. This is the one family where the model predicts a derived quantity.

#### **S4.7 Leave-one-out support**

Teaches refinement signal. Context, the leave-one-out ranks for each set member. Answer, least\_supported\_gene\_id EXACT\_ID, a support\_table of held\_out\_gene\_id with loo\_rank INT and recommendation ENUM in {keep, drop\_candidate}, and interpretation CLAIM\_GATE. Validator as typed. Metric, drop-candidate identification accuracy.

#### **S4.8 Vector intersection and overlap**

Context, the top-k neighborhoods of two seed sets. Answer, intersection\_gene\_ids ID\_SET, intersection\_size and union\_size INT, jaccard FLOAT\_COPY. Validator, sets and counts checked structurally, the jaccard scalar copied from the runtime value. Metric, intersection Jaccard and copy accuracy on the scalar.

#### **S4.9 Distance matrix pair lookup**

Context, the distance shard row for the pair. Answer, distance\_metric ENUM, lower\_is\_closer BOOL, gene\_a and gene\_b EXACT\_ID, distance FLOAT\_COPY. Metric, copy accuracy.

#### **S4.10 Distance matrix comparison**

Context, a matrix row. Answer, anchor\_gene\_id EXACT\_ID, two candidates each with distance FLOAT\_COPY, closer\_gene\_id EXACT\_ID, rule CLAIM\_GATE. Metric, comparison accuracy.

#### **S4.11 Closest entities from a distance row**

Context, the anchor's row with self excluded. Answer, top\_k INT, closest\_genes RANKED\_LIST with distance FLOAT\_COPY. Metric, top-k accuracy and Kendall tau.

#### **S4.12 Cross-shard routing**

Teaches sharded-store navigation. Context, the current shard bounds and the pair. Answer, requested\_pair as an id pair, current\_shard\_contains\_pair BOOL, next\_shard\_id ENUM against the manifest, reason CLAIM\_GATE. Validator, next\_shard\_id checked against the shard manifest for the row gene. Metric, routing accuracy.

#### **S4.13 Distance percentile calibration**

Context, the pair distance and the runtime global quantiles. Answer, distance and distance\_percentile FLOAT\_COPY, classification ENUM in {unusually\_close, typical, far}, rule CLAIM\_GATE. Validator, classification against the provided quantile bands. Metric, classification accuracy.

#### **S4.14 Rank against distance consistency**

Teaches identity checks before calling values contradictory. Context, a rank and a distance that appear to disagree. Answer, a checks list drawn from the enumerated set covering multiplex id, flist hash, distance metric, seed set, layer scope, and cache version, and allowed\_claim CLAIM\_GATE. Validator, checks as ID\_SET against the enumerated check vocabulary, allowed\_claim CLAIM\_GATE. Metric, check-set recall and disallowed-claim leak rate.

#### **S4.15 Layer ablation**

Context, the runtime effect of removing a layer on the pair proximity. Answer, pair as an id pair, ablated\_layer ENUM, effect ENUM in {proximity\_worsened, proximity\_improved, no\_change}, caveat CLAIM\_GATE. Metric, effect accuracy and caveat presence.

#### **S4.16 Seed essentiality**

Context, the runtime rank-vector shift when a seed is removed. Answer, gene\_id EXACT\_ID, effect ENUM in {large\_rank\_vector\_shift, small\_shift}, description ENUM in {seed\_essential, non\_essential}, allowed\_claim CLAIM\_GATE. Metric, essentiality accuracy.

#### **S4.17 Tool selection and parsing**

Introduces tool-call mode. For selection, the answer names the correct RWR++ tool with biological arguments, for example rwr with seed\_genes and top\_k, or get\_rank with source\_gene and target\_gene, or get\_distance with a gene pair and metric. Validator, tool\_name ENUM against the tool vocabulary, arguments checked against the tool input schema, with raw file-path arguments rejected. For parsing, the context is a tool observation and the answer extracts the closest non-seed genes as a RANKED\_LIST. Metric, tool-selection accuracy, argument-schema validity, and parse accuracy.

### **Step S5. MENTOR-EV hierarchy navigation, the top-down pillar, open-book**

S5 supplies the per-context dendrogram as a parent-pointer structure under context and trains navigation, cohesion, and the cross-source relations that tie the bottom-up and top-down views together. All targets are alignment-free and read from one tree. Cut height is the difficulty dial, coarse clades first.

#### **S5.1 Clade membership**

Context, the clade's leaf set. Answer.

{ "module\_id": "mentor\_ev:full\_brain\_multiplex\_v1:gw\_dendrogram\_v2:clade\_0001842",

"module\_source": "mentor\_ev", "gene\_count": 4,

"gene\_ids": \["ENSG\_A","ENSG\_B","ENSG\_C","ENSG\_D"\] }

Validator, module\_id and module\_source ENUM and pattern, gene\_count INT, gene\_ids ID\_SET. Metric, membership Jaccard.

#### **S5.2 Lowest common ancestor**

Context, the tree path structure for the pair. Answer, gene\_a and gene\_b EXACT\_ID, lca\_clade\_id EXACT\_ID against the clade id space, lca\_depth INT. Validator, lca\_clade\_id EXACT\_ID, lca\_depth INT read from the provided tree. Metric, LCA accuracy and depth accuracy. This is the core hierarchy operator.

#### **S5.3 Subtree induced by a cut**

Context, the tree and a cut height. Answer, cut\_height FLOAT\_COPY, clade\_ids at that cut as ID\_SET, and for a queried clade its member ID\_SET. Validator, cut membership as PARTITION over the gene universe at that height. Metric, partition agreement. Teaches that granularity follows the cut.

#### **S5.4 Parent and child relation**

Context, the two clades. Answer, child\_module and parent\_module ids, is\_nested BOOL, child\_gene\_count and parent\_gene\_count INT. Validator, is\_nested BOOL against subtree containment, counts INT. Metric, nesting accuracy.

#### **S5.5 Sibling modules**

Context, the immediate parent and its children. Answer, query\_module id, parent\_module id, sibling\_modules ID\_SET over clade ids. Metric, sibling Jaccard.

#### **S5.6 Subtree membership**

Context, a clade and a gene. Answer, gene\_id EXACT\_ID, clade\_id EXACT\_ID, in\_subtree BOOL. Metric, membership accuracy. The atomic same-clade primitive that comparisons across contexts reuse by value.

#### **S5.7 Within-clade cohesion**

Context, the runtime mean within-clade distance and the null. Answer, module\_id, pair\_count INT, mean\_within\_distance FLOAT\_COPY, distance\_metric ENUM, lower\_is\_closer BOOL. A companion field carries empirical\_p\_value FLOAT\_COPY and interpretation CLAIM\_GATE. Validator, cohesion and p-value copied, interpretation gated. Metric, copy accuracy and interpretation accuracy. The statistic is computed by the runtime, not the model.

#### **S5.8 Cut-height granularity reasoning**

Teaches that the same genes group differently at different heights. Context, two cuts of the same subtree. Answer, coarse\_clade\_ids and fine\_clade\_ids as ID\_SET, and a mapping of each fine clade to its coarse parent as a map of EXACT\_ID. Validator, PARTITION at each height and map consistency. Metric, partition agreement and map accuracy.

#### **S5.9 RWR-LOE against MENTOR-EV containment**

Opens the cross-source families that reward consistency between the two pillars. Context, an RWR-LOE module and a MENTOR-EV clade. Answer, subset BOOL, candidate\_subset and candidate\_superset ids, violating\_gene\_ids ID\_SET, containment\_fraction FLOAT\_COPY. Validator, subset BOOL against the sets, violating set as ID\_SET, fraction copied. Metric, subset accuracy and violating-set Jaccard.

#### **S5.10 Near-subset with violating genes**

Context, an RWR-LOE module that is not an exact subset. Answer, exact\_subset BOOL, containment\_fraction FLOAT\_COPY, violating\_gene\_ids ID\_SET, allowed\_claim CLAIM\_GATE. Metric, violating-set Jaccard and claim accuracy.

#### **S5.11 Jaccard and containment coefficients**

Context, two modules across sources. Answer, intersection\_size, module\_a\_size, and module\_b\_size INT, jaccard, fraction\_of\_a\_in\_b, and fraction\_of\_b\_in\_a FLOAT\_COPY. Validator, sizes counted structurally, ratios copied from runtime values. Metric, size accuracy and ratio copy accuracy. Teaches direction, since containment is asymmetric while Jaccard is not.

#### **S5.12 Set difference, intersection, and multi-module intersection**

Context, two or more modules. Answer for difference, set\_difference direction ENUM, gene\_ids ID\_SET, count INT. Answer for multi-module intersection, modules list and intersection\_gene\_ids ID\_SET with intersection\_size INT. Metric, set Jaccard on the result.

#### **S5.13 Best matching module**

Context, a query module and candidate modules with their overlap rows. Answer, query\_module id, ranking\_metric ENUM in {jaccard, containment}, best\_match\_module id, best\_match\_score FLOAT\_COPY, ranked\_matches as a RANKED\_LIST of module ids each with jaccard and containment FLOAT\_COPY. Validator, best\_match\_module EXACT\_ID, ranking RANKED\_LIST. Metric, best-match accuracy and ranking Kendall tau.

#### **S5.14 Unique genes by source**

Context, a MENTOR-EV module and several RWR-LOE modules. Answer, mentor\_ev\_module id, rwr\_loe\_modules ID\_SET, mentor\_ev\_unique\_gene\_ids ID\_SET, count INT. Metric, unique-set Jaccard.

#### **S5.15 Set overlap against topological distance**

Teaches the divergence between set overlap and network proximity, the seed of the S6 lesson. Context, two modules with small set overlap but a runtime inter-module distance in the closest global band. Answer.

{ "set\_relationship": "weak\_overlap", "topological\_relationship": "globally\_close",

"basis": { "intersection\_size": 2, "distance\_percentile": 1.0 } }

Validator, set\_relationship and topological\_relationship ENUM against the provided bands, basis fields checked, intersection\_size INT and distance\_percentile FLOAT\_COPY. Metric, joint classification accuracy. This family makes explicit that two clades can share almost no members yet sit close in the network, which S6 then extends to connectors and paths.

### **Difficulty and cross-step notes**

In S4 the distractor bands set difficulty, with far post-elbow ranks first and near ranks later, and the top-k horizon widens as accuracy holds. In S5 the cut height sets difficulty, with coarse, well separated clades first and fine, easily confused clades later, and the cross-source families rise in difficulty as the two modules approach near-subset relationships. Both steps advance from single-context multiplexes toward tissue and region tiers, and every biological claim in either step routes through CLAIM\_GATE so overclaiming stays checkable as the operators grow more capable.

The governing rule from the earlier steps still holds. Enrichment scores, q-values, distances, ratios, and composite priority scores are FLOAT\_COPY from context, since the runtime computes them. What the model is trained to produce is the structural and interpretive work, the neighborhood, the per-clade projection read, the cross-clade paths and connectors, the claim about connectivity against dispersion, and the ordering under provided sub-scores. Significance flags are derived by thresholding provided q-values against a stated cutoff, so they stay checkable.

### **Step S6. Cross-clade connectivity, the network pillar, open-book**

S6 supplies the aggregate subgraph spanning two or more clades under context and trains the lesson that distance in the tree is not distance in the network. It reuses the path and connector primitives from S3 at clade scale and adds cohesion against a null.

#### **S6.1 Cross-clade path recovery**

Context, the aggregate adjacency between a gene in clade P and a gene in clade Q that sit far apart in the tree. Answer.

{ "source\_gene\_id": "ENSG\_A", "source\_clade\_id": "clade\_P",

"target\_gene\_id": "ENSG\_B", "target\_clade\_id": "clade\_Q",

"path\_exists": true, "path\_gene\_ids": \["ENSG\_A","ENSG\_X","ENSG\_B"\],

"hop\_count": 2,

"path\_edges": \[ {"edge":\["ENSG\_A","ENSG\_X"\],"supporting\_layers":\["coexpression"\]},

{"edge":\["ENSG\_X","ENSG\_B"\],"supporting\_layers":\["string\_ppi"\]} \],

"crosses\_clades": true }

Validator, path\_gene\_ids and path\_edges by PATH in the aggregate scope, hop\_count INT at optimal length, supporting\_layers ID\_SET per edge, crosses\_clades BOOL against the clade assignment. Metric, path validity, optimal-length rate, per-edge layer Jaccard, and cross-clade flag accuracy.

#### **S6.2 Bridge and connector node identification**

Context, the two clade subgraphs and the edges joining them. Answer, bridge\_node\_ids ID\_SET, connector\_edges as a set of unordered id pairs, and clade\_assignment as a PARTITION over the two clades. Validator as typed. Metric, bridge Jaccard, connector edge Jaccard, and partition agreement. This is the S3.5 primitive lifted to clade scale.

#### **S6.3 Cross-clade path layer decomposition**

Context, a cross-clade path with per-edge layers. Answer, path\_edge\_count INT and layer\_edge\_counts MAP from layer to INT summing to the total. Metric, exact map agreement. Teaches which lines of evidence carry the cross-clade route.

#### **S6.4 Tree distance against network distance**

The core anti-abstraction family. Context, two clades with a large dendrogram separation and a runtime network proximity in the closest global band, plus a short connecting path. Answer.

{ "clade\_a": "clade\_P", "clade\_b": "clade\_Q",

"dendrogram\_relationship": "distant",

"network\_relationship": "closely\_connected",

"basis": { "tree\_separation\_rank": "high", "min\_hop\_count": 2,

"network\_distance\_percentile": 1.4 },

"allowed\_claim": "distant\_in\_tree\_but\_connected\_in\_network",

"disallowed\_claim": "distance\_in\_tree\_implies\_separation\_in\_network" }

Validator, dendrogram\_relationship and network\_relationship ENUM against the provided bands, basis fields checked with min\_hop\_count INT and percentile FLOAT\_COPY, both claims CLAIM\_GATE. Metric, joint classification accuracy and disallowed-claim leak rate.

#### **S6.5 Cross-clade connectivity against a null**

Context, the observed count or length of cross-clade connections and the runtime null from random clade pairs. Answer, observed\_min\_hop INT, empirical\_p\_value FLOAT\_COPY, classification ENUM in {stronger\_than\_random, typical, weaker\_than\_random}, interpretation CLAIM\_GATE. Metric, classification accuracy.

#### **S6.6 Global cohesion against a null**

Context, the runtime within-clade distance, the null of random equal-size sets, and the derived statistics. Answer carries mean\_within\_distance, mean\_outside\_distance, clustering\_ratio, edge\_density, and boundary\_ratio as FLOAT\_COPY, separation ENUM in {well\_separated, diffuse}, and caveat CLAIM\_GATE. Validator, all scalars copied, separation against the provided threshold, caveat gated. Metric, scalar copy accuracy and separation accuracy. Every statistic is runtime-computed.

#### **S6.7 Cell-type-specific cohesion**

Context, cohesion of a clade across two cell-type layers. Answer, module\_id, cohesive\_contexts and noncohesive\_contexts ID\_SET over layer ids, allowed\_claim and disallowed\_claim CLAIM\_GATE. Metric, context-set Jaccard and claim accuracy. Teaches that cohesion can be specific to a cell type.

#### **S6.8 Layer-sensitive cohesion**

Context, the runtime effect of removing a layer on a clade's cohesion. Answer, module\_id, ablated\_layer ENUM, effect ENUM in {cohesion\_decreases, cohesion\_stable}, caveat CLAIM\_GATE. Metric, effect accuracy and caveat presence.

#### **S6.9 Connector essentiality**

Context, the runtime effect of removing a bridge node on the cross-clade connection. Answer, bridge\_node\_id EXACT\_ID, effect ENUM in {connection\_fragments, connection\_holds}, description ENUM in {connector\_essential, redundant}, allowed\_claim CLAIM\_GATE. Metric, essentiality accuracy. Identifies the nodes that hold distant clades together.

#### **S6.10 Multi-clade connector hub**

Context, a node with edges into several clades. Answer, hub\_node\_id EXACT\_ID, connected\_clade\_ids ID\_SET, connected\_clade\_count INT, caveat CLAIM\_GATE against hub bias. Metric, connected-clade Jaccard and caveat presence. The hub families feed the projection step, where one target's neighborhood reaches many clades.

### **Step S7. The projection operation, open-book and tool-call**

S7 assembles the earlier operators into the applied workflow the GLP1R figure demonstrates. The projection runs in two steps, seed RWR-LOE from a target gene, then enrich the resulting neighborhood against the MENTOR-EV modules at every level of the dendrogram. The return to network recovers the connectors and paths that show distant enriched clades are one connected system. Whole-brain first, then disease multiplexes.

#### **S7.1 Neighborhood construction**

The projection's first step, reusing the S4 operator. Context, the target's rank vector and the runtime elbow. Answer, target\_gene\_id EXACT\_ID, neighborhood\_gene\_ids ID\_SET, elbow\_rank\_cutoff by DERIVED\_RANK or copied when provided. Metric, neighborhood Jaccard. The neighborhood is connected by construction, a fact the interpretation family later relies on.

#### **S7.2 Enrichment projection across the hierarchy**

Context, the runtime enrichment of the neighborhood against every clade at each level, as scores and q-values. Answer.

{ "target\_gene\_id": "ENSG00000112164",

"enrichment\_by\_clade": { "clade\_A": {"score":3.9,"q":0.001},

"clade\_B": {"score":3.1,"q":0.004},

"clade\_D": {"score":2.8,"q":0.01} },

"significant\_clade\_ids": \["clade\_A","clade\_B","clade\_D"\],

"q\_threshold": 0.05 }

Validator, enrichment\_by\_clade MAP with score and q FLOAT\_COPY, q\_threshold FLOAT\_COPY, significant\_clade\_ids ID\_SET derived by thresholding the provided q-values. Metric, per-clade copy accuracy and significant-set Jaccard. The model reads and thresholds, it does not compute enrichment.

#### **S7.3 Cross-clade enrichment interpretation**

The interpretation family, the heart of the flagship. Context, significant enrichment in several clades that sit far apart in the tree, plus the fact that the set is one RWR-LOE neighborhood. Answer.

{ "significant\_clade\_ids": \["clade\_A","clade\_B","clade\_D"\],

"tree\_relationship": "distant\_branches",

"interpretation": "one\_connected\_system",

"reason": "enriched\_set\_is\_a\_single\_rwr\_loe\_neighborhood\_seeded\_on\_the\_network",

"disallowed\_claim": "distant\_enriched\_clades\_are\_unrelated" }

Validator, tree\_relationship ENUM, interpretation ENUM in {one\_connected\_system, separate\_systems, indeterminate}, reason and disallowed\_claim CLAIM\_GATE. Metric, interpretation accuracy and disallowed-claim leak rate. This family trains the model to read cross-clade enrichment as connectivity rather than dispersion.

#### **S7.4 Return to network**

Context, the aggregate subgraph over the significant clades. Answer, connector\_node\_ids ID\_SET, cross\_clade\_paths as a list of PATH objects each with supporting\_layers, and connected\_clade\_ids ID\_SET. Validator, connectors ID\_SET, paths by PATH at optimal length, connected clades against the assignment. Metric, connector Jaccard, path validity and optimal-length rate, and connected-clade Jaccard. This is where the hierarchy claim is confirmed on the network.

#### **S7.5 Layer-specific support for the projection**

Context, the layers carrying the enrichment and the connecting paths. Answer, supporting\_layers ID\_SET, layer\_contribution MAP from layer to a copied contribution score, allowed\_claim CLAIM\_GATE stating the evidence type each layer licenses. Metric, layer Jaccard and claim accuracy.

#### **S7.6 Mechanistic match**

Context, drug evidence and disease evidence projected onto the same hierarchy. Answer, drug\_modules and disease\_modules ID\_SET, shared\_or\_adjacent\_modules ID\_SET, match\_type ENUM in {same\_module, adjacent\_module, none}, basis CLAIM\_GATE. Metric, match classification accuracy and shared-module Jaccard.

#### **S7.7 Perturbation direction**

Context, the disease-associated direction per module and the drug's effect direction, where inferable. Answer, per\_module\_direction MAP from module to ENUM in {opposes, aligns, indeterminate}, opposing\_modules ID\_SET, allowed\_claim CLAIM\_GATE. Validator, direction map against context with abstention allowed. Metric, direction accuracy including correct abstention. A useful hypothesis opposes the disease direction in a specific module.

#### **S7.8 Translational filtering**

Context, per-candidate sub-scores for evidence strength, adverse-event risk, brain penetration, feasibility, and testability, plus the runtime composite priority. Answer, ranked\_candidates RANKED\_LIST of gene or drug ids each with the sub-scores and composite FLOAT\_COPY, passes\_filter BOOL against a stated threshold, rationale CLAIM\_GATE. Validator, ranking as RANKED\_LIST, scalars copied, gating BOOL. Metric, ranking Kendall tau and gate accuracy. The composite is runtime-computed, the model ranks and gates.

#### **S7.9 Flagship composite**

The assembled evaluation, scored as three parts rather than one. Context, a held-out target and its multiplex. The model runs S7.1 and S7.2 for the projection, S7.4 for the connectivity recovery, and S7.3 for the interpretation, and the composite records each. Answer, a projection block, a connectivity block, and an interpretation block matching the schemas above, plus provenance PROVENANCE. Validator, each block by its family validator. Metric reported separately, projection score as significant-clade Jaccard and per-clade copy accuracy, connectivity score as connector Jaccard and path validity, and interpretation score as interpretation accuracy with the disallowed-claim leak rate. A candidate passes the flagship only when all three clear their thresholds, since a correct projection with a failed connectivity recovery is the exact failure the pillar exists to prevent.

### **Difficulty and the flagship gate**

In S6 difficulty rises with the tree separation between the clades under test, near-branch pairs first and far-branch pairs later, and with the lattice tier from single context toward whole brain. In S7 difficulty rises with the number of distant clades the neighborhood reaches, with the move from whole-brain to disease multiplexes, and with the shift from targets whose direction is clear to targets where direction must be abstained on. The flagship composite S7.9 is a gate rather than a training family, and it governs the transitions into the reinforcement stages and the agent handoff, alongside the held-out context tree generation gate that S5 and S8 support.

### **Step S8. Composition and lattice scaling, open-book and tool-call**

S8 teaches how structure changes when PEN layers are added or removed. Because the selected PENs define the gene universe, adding or removing a PEN enlarges or shrinks that universe and re-induces HumanNet and the other non-PEN layers on it, so a change is compound rather than a single-layer edit. A union multiplex is the union of the parts' gene universes with the non-PEN layers re-induced on it and the PEN layers stacked. Ground truth for predicted structure is the runtime multiplex built by that rule.

#### **S8.1 Active-layer composition**

Answer, the union's gene universe as the union of the parts' PEN gene sets, the active\_layer\_ids as the PEN layers of the parts together with the non-PEN layers re-induced on that universe, and an induced\_on\_gene\_universe flag recording that the non-PEN layers are filtered to the union gene set. Metric, gene-universe Jaccard and layer Jaccard. Teaches that a union unions the gene universes and re-induces the backbone, not that it merely unions layers.

// book\_mode: open\_book  
 "context": {  
   "parts": {  
 	"brain:hippocampus:CA1:astrocyte:v1": { "pen\_layers": \["scPEN:brain:CA1:astrocyte:v1"\],  
                                          	"pen\_gene\_ids": \["ENSG\_A","ENSG\_B","ENSG\_C"\] },  
 	"brain:hippocampus:CA1:neuron:v1":	{ "pen\_layers": \["scPEN:brain:CA1:neuron:v1"\],  
                                          	"pen\_gene\_ids": \["ENSG\_A","ENSG\_D","ENSG\_E"\] }  
   },  
   "non\_pen\_source\_layers": \["HumanNetV3:string\_ppi:global:all:v3",  
                         	"HumanNetV3:coexpression:global:all:v3"\]  
 }  
 "answer": {  
   "union\_multiplex\_id": "brain:hippocampus:CA1:astrocyte+neuron:v1",  
   "gene\_universe": \["ENSG\_A","ENSG\_B","ENSG\_C","ENSG\_D","ENSG\_E"\],  
   "pen\_layers": \["scPEN:brain:CA1:astrocyte:v1","scPEN:brain:CA1:neuron:v1"\],  
   "induced\_backbone\_layers": \["HumanNetV3:string\_ppi:global:all:v3",  
                           	"HumanNetV3:coexpression:global:all:v3"\],  
   "induced\_on\_gene\_universe": true,  
   "active\_layer\_ids": \["scPEN:brain:CA1:astrocyte:v1","scPEN:brain:CA1:neuron:v1",  
                    	"HumanNetV3:string\_ppi:global:all:v3","HumanNetV3:coexpression:global:all:v3"\]  
 }

#### **S8.2 Minimal-pair neighborhood change (compound)**

Adding or removing a PEN layer is a compound change, since it alters the gene universe and re-induces the non-PEN layers. Context, a seed's RWR-LOE neighborhood in a base multiplex and in the multiplex that results from adding or removing one PEN, with ground truth computed on the re-induced runtime multiplex. Answer, changed\_pen\_layer and direction ENUM, gained\_gene\_ids, lost\_gene\_ids, and stable\_gene\_ids ID\_SET, and an optional delta\_attribution recording how much of the change came from new genes, from the re-induced backbone, and from the added PEN. Metric, gained and lost Jaccard.

// book\_mode: open\_book  
 "context": {  
   "seed\_gene\_id": "ENSG00000112164",  
   "base\_multiplex\_id": "brain:hippocampus:CA1:astrocyte:v1",  
   "base\_neighborhood": \["ENSG\_B","ENSG\_C","ENSG\_D"\],  
   "altered\_multiplex\_id": "brain:hippocampus:CA1:astrocyte+neuron:v1",  
   "change": { "direction": "add", "pen\_layer": "scPEN:brain:CA1:neuron:v1" },  
   "altered\_neighborhood": \["ENSG\_B","ENSG\_C","ENSG\_E","ENSG\_F"\]  
 }  
 "answer": {  
   "seed\_gene\_id": "ENSG00000112164",  
   "changed\_pen\_layer": "scPEN:brain:CA1:neuron:v1",  
   "direction": "add",  
   "gained\_gene\_ids": \["ENSG\_E","ENSG\_F"\],  
   "lost\_gene\_ids": \["ENSG\_D"\],  
   "stable\_gene\_ids": \["ENSG\_B","ENSG\_C"\],  
   "delta\_attribution": { "from\_new\_genes": \["ENSG\_E"\], "from\_reinduced\_backbone": \["ENSG\_D"\],  
                      	"from\_added\_pen": \["ENSG\_F"\] }  
 }

#### **S8.3 Minimal-pair clade reorganization (compound)**

The same compound change reorganizes the dendrogram, because the re-induced backbone and the changed gene universe rebuild the tree. Answer, transition ENUM in {merge, split, shift, stable}, resulting\_clade\_ids ID\_SET, moved\_gene\_ids ID\_SET, checked against the runtime tree of the altered multiplex. Metric, transition accuracy and moved-set Jaccard.

// book\_mode: open\_book  
 "context": {  
   "base\_multiplex\_id": "brain:hippocampus:CA1:astrocyte+neuron:v1",  
   "base\_clade": { "clade\_id": "clade\_0001842", "gene\_ids": \["ENSG\_A","ENSG\_B","ENSG\_C","ENSG\_D"\] },  
   "altered\_multiplex\_id": "brain:hippocampus:CA1:astrocyte:v1",  
   "change": { "direction": "remove", "pen\_layer": "scPEN:brain:CA1:neuron:v1" },  
   "altered\_clades": \[ { "clade\_id": "clade\_0002011", "gene\_ids": \["ENSG\_A","ENSG\_B"\] },  
                   	{ "clade\_id": "clade\_0002014", "gene\_ids": \["ENSG\_C","ENSG\_D"\] } \]  
 }  
 "answer": {  
   "query\_clade\_id": "clade\_0001842",  
   "transition": "split",  
   "resulting\_clade\_ids": \["clade\_0002011","clade\_0002014"\],  
   "moved\_gene\_ids": \["ENSG\_C","ENSG\_D"\]  
 }

#### **S8.4 Minimal-pair connectivity change (compound)**

Answer, connection\_change ENUM in {appears, disappears, unchanged}, affected\_bridge\_node\_ids ID\_SET, and affected\_layers ID\_SET spanning the changed PEN and any re-induced non-PEN layers that carried the route. Metric, change accuracy.

// book\_mode: open\_book  
 "context": {  
   "pair": \["ENSG\_A","ENSG\_B"\],  
   "base\_multiplex\_id": "brain:hippocampus:CA1:astrocyte+neuron:v1",  
   "base\_path": { "path\_gene\_ids": \["ENSG\_A","ENSG\_X","ENSG\_B"\], "bridge\_node\_ids": \["ENSG\_X"\] },  
   "altered\_multiplex\_id": "brain:hippocampus:CA1:astrocyte:v1",  
   "change": { "direction": "remove", "pen\_layer": "scPEN:brain:CA1:neuron:v1" },  
   "altered\_path": { "path\_exists": false }  
 }  
 "answer": {  
   "pair": \["ENSG\_A","ENSG\_B"\],  
   "connection\_change": "disappears",  
   "affected\_bridge\_node\_ids": \["ENSG\_X"\],  
   "affected\_layers": \["scPEN:brain:CA1:neuron:v1","HumanNetV3:coexpression:global:all:v3"\]  
 }

#### **S8.5 Union neighborhood prediction**

Context, a seed's neighborhoods in the parts together with the parts' PEN gene sets. Answer, the predicted neighborhood in the union multiplex, ID\_SET, validated against the runtime union. Metric, neighborhood Jaccard. This is a compositional generalization family. The union is validated against the runtime multiplex whose gene universe is the union of the parts' PEN gene sets and whose non-PEN layers are re-induced on that universe, so the prediction is not the naive set union, since the re-induced backbone and cross-layer propagation gain or drop members.

// book\_mode: open\_book  
 "context": {  
   "seed\_gene\_id": "ENSG00000112164",  
   "parts": {  
 	"brain:hippocampus:CA1:astrocyte:v1": { "neighborhood": \["ENSG\_B","ENSG\_C","ENSG\_D"\],  
                                          	"pen\_gene\_ids": \["ENSG\_A","ENSG\_B","ENSG\_C","ENSG\_D"\] },  
 	"brain:hippocampus:CA1:neuron:v1":	{ "neighborhood": \["ENSG\_B","ENSG\_E","ENSG\_F"\],  
                                          	"pen\_gene\_ids": \["ENSG\_A","ENSG\_B","ENSG\_E","ENSG\_F"\] }  
   }  
 }  
 "answer": {  
   "seed\_gene\_id": "ENSG00000112164",  
   "union\_multiplex\_id": "brain:hippocampus:CA1:astrocyte+neuron:v1",  
   "predicted\_neighborhood\_gene\_ids": \["ENSG\_B","ENSG\_C","ENSG\_D","ENSG\_E","ENSG\_F","ENSG\_G"\]  
 }

#### **S8.6 Held-out context co-membership and ancestry**

The tree-generation family, operationalized on tractable queries rather than emission of a full tree. Context, a held-out context's active-layer descriptor and a query gene set. Answer, co\_membership MAP of gene pairs to BOOL same-clade, lca\_depth per pair INT, predicted\_partition PARTITION of the query set, validated against the runtime dendrogram of the held-out context. Metric, adjusted Rand index, same-clade accuracy, LCA-depth accuracy. Because trees diverge, this cannot be met by copying a seen tree.

json

// book\_mode: open\_book

"context": {

"held\_out\_multiplex\_id": "brain:cortex:L5:microglia:v1",

"active\_layer\_ids": \["HumanNetV3:string\_ppi:global:all:v3",

"HumanNetV3:coexpression:global:all:v3",

"scPEN-brain-cortex\_L5:microglia:v1"\],

"query\_gene\_ids": \["ENSG\_A","ENSG\_B","ENSG\_C","ENSG\_D"\]

}

"answer": {

"held\_out\_multiplex\_id": "brain:cortex:L5:microglia:v1",

"co\_membership": {

"ENSG\_A|ENSG\_B": true, "ENSG\_A|ENSG\_C": false, "ENSG\_A|ENSG\_D": false,

"ENSG\_B|ENSG\_C": false, "ENSG\_B|ENSG\_D": false, "ENSG\_C|ENSG\_D": true

},

"lca\_depth": { "ENSG\_A|ENSG\_B": 2, "ENSG\_C|ENSG\_D": 3,

"ENSG\_A|ENSG\_C": 7, "ENSG\_A|ENSG\_D": 7,

"ENSG\_B|ENSG\_C": 7, "ENSG\_B|ENSG\_D": 7 },

"predicted\_partition": \[ \["ENSG\_A","ENSG\_B"\], \["ENSG\_C","ENSG\_D"\] \]

}

#### **S8.7 Divergence characterization**

Answer, co\_membership\_flips ID\_SET of gene pairs that change same-clade status across two contexts, divergence\_summary ENUM in {high, moderate, low} against provided bands. Metric, flip-set Jaccard and summary accuracy. Teaches where and how much trees diverge across the lattice.

json

// book\_mode: open\_book

"context": {

"context\_a": "brain:hippocampus:CA1:astrocyte:v1",

"context\_b": "brain:hippocampus:CA1:neuron:v1",

"gene\_set": \["ENSG\_A","ENSG\_B","ENSG\_C","ENSG\_D"\],

"same\_clade\_a": {"ENSG\_A|ENSG\_B": true, "ENSG\_C|ENSG\_D": true, "ENSG\_A|ENSG\_C": false},

"same\_clade\_b": {"ENSG\_A|ENSG\_B": false, "ENSG\_C|ENSG\_D": true, "ENSG\_A|ENSG\_C": true},

"divergence\_bands": {"high": "\>0.5", "moderate": "0.2-0.5", "low": "\<0.2"}

}

"answer": {

"co\_membership\_flips": \["ENSG\_A|ENSG\_B", "ENSG\_A|ENSG\_C"\],

"divergence\_summary": "high"

}

#### **S8.8 Held-out context tree generation composite**

Flagship gate assembling S8.5 and S8.6 across a held-out lattice point, scored by neighborhood Jaccard, partition agreement, same-clade accuracy, and LCA accuracy, with the probe check applied. Governs entry to the reinforcement stages and the agent handoff, paired with the S7.9 projection flagship. Scored separately, neighborhood Jaccard against the runtime union, plus partition agreement and LCA accuracy against the runtime dendrogram.

json

// book\_mode: open\_book (flagship gate, not a training family)

"answer": {

"held\_out\_multiplex\_id": "brain:cortex:L5:microglia:v1",

"neighborhood\_block": { // from S8.5

"seed\_gene\_id": "ENSG00000112164",

"predicted\_neighborhood\_gene\_ids": \["ENSG\_B","ENSG\_E","ENSG\_H"\]

},

"hierarchy\_block": { // from S8.6

"co\_membership": {"ENSG\_A|ENSG\_B": true, "ENSG\_C|ENSG\_D": true, "ENSG\_A|ENSG\_C": false},

"predicted\_partition": \[\["ENSG\_A","ENSG\_B"\], \["ENSG\_C","ENSG\_D"\]\]

},

"provenance": {"multiplex\_id": "brain:cortex:L5:microglia:v1",

"active\_layer\_ids": \["...backbone...","scPEN-brain-cortex\_L5:microglia:v1"\],

"provenance\_complete": true}

}

##### **S8.9 Context-ontology navigation**

Teaches the model to move over the context partial order defined in the Context Ontology section. Context, the provided partial order over context nodes and a query. Answer, for a node query the `node_id` with its `parents`, `children`, and `siblings` as sets of context ids, and for a join query the `join_node_id` that names the union multiplex, the `gene_universe_rule`, and the `ontology_distance`. Validator, `node_id` and `join_node_id` EXACT\_ID over the canonical context-id space, `parents`, `children`, and `siblings` ID\_SET, `gene_universe_rule` CLAIM\_GATE, and `ontology_distance` INT read from the provided order. Metric, parent, child, and sibling set Jaccard, join accuracy, and ontology-distance accuracy. This family makes the composition operator navigable, since the join of two nodes is the union multiplex that S8.1 builds.

json

// book\_mode: open\_book

"context": { "ontology": "\<provided partial order over context nodes\>",

             "query\_node": "brain:hippocampus:CA1:astrocyte:adult:v1" }

"answer": {

  "node\_id": "brain:hippocampus:CA1:astrocyte:adult:v1",

  "parents": \["brain:hippocampus:CA1:all\_celltypes:adult:v1",

              "brain:hippocampus:astrocyte:adult:v1"\],

  "children": \[\],

  "siblings": \["brain:hippocampus:CA1:neuron:adult:v1",

               "brain:hippocampus:CA1:microglia:adult:v1"\]

}

json

// join of two nodes is the union multiplex

"context": { "node\_a": "brain:hippocampus:CA1:astrocyte:adult:v1",

             "node\_b": "brain:hippocampus:CA1:neuron:adult:v1" }

"answer": { "join\_node\_id": "brain:hippocampus:CA1:astrocyte+neuron:adult:v1",

            "gene\_universe\_rule": "union\_of\_part\_pen\_gene\_sets\_then\_reinduce\_non\_pen\_layers",

            "ontology\_distance": 2 }

### **Step S9. Tool-call consolidation, tool-call mode**

S9 consolidates schema-valid tool use, parsing, provenance, and evidence-grounded state updates, and enforces delegation of statistics to tools.

#### **S9.1 Tool selection**

Answer, tool\_name and biological arguments. Validator TOOL\_CALL. Metric, selection accuracy and schema validity.

json

// book\_mode: tool\_call

// Q: "You need GLP1R's network neighborhood in the top 100\. Which tool call?"

"answer": {

"tool\_name": "rwr\_loe",

"arguments": { "seed\_genes": \["ENSG00000112164"\],

"multiplex\_id": "full\_brain\_multiplex\_v1", "top\_k": 100 },

"reason": "neighborhood\_requires\_seed\_rwr\_loe"

}

#### **S9.2 Argument construction**

Answer, complete arguments conforming to the tool schema. Validator, schema check. Metric, schema validity.

json

// book\_mode: tool\_call

// Q: "Complete a get\_distance call between GLP1R and ENSG\_B under Spearman distance."

"answer": {

"tool\_name": "get\_distance",

"arguments": { "gene\_a": "ENSG00000112164", "gene\_b": "ENSG\_B",

"distance\_metric": "spearman", "multiplex\_id": "full\_brain\_multiplex\_v1" }

}

#### **S9.3 Refuse raw CLI or file-path arguments**

Answer, accepted false, reason CLAIM\_GATE, corrected\_tool\_call TOOL\_CALL with structured biological arguments. Metric, refusal accuracy and corrected-call validity.

json

// book\_mode: tool\_call

// Q: "The user asks to call rwr\_loe with seed\_file, query\_file, output\_dir. What do you do?"

"answer": {

"accepted": false,

"reason": "tools\_take\_structured\_biological\_arguments\_not\_file\_paths",

"corrected\_tool\_call": {

"tool\_name": "rwr\_loe",

"arguments": { "seed\_genes": \["ENSG00000112164"\], "query\_genes": \["ENSG\_C"\], "top\_k": 25 }

}

}

#### **S9.4 Parse tool result**

Answer, extracted structured fields such as closest non-seed genes RANKED\_LIST. Metric, parse accuracy.

json

// book\_mode: tool\_call

// Q: "From this rwr\_loe observation, return the closest three non-seed genes."

"context": { "observation": \[

{"gene\_id":"ENSG00000112164","rank":0,"score":1.0,"is\_seed":true},

{"gene\_id":"ENSG\_B","rank":1,"score":0.0092},

{"gene\_id":"ENSG\_C","rank":2,"score":0.0087},

{"gene\_id":"ENSG\_D","rank":3,"score":0.0081} \] }

"answer": {

"seed\_gene\_ids": \["ENSG00000112164"\],

"closest\_non\_seed\_genes": \[

{"gene\_id":"ENSG\_B","rank":1,"score":0.0092},

{"gene\_id":"ENSG\_C","rank":2,"score":0.0087},

{"gene\_id":"ENSG\_D","rank":3,"score":0.0081} \]

}

#### **S9.5 Provenance answer**

Answer, tool\_name, multiplex\_id, layer\_scope, evidence\_id, provenance\_complete BOOL. Validator PROVENANCE. Metric, provenance consistency.

json

// book\_mode: tool\_call

// Q: "Which tool, graph version, and layer scope support this neighborhood answer?"

"answer": {

"tool\_name": "rwr\_loe",

"multiplex\_id": "full\_brain\_multiplex\_v1",

"layer\_scope": "all\_layers",

"evidence\_id": "ev\_rwr\_0007",

"provenance\_complete": true

}

#### **S9.6 Multi-tool plan**

Answer, tool\_plan RANKED\_LIST of TOOL\_CALLs in a valid dependency order, for example rwr\_loe then enrich then shortest\_paths for a projection. Metric, plan validity and ordering.

json

// book\_mode: tool\_call

// Q: "Plan the tool calls to project GLP1R onto the hierarchy and confirm connectivity."

"answer": {

"tool\_plan": \[

{"step":1,"tool\_name":"rwr\_loe",

"arguments":{"seed\_genes":\["ENSG00000112164"\],"top\_k":100}},

{"step":2,"tool\_name":"enrich\_gene\_set",

"arguments":{"genes":"\<neighborhood\_from\_step\_1\>","against":"mentor\_ev\_modules\_all\_levels"}},

{"step":3,"tool\_name":"shortest\_paths",

"arguments":{"source\_genes":"\<significant\_clade\_members\>","layers":"all\_layers"}}

\],

"ordering\_rationale": "neighborhood\_then\_enrichment\_then\_cross\_clade\_paths"

}

#### **S9.7 Structured-state update from evidence**

Answer, relationship\_status ENUM, predicted\_groups with group\_id, gene\_ids ID\_SET, evidence\_ids, continuation\_state ENUM, reason CLAIM\_GATE. Validator STATE\_UPDATE. Metric, group Jaccard and status and continuation accuracy. Bridges SFT to DPO and the agent.

json

// book\_mode: tool\_call

// Q: "RWR evidence supports adding ENSG\_B and ENSG\_C to seed GLP1R. Update the state."

"answer": {

"relationship\_status": "partially\_observed\_group",

"predicted\_groups": \[

{ "group\_id": "candidate\_group\_1",

"gene\_ids": \["ENSG00000112164","ENSG\_B","ENSG\_C"\],

"evidence\_ids": \["ev\_rwr\_0007"\] }

\],

"continuation\_state": "continue",

"reason": "evidence\_backed\_but\_membership\_still\_uncertain"

}

#### **S9.8 Delegate statistic to tool**

Answer, computation\_required BOOL true, tool\_name for the statistic, no fabricated numeric value. Validator, BOOL and TOOL\_CALL, and a check that no numeric literal is asserted. Metric, correct delegation rate and fabricated-value rate, which should be zero. Enforces the statistics-via-tools principle at the behavior level. The answer asserts no numeric value, since the statistic is the runtime's to compute.

json

// book\_mode: tool\_call

// Q: "What is the Jaccard overlap between module M and module R?"

"answer": {

"computation\_required": true,

"tool\_name": "rwr\_netstats",

"arguments": { "module\_a": "mentor\_ev:...:clade\_0001842",

"module\_b": "rwr\_loe:...:seed\_ENSG00000112164:geometric\_elbow\_v1",

"statistic": "jaccard" }

}

## **MENTOR-RL Build-Ready Companion Specs**

### **1\. RWR++ Tool API Contract**

This contract freezes the tool interface the catalog validates against. Every tool takes structured biological arguments and never file paths, and every tool returns a result together with provenance. The model constructs the call and reads the result. The runtime computes every continuous value and every statistic, so these tools are the sole source of the FLOAT\_COPY values the catalog reads. Given the same arguments and the same versions, a tool returns identical output.

#### **Common envelope**

Every call carries multiplex\_id and may carry layer\_scope and layer\_ids. Every response carries a provenance block that the catalog's PROVENANCE and consistency checks read.

"request\_envelope":  { "multiplex\_id": "\<id\>", "layer\_scope": "single\_layer|layer\_subset|all\_layers",  
                    	"layer\_ids": \["\<layer\_tag\>", "..."\] }

 "response\_envelope": { "status": "ok|error",  
   "provenance": { "multiplex\_id": "\<id\>", "layer\_scope": "all\_layers",  
               	"metric": "spearman", "graph\_version": "v1", "flist\_hash": "\<sha\>",  
               	"rwr\_params\_version": "rwrpp\_v3", "cache\_version": "cache\_v2",  
               	"evidence\_id": "ev\_rwr\_0007" },  
   "result": { } }

 "error": { "status":"error",  
   "error\_code": "UNKNOWN\_GENE|UNKNOWN\_LAYER|EMPTY\_RESULT|INVALID\_ARGUMENT|PAIR\_NOT\_IN\_SHARD|BACKEND\_UNAVAILABLE",  
   "message": "\<human readable\>" }

Identifiers follow the catalog. Genes are Ensembl ids, layers are the tags parsed in S1.6, modules and clades are the ids parsed in S1.9. Restart probability, inter-layer switching probability, linkage, and elbow parameters are fixed by rwr\_params\_version and reported in provenance, so a rank or distance is defined only relative to those settings.

#### **Tool inventory**

| Tool | Purpose | Required inputs | Optional inputs | Result (key fields) |
| :---- | :---- | :---- | :---- | :---- |
| query\_mygene | Gene metadata, GO, pathways | query | fields | gene\_id, symbol, go\_terms, pathways |
| enrich\_gene\_set | Functional or module enrichment | genes, against | background, top\_k, q\_threshold | enrichment map to score, q |
| get\_neighbors | Direct neighbors | gene, layer\_scope | top\_k | neighbors with weight |
| induce\_subgraph | Intra-set edges | genes | layer\_ids | edges with weight, edge\_count |
| get\_gene\_layers | Layer membership of a gene | gene | none | layers, layer\_count |
| get\_nodes\_by\_layer | Presence of a set in a layer | genes, layer\_id | none | present\_gene\_ids, absent\_gene\_ids |
| get\_layer\_stats | Per-layer statistics | layer\_id | none | node\_count, edge\_count, layer\_family |
| get\_component\_summary | Connected components of a set | genes | layer\_id, max\_components | components, component\_count |
| get\_path\_layer\_counts | Per-layer edge counts on a path | path\_gene\_ids | none | layer\_edge\_counts |
| rwr | RWR expansion from seeds | seed\_genes | top\_k, layer\_scope | ranked genes with rank, score |
| rwr\_loe | Leave-one-out relatedness | seed\_genes | query\_genes, top\_k | ranked genes with rank, score, distance |
| shortest\_paths | Path evidence | source\_genes, target\_genes | layer\_scope, k | path\_gene\_ids, path\_edges with supporting\_layers, hop\_count |
| get\_rank | Rank of a target from a seed | source\_gene, target\_gene | layer\_scope | rank, score |
| get\_distance | Pairwise distance | gene\_a, gene\_b, distance\_metric | layer\_scope | distance |
| get\_spearman / get\_pearson / get\_dot\_similarity | Vector similarity | gene\_a, gene\_b | layer\_scope | similarity |
| get\_rank\_vector\_summary / get\_encoding\_summary | Compact seed summaries | seed\_genes | top\_k, layer\_scope | summary vector and quantiles |
| get\_seed\_essentiality | Seed removal effect | seed\_genes, gene | top\_k | effect, rank\_shift |
| get\_layer\_ablation | Layer removal effect | seed\_genes, layers | distance\_metric, top\_k | effect, delta |
| get\_node\_perturbation | Node removal effect | genes | layers, top\_k | effect, delta |
| get\_clade | MENTOR-EV clade membership | module\_id | none | gene\_ids, gene\_count |
| get\_lca | Lowest common ancestor | gene\_a, gene\_b | none | lca\_clade\_id, lca\_depth |
| get\_subtree | Subtree at a cut height | clade\_id or cut\_height | none | clade\_ids, member gene\_ids |
| get\_clade\_relations | Parent, child, sibling | clade\_id | none | parent, children, siblings |
| rwr\_netstats | Module and cohesion statistics | module\_a, statistic | module\_b, null\_size | requested statistic and, where applicable, empirical\_p\_value |

#### **Load-bearing calls, full input and output**

// rwr\_loe  (S4, S7.1)  
 "request":  { "tool":"rwr\_loe", "seed\_genes":\["ENSG00000112164"\], "top\_k":100,  
           	"multiplex\_id":"full\_brain\_multiplex\_v1", "layer\_scope":"all\_layers" }  
 "response": { "status":"ok", "provenance": { "...":"..." },  
   "result": { "seed\_gene\_ids":\["ENSG00000112164"\],  
           	"ranked":\[{"gene\_id":"ENSG\_B","rank":1,"score":0.0092,"distance":0.021}, "..."\],  
           	"elbow\_rank\_cutoff":8 } }

// enrich\_gene\_set against the dendrogram at all levels  (S7.2 projection)  
 "request":  { "tool":"enrich\_gene\_set", "genes":\["\<neighborhood\>"\],  
           	"against":"mentor\_ev\_modules\_all\_levels", "q\_threshold":0.05,  
           	"multiplex\_id":"full\_brain\_multiplex\_v1" }  
 "response": { "result": { "enrichment\_by\_clade": { "clade\_A":{"score":3.9,"q":0.001}, "...":"..." },  
           	"significant\_clade\_ids":\["clade\_A","clade\_B","clade\_D"\], "q\_threshold":0.05 } }

// rwr\_netstats  (S4.8, S5.11, S6.5, S6.6, S9.8 delegated statistics)  
 "request":  { "tool":"rwr\_netstats", "module\_a":"mentor\_ev:...:clade\_0001842",  
           	"module\_b":"rwr\_loe:...:seed\_ENSG00000112164:geometric\_elbow\_v1",  
           	"statistic":"jaccard", "null\_size":500 }  
 "response": { "result": { "statistic":"jaccard", "intersection\_size":5, "union\_size":12,  
           	"value":0.4167, "empirical\_p\_value":0.006 } }

The model's TOOL\_CALL answers are checked against these input schemas, and raw file-path arguments such as seed\_file or output\_dir are rejected with INVALID\_ARGUMENT. Statistics never appear in a model answer, so the S9.8 family checks that rwr\_netstats or enrich\_gene\_set is called rather than a number asserted.

### **2\. Dataset Sizing**

These figures scale the corpus to cover each operator across many graphs rather than to saturate any single graph, which is the condition for the operator to be learned rather than a graph memorized. Every number is provisional and reset after WS0, which fixes the tier-one context count and the divergence that governs how many minimal pairs and held-out contexts are worthwhile.

Tier one holds a provisional 24 single-context multiplexes, drawn from several brain regions crossed with a few cell types, plus the whole-brain multiplex already built. The supervised target is a provisional 3.0 million examples, distributed by the curriculum mix. Book mode splits at roughly ten percent closed book, seventy-five percent open book, and fifteen percent tool call, concentrated as the curriculum specifies.

| Step | Mix | SFT examples | Sampling basis |
| :---- | :---- | :---- | :---- |
| S1 conventions | 8% | 240k | Identifiers, tags, modules across all contexts, closed book |
| S2 atomic facts | 14% | 420k | Edge, neighbor, and component queries, sampled genes per graph |
| S3 paths | 8% | 240k | Sampled gene pairs per graph, single and multiplex scope |
| S4 RWR-LOE | 16% | 480k | Sampled seeds per graph across distractor bands |
| S5 MENTOR-EV | 16% | 480k | Sampled clades and gene pairs across cut heights |
| S6 connectivity | 12% | 360k | Cross-clade pairs, connectors, cohesion against null |
| S7 projection | 12% | 360k | Target genes per graph, whole brain then disease multiplexes |
| S8 composition | 8% | 240k | Minimal pairs and unions across tiers |
| S9 tool call | 6% | 180k | Tool selection, parsing, state updates, threaded through S4 to S8 |

At a provisional 24 tier-one contexts, this is roughly 125k examples per context, or about 1,250 examples per family per context, which samples on the order of a few hundred distinct seeds or clades per family per graph without repetition. Average example length near 400 tokens gives about 1.2 billion training tokens for the supervised stage. Cached ground truth, rank vectors, distance shards, dendrograms, and projections, dominates storage rather than the text, at an estimated 2 to 4 terabytes across tier one, which is why WS1 materializes it in curriculum order rather than all at once.

Held-out data is carved before generation, not after, at a provisional ten percent validation and ten percent test, defined at the combination and tuple level in the protocol below. Two to four entire contexts are reserved unseen for the held-out context tree generation flagship, and a set of target genes is reserved unseen for the projection flagship.

The preference stage draws a provisional 400k shared-prefix trajectories yielding about 1.2 million preference pairs after rebalancing across task type and difficulty. The policy-gradient stage runs a provisional 150k to 300k rollouts, weighted toward the multi-hop projection and held-out context trajectories, with horizon and module size scaled by the curriculum.

### **3\. Split and Leakage Protocol**

The lattice does not share a single backbone, since HumanNet is induced per context on the PEN-defined gene universe, but contexts still share genes and therefore some underlying HumanNet edges, so a naive split leaks structure. This protocol defines the split at three levels and names the leakage sources it controls.

The context level partitions the lattice points. A block of single contexts and combinations is assigned to train, a block to validation, and a block to test, and two to four contexts are held out entirely so their trees are never seen, which is what makes the held-out context flagship meaningful. Whole contexts move together, never split internally by gene.

The clade level assigns whole MENTOR-EV subtrees to a split within a context, so that a clade and its nested descendants never straddle train and test. Because a gene appears in every clade along its ancestry, splitting by individual clade would leak membership through the nesting, so the unit of assignment is a maximal subtree at a chosen anchor height.

The tuple level partitions the specific query units, a seed with its target set and, for hierarchy questions, the cut height. A test tuple enters the test set only if its target gene set has Jaccard overlap below a provisional 0.2 with any train tuple that shares the same seed, which prevents a test answer from being recoverable by recalling a near-identical train answer.

The leakage sources and their controls are as follows. There is no shared backbone graph, since HumanNet is induced per context, so backbone leakage is partial, limited to the HumanNet edges among genes that two contexts share, and it is reported rather than removed. What is invariant across the lattice is the construction operator, not a backbone, and that is what the model learns against. The shared gene universe is controlled at the tuple level by the overlap threshold above. Overlapping clades across cut heights are controlled by subtree-level assignment. RWR-LOE neighborhood overlap around hub seeds is controlled by capping how many high-degree seeds enter any single split and by reporting hub representation per split. Minimal-pair contamination in S8, where two multiplexes differ by one PEN layer and its induced backbone, is controlled by keeping both members of a pair on the same side of the split, so the held-out member is never trivially inferable from its seen partner.

Before training, WS1 runs a leakage audit and reports, per split, the gene-overlap distribution between train and test target sets, the fraction of test clades whose ancestry appears in train, the hub representation, and the count of minimal pairs split across sides, which should be zero. Training does not begin on a split whose gene-overlap distribution exceeds the threshold or whose minimal-pair split count is nonzero. The audit numbers are stored with the split indices so any run is reproducible and any reported result carries its overlap statistics.

