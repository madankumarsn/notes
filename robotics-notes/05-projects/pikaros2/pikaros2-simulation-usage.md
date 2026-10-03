---
topic: What pikaros2 actually uses simulation for, vs. sim-based policy/regression evaluation
status: draft
layer: projects
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# What pikaros2 uses simulation for (and what it doesn't)

**One-line idea:**
pikaros2 (a warehouse depalletizing robot stack) already leans on simulation heavily for synthetic training data and smoke-testing, but has no CI and no sim-based policy/regression evaluation pipeline — confidence before release currently comes from rosbag replay and on-site field trials instead, which is a distinct gap from "we don't simulate at all."

**Why it exists:**
A useful reference point for reasoning about where simulation actually adds value in a real perception+manipulation stack: training-data generation, smoke-testing, and policy evaluation are three *different* jobs that simulation can do, and a project can be heavily invested in some of them while having none of another.

**Math / mechanics:**
One quantitative point worth keeping in mind generally: you cannot statistically distinguish a 90%-successful policy from an 80%-successful one with only ~20 hardware trials — the confidence interval at that sample size is far too wide. This is the core quantitative argument for needing many cheap (simulated) trials rather than few expensive (hardware) ones whenever a policy's success rate needs to be measured precisely.

**Code:**
N/A — this note reflects repo-level architecture observations, not specific code.

**Gotchas:**
- Simulation is already load-bearing for **training data**: the shipped perception model is trained on synthetic warehouse imagery spanning thousands of lighting variations — i.e. sim is essential to how the perception model was built, just not to how it's *evaluated* before release.
- A real stack can run **several different simulators simultaneously**, each covering a different job, precisely because none of them covers every need on its own: a physics engine for nav/control integration, a separate (e.g. game-engine-based) warehouse-scene simulator for perception/visual realism, fake-hardware stand-ins for interface-level testing (no physics), and a hand-rolled, physics-free logic simulator for comparing high-level decision strategies (e.g. pick-ranking) without needing any of the heavier simulators at all.
- **Real confidence before release can come from bag replay and field trials, not simulation**, when there is no sim-based policy evaluation pipeline and no CI running automated regression tests — worth explicitly separating "do we use simulation" from "do we have automated, sim-based confidence before shipping a change," since a project can answer yes to the first and no to the second.
- A learned-perception-front-end-plus-hand-written-downstream-logic architecture (e.g. a detector feeding typed data structures into rule-based/geometric reasoning, rather than an end-to-end learned policy) is a genuinely different, and currently more testable/observable, architecture than an end-to-end learned policy — worth naming explicitly as a real design choice, not a gap, when the comparison point is "why don't you use an end-to-end model."

**Related:** [[_projects-hub]]

**Still unclear:**
The specific repo facts behind these observations (exact simulator names/packages, absence of CI, specific model filename) were gathered from a single read of the repo at one point in time and should be independently re-verified before being relied on for any external-facing claim — simulators/tooling can change. The broader "three jobs simulation does" framing and the hardware-trial-sample-size argument are general and durable; the repo-specific inventory is a snapshot.
