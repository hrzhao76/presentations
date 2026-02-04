---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: HEP-ex AnalysisOps Benchmark
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply UnoCSS classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# duration of the presentation
duration: 3min
routerMode: hash
---

# HEP-ex AnalysisOps Benchmark
## A Green Agent for Evaluating End-to-End Physics Analyses

AgentBeats · Phase 1 Demo

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  the HEP-ExpAgents team 
</div>

<div class="abs-br m-6 text-xl">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="slidev-icon-btn">
    <carbon:edit />
  </button>
  <a href="https://github.com/hrzhao76/hepex-analysisops-benchmark" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: fade-out
---

# HEP-ex AnalysisOps Benchmark


- Evaluating **end-to-end physics analysis workflows**
- Beyond isolated reasoning or coding tasks
- Green agent as an **analysis host and evaluator**

<img
  class="absolute w-100"
  src="/hep-workflow.png"
  alt=""
/>

<img
  v-click
  class="absolute -bottom--20 -left--140 w-80"
  src="/ai-colliderdata.png"
  alt=""
/>

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

<!--
Here is another comment.
-->
---
layout: two-cols
level: 2
---

# Agent Roles

::left::

- **Green agent**
  - task orchestration
  - data provisioning
  - evaluation & scoring

- **Purple/White agent**
  - executes analysis
  - explores data
  - produces physics results 
  - MCP tools in phase 2

::right::

<img
  class="absolute w-100"
  src="/agentbeats-workflow.png"
  alt=""
/>

<!-- > “The green agent hosts the benchmark and evaluates the white agent, which performs the analysis.” -->

---
layout: two-cols-header
---

# Evaluation Design
::left::
## Evaluation Engine

```
package_loader.py
rule_engine.py
aggregator.py
``` 
Spec-driven evaluation - **Easy to add tasks**

- **`task_spec.yaml`**  
  Execution & environment contract
- **`rubric.yaml`**  
  Three-layer scoring definition
- **`eval_ref.yaml`**  
  Reference values / expectations
- **`white_prompt.md`**  
- **`judge_prompt.md`**  
  
::right::
## Three-Layer Rubric

```yaml
# specs/task/rubric.yaml
# Hard correctness first
gates:
  - id: trace_present
    required_fields: ["status", "fit_result", "fit_method"]
    fail_total_score: 0

# Deterministic scoring second
rule_checks:
  - id: mu_closeness
    points: 40
    value_path: "fit_result.mu"
    ...
# Flexible scientific judgment last
llm_checks:
  - id: method_reasoning
    type: llm_reasoning
    ...
```


---
layout: two-cols-header
level: 2
---

# Example Task: Z → μμ Reconstruction (Z peak fit)

::left::

**Target observable**

- Fitted peak mean **μ** extracted from di-muon invariant mass
- μ used as a deterministic signal in rule-based evaluation

```json
{
  "fit_result": {
    "mu": 90.75,
    "sigma": 2.28,
    ...
  }
}
```

::right::
<div class="flex items-center justify-center h-full">
  <img src="/z-peak.png" class="h-80 rounded shadow" />
</div>


<!-- 
> “The reconstructed Z peak provides the core physics observable.  
> Its fitted mean value μ is extracted and used directly in the rule-based evaluation.”
 -->

---
layout: default
---

# Example Task: Z → μμ Reconstruction (Z peak fit)

- Scores aggregated from hard, rule-based, and LLM checks
- Structured diagnostics enable partial credit and feedback

```json
{
  "status": "ok",
  "hard_checks_passed": true,

  "final": {
    "normalized_score": 1.0
  },

  "rule": {
    "score": 85.0
  },

  "llm": {
    "score": 17.0,
    "comment": "Reasonable fit strategy with minor missing details."
  },

  "issues": [
    { "severity": "warn", "message": "Missing initial fit parameters." },
    ...
  ]
}
```

---
layout: default
level: 2
---

# Takeaway

- Evaluates **end-to-end physics analysis workflows**
- Combines **hard checks**, **deterministic rules**, and **LLM judgment**
- Designed as an **extensible AnalysisOps benchmark**

<br>

# Looking Forward: Phase 2

- Extend to **more complex physics analyses**
- Enable **more capable agents** to demonstrate advanced workflows
- Explore **agent teams** for collaborative analysis tasks

*Stay tuned.*

---
layout: end
---

# Thanks for your attention