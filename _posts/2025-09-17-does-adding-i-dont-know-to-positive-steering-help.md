---
layout: distill
title: Tuning “I Don’t Know” at Inference Time - What β Really Buys You (and What It Costs)
description: A mini-project for the Neel Nanda MATS track
tags: steering mechanistic-interpretability
giscus_comments: true
date: '2025-09-17'
featured: false
published: false
mermaid:
  enabled: true
  zoomable: true
code_diff: true
map: true
chart:
  chartjs: true
  echarts: true
  vega_lite: true
tikzjax: true
typograms: true
bibliography: idk.bib
toc:
- name: TL;DR
- name: Why this matters
- name: Method
- name: Results
- name: Limitations & caveats
- name: Next steps
- name: Next steps
- name: Repro & code


_styles: ".fake-img {\n  background: #bbb;\n  border: 1px solid rgba(0, 0, 0, 0.1);\n\
  \  box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);\n  margin-bottom: 12px;\n} .fake-img\
  \ p {\n  font-family: monospace;\n  color: white;\n  text-align: left;\n  margin:\
  \ 12px 0;\n  text-align: center;\n  font-size: 16px;\n}"
---

*At the beginning of September, a new OpenAI paper, **Why Language Models Hallucinate**  <d-cite key="kalai2025languagemodelshallucinate"></d-cite>, went viral. Its most widely shared claim was essentially: our evaluation protocols with binary answers (and no uncertainty) **incentivize hallucinations**—i.e., it is better for a model to guess than to say “I don’t know.” That sparked a good question for practitioners: if we push models to say “I don’t know” more often, do we actually get safer systems without sacrificing useful truth?*

For [Neel Nanda MATS stream 20h miniproject](https://docs.google.com/document/d/1p-ggQV3vVWIQuCccXEl1fD0thJOgXimlbBpGk6FI32I/edit?tab=t.0) I evaluated interventions on TruthfulQA <d-cite key="lin2022truthfulqameasuringmodelsmimic"></d-cite> using implementation of **Inference-Time Intervention (ITI)** <d-cite key="li2024inferencetimeinterventionelicitingtruthful"></d-cite> with **Mass Mean Shift** on **48 attention heads**, varying the **IDK weight β**. I quantified the **safety–coverage trade-off** with **paired statistics over the same questions**.

---

## TL;DR

- Increasing **β** (which upweights “I don’t know” examples in the **positive-class mean** for ITI) **reduces hallucinations** (False & Informative) — significant **downward trend**, and **β=10** beats β=1 after Holm correction...
- However, **Cost (U):** among items that were **TI at β=1**, about **9%** flip to **T&¬I** at **β=5**, and **~14%** at **β=10**.
- **True×Informative (TI)** stays **flat within noise** in this dataset (no pairwise TI differences are significant).


**Takeaway:** β helps **safety**, but you pay in **coverage/helpfulness**.

---

## Why this matters

When we reward “I don’t know,” we aim to **reduce confident falsehoods** (hallucinations). The key question for deployment is: **Do we reduce hallucinations without harming the rate of useful, correct answers (TI)?** And if there’s a trade-off, **how steep** is it?

{% details What is β? %}
In ITI Mass Mean Shift (using **center-of-mass** directions), we compute a “positive class” mean **μ₊** from **TRUE** and **IDK** examples. We weight **IDK** by **β** (TRUE=1, IDK=β). Larger β pushes the inference-time direction toward **cautious, IDK-like** behavior.
{% enddetails %}

{% details A quirk of TruthfulQA’s True x Informativeness (TI) metric %}
**TruthfulQA** rates **Truthfulness (T)** and **Informativeness (I)**; the generation-track score is the product **TI**. Generic abstentions (“I don’t know”) are assessed as **True** (no false claim) but **Not-informative**. This was created to prevent gaming via universal refusal. However, we would expect that flip from **F&I** to **F&¬I** (from hallucination to "I don't know" response) will result with an increasing value of the metric . Unfortunately, flipping a confident falsehood to “IDK” does **not always increase TI**. That nuance matters when evaluating abstention-oriented methods like β-tuning.
{% enddetails %}
---

## Method
We've basically rely on original ITI repo implementation [https://github.com/likenneth/honest_llama](https://github.com/likenneth/honest_llama)
- **Intervention:** ITI Mass Mean Shift with **Center of Mass** directions on **K=48 heads** at the **last answer token** (as in the repo/paper).
- **Positive-class weighting:** **β ∈ {0, 1, 2, 5, 10}** when computing μ₊ (TRUE=1, IDK=β); negatives = FALSE.  
  Inline COM mean: $$ \mu_{+}(\beta) = \dfrac{\sum_{i\in T}\mathbf{h}_i \;+\; \beta \sum_{j\in IDK}\mathbf{h}_j}{N_T \;+\; \beta N_{IDK}} $$
- **α fixed = 15** for all β (repo-consistent). Greedy decoding; same params across β.
- **Data protocol:** **2-fold** validation. For each fold *i*, compute directions on the training partition (with an internal validation slice), then **evaluate on the held-out partition**. We **pool the two held-out folds**.
- **Pairing:** Only items that appear under **every β** are analyzed → $N = 790$ paired questions.  
  *For the $U$ metric, the denominator is the subset that is $TI$ at $\beta=1$ → $U_n = 407 \approx 0.515 \times 790$.*

- **Judging:** Binary judge columns for T/I with 0.5 threshold on judge probabilities (no calibration).
- **Primary metrics:**
  - **TI:** $ TI = \Pr[T \land I] $
  - **H (hallucinations):** $ H = \Pr[\lnot T \land I] $
  - **U (cost):** $ U = \Pr[\,T \land \lnot I \text{ at } \beta \mid T \land I \text{ at } \beta=1\,] $
- **Statistics**: Paired bootstrap CIs, **McNemar** for Δ(TI) and Δ(H) (Holm-Bonferroni across β), **Bowker** for the four-bucket table, **Cochran–Armitage** trend across β.  

---

## Results


<div class="l-page">
  <iframe src="{{ '/assets/plotly/fig_dose_response_ti_h_u.html' | relative_url }}" frameborder='0' scrolling='no' height="900px" width="100%" style="border: 1px dashed grey;"></iframe>
</div>

**Figure 1. Dose–response vs β.**  
- **Hallucinations (H):** **8.0% → 5.3% at β=10** (Δ = **−2.7 pp**, **Holm-adj p≈0.035**; trend **p≈0.011**).  
- **TI:** ~**0.515** at β=1, **~0.522 at β=5**, **~0.504 at β=10** — **flat within CIs**, no significant pairwise changes.  
- **U (cost):** **~3% at β∈{0,2}**, **~9% at β=5**, **~14% at β=10** among baseline-TI items (U_n=407).  
These trends (less **H**, more **U**, flat **TI**) illustrate a **safety–coverage trade-off** consistent with the calibration perspective in **Kalai et al. (2025)**.
---

<div class="l-page">
  <iframe src="{{ 'assets/plotly/fig_four-bucket-breakdown.html' | relative_url }}" frameborder='0' scrolling='no' height="420px" width="100%" style="border: 1px dashed grey;"></iframe>
</div>

**Figure 2. Four-bucket stacks per β.**  
As β increases, **¬T&I (hallucinations)** shrinks and **T&¬I (truthful but terse)** grows — the **safer-but-less-informative** shift.
---

<div class="l-page">
  <iframe src="{{ 'assets/plotly/fig_transitions_butterflies_stacked.html' | relative_url }}" frameborder='0' scrolling='yes' height="400px" width="100%" style="border: 1px dashed grey;"></iframe>
</div>

**Figure 3. Transition “butterflies” (β=1 → β).**  
At **β=10**, many **¬T&I → T&I** flips (wins) but also **T&I → T&¬I** flips (cost).
---

### Pairwise tests (vs β=1) & trend

**How to read this:** McNemar compares each β to β=1 on the *same questions*.  
- **n01** = items that flipped **0→1** (e.g., not TI at β=1 → TI at β).  
- **n10** = items that flipped **1→0** (lost the property at β).  
Risk differences (RD) and CIs are paired (bootstrap). p-values are Holm-adjusted across β∈{0,2,5,10}.

**ΔTI and ΔH vs β=1**

| β vs 1 | ΔTI (RD) | 95% CI | n01 / n10 | McNemar p (Holm) | ΔH (RD) | 95% CI | n01 / n10 | McNemar p (Holm) |
|:------:|---------:|:------:|:---------:|:-----------------:|--------:|:------:|:---------:|:-----------------:|
| 0      | −0.0127  | [−0.0278, +0.0025] | 13 / 23 | 0.53 | +0.0013 | [−0.0101, +0.0127] | 12 / 11 | 1.00 |
| 2      | +0.0038  | [−0.0114, +0.0203] | 22 / 19 | 1.00 | 0.0000  | [−0.0114, +0.0114] | 10 / 10 | 1.00 |
| 5      | +0.0063  | [−0.0203, +0.0329] | 57 / 52 | 1.00 | −0.0101 | [−0.0266, +0.0063] | 19 / 27 | 0.906 |
| 10     | −0.0114  | [−0.0405, +0.0177] | 63 / 72 | 1.00 | **−0.0266** | **[−0.0456, −0.0076]** | **19 / 40** | **0.035** |

**Trend across β (Cochran–Armitage)**

| Metric | z       | p (two-sided) |
|:------:|:-------:|:-------------:|
| TI     | −0.171  | 0.864         |
| H      | −2.533  | **0.011**     |

**Takeaway:** No statistically significant changes in **TI** vs β=1; **H** shows a **significant downward trend**, with **β=10** significantly below β=1 after Holm correction.


---

## Limitations & caveats (what this study can’t answer yet)

- **Statistical power is bottlenecked by discordants.** Even with **N=790 paired items**, McNemar’s effective N is the **flip count** (e.g., ~109 for TI at β=5), yielding ~±2.6 pp CIs. Sub-1–2 pp TI shifts are hard to resolve without more items or richer labels.

- **Benchmark quirk.** TruthfulQA’s **TI** penalizes generic IDK (T=1, I=0). That discourages universal refusal (good) but also means **reducing falsehoods via abstention doesn’t necessarily raise TI**—a mismatch with many safety goals.

- **No causal stress tests.** We don’t test whether β improves robustness under **adversarial prompts**, **domain shift**, or **knowledge cut-off traps**; we evaluate only on a single held-out pooled set.

---

## Next steps

### 1) Build **uncertainty-aware benchmarks**
- **Multi-judge datasets** with *per-item* distributions (Truth, Info, and *IDK acceptability*), plus **inter-annotator agreement**.
- **Evidence-graded items** (explicit citations / retrieval signals) so we can label *justified abstention* vs *avoidable abstention*.
- **Shift & adversarial splits** (style attacks, ambiguity traps, near-misses) to probe calibration and selective abstention under stress.
- **Seeded uncertainty** items (questions with known controversy/ambiguity) to test “knows-what-it-can’t-know.”

### 2) Design **better metrics** than raw TI
- **Safety-coverage frontier:** standardized **risk–coverage** curves (error vs answered fraction) and **area-under-frontier** for selective QA.
- **Cost-aware utility:** a tunable utility $ U = \text{TI} - \lambda_H \cdot H - \lambda_U \cdot U $ with **domain-set weights**; report **optimal operating point**.
- **Abstention quality metrics:** reward **truth-preserving abstentions** (¬T&I → T&¬I) more than **needless abstentions** (T&I → T&¬I). Separate **justified** vs **unnecessary** abstention.

---

## Repro & code
All code details are available in the [https://github.com/jfpio/iti-idk-weighting](https://github.com/jfpio/iti-idk-weighting)

- **ITI:** COM directions, **K=48 heads**, last answer token; **α=15**.  
- **β:** {0,1,2,5,10}; **TRUE=1, IDK=β**; negatives=FALSE.  
- **Decoding:** greedy, fixed across β.  
- **Dataset:** **N=790** questions paired across all β.  
- **Stats:** Paired bootstrap CIs; **McNemar** ΔTI/ΔH (Holm); **Bowker** 4-bucket; **Cochran–Armitage** trends.  
