# MNEMOSYNE: Savings-Based Recursive Self-Improvement for AI That Learns, Forgets, and Re-Learns Like a Human

**Balamurugan Balakreshnan**
Microsoft
`babal@microsoft.com`

*Concept / position paper · Draft v1 · June 2026*

> **Note on status.** This is a conceptual (position) paper. It proposes an architecture and a research program, including a detailed experimental protocol. It does **not** report measured empirical results; all experiments described in Section 8 are *proposed*, and any quantitative targets are stated as hypotheses to be tested, not findings.

---

## Abstract

Recursive self-improvement (RSI) — AI that improves itself in a feedback loop where each improvement increases its capacity for further improvement — has progressed from Schmidhuber's proof-bound Gödel Machine (2003) to empirically validated, self-editing coding agents such as the Darwin Gödel Machine (DGM) and the Self-Improving Coding Agent (SICA) in 2025. Yet these systems manage *memory* in one of two deficient ways: they either bake knowledge into weights (incurring **catastrophic forgetting**) or accumulate it in ever-growing external stores (never internalizing skills, and scaling poorly). Humans do neither. We learn quickly, **forget gracefully**, and — most importantly — **re-learn forgotten material almost instantly upon brief re-exposure**, a phenomenon Ebbinghaus named the *savings effect*. We argue this third property is the missing ingredient in recursive intelligence. We propose **MNEMOSYNE**, a self-improving agent built around a four-tier, human-like memory (working context, a fast hippocampal episodic store, a slow neocortical consolidated core, and — the key novelty — an **Engram Residue Layer**). Skills that fall out of use are not deleted but **compressed into low-rank "engrams"**; a small reconstructor network plus a few test-time-training steps then re-consolidate them from a warm start when a matching cue ("a quick glance") reappears. We formalize this with a **Savings Ratio** $S = \text{cost(relearn)} / \text{cost(first-learn)} \ll 1$ and pose the central objective as **minimizing $S$ subject to a bounded memory footprint**. Crucially, MNEMOSYNE also **recursively improves its own memory policy** — what to consolidate, what to decay, and how fast to forget — and validates those changes empirically, in the DGM/SICA tradition. We situate the idea against complementary learning systems theory, generative replay, and test-time training, present a falsifiable experimental program, and discuss safety. Our thesis: making forgetting cheap and re-learning nearly free turns the costly explore-and-abandon loop of open-ended self-improvement into a sustainable, lifelong one.

**Keywords:** recursive self-improvement, continual learning, catastrophic forgetting, complementary learning systems, savings effect, test-time training, memory consolidation, self-improving agents.

---

## 1. Introduction

For most of AI's history, humans drove every step of the development cycle. That is changing: AI systems now write, evaluate, and rewrite their own code. The Darwin Gödel Machine raised its score on the SWE-bench software-engineering benchmark from 20.0% to 50.0% by iteratively modifying its own codebase and validating each change empirically [Zhang et al., 2025], and the Self-Improving Coding Agent improved from 17% to 53% on SWE-Bench Verified by editing itself [Robeyns et al., 2025]. These results make recursive self-improvement (RSI) a present-day engineering reality rather than a thought experiment.

But there is a conspicuous asymmetry between these systems and the one biological intelligence we are trying to emulate. A human professional does not retrain from scratch each morning, nor do they carry a verbatim transcript of every experience they have ever had. Instead they **continuously update themselves**: absorbing new information, letting unused details fade, and — the property we focus on — **recovering faded knowledge almost instantly when they encounter it again.** Anyone who has "forgotten" a programming language and then become fluent again within an afternoon has experienced this directly.

This paper makes three claims:

1. **The memory substrate, not raw capability, is the next bottleneck for RSI.** Open-ended self-improvement explores an unbounded design space; without a principled way to forget, its memory and compute costs grow without bound.
2. **Human forgetting is not a defect to be eliminated but a mechanism to be copied.** In particular, the *savings effect* — that relearning is far cheaper than first learning — is precisely what lets humans forget safely.
3. **No existing self-improving system exploits the savings effect**, and doing so requires a specific new component, which we name the **Engram Residue Layer**.

We propose **MNEMOSYNE** (Memory-Native Engram Machine for Open-ended Self-improvement), an architecture that operationalizes these claims, and lay out a falsifiable research program to test it.

### 1.1 Contributions

- **A diagnosis** of the memory dilemma in current RSI systems (weights vs. external store) and why both fail the lifelong-learning test (§3).
- **MNEMOSYNE**, a four-tier human-like memory architecture for a self-improving agent (§5).
- **The Savings Mechanism**: a concrete method — engram compression + reconstructor + test-time re-consolidation — for near-instant relearning of forgotten skills, with a formal **Savings Ratio** objective (§6).
- **Recursive self-improvement of the memory policy itself**, extending DGM/SICA-style self-editing to the forgetting and consolidation rules (§7).
- **A falsifiable experimental protocol** with baselines, benchmarks, metrics, and ablations (§8), plus a safety analysis (§9).

---

## 2. Background and Related Work

### 2.1 The recursive self-improvement lineage

**Gödel Machine** [Schmidhuber, 2003/2007]. The first mathematically rigorous, fully self-referential self-improver: it rewrites any part of its own code once it finds a *proof* that the rewrite is beneficial, and such a rewrite is globally optimal. Its limitation is practical — for almost all real self-modifications, no such proof is obtainable.

**Empirically validated self-improvers.** The **Darwin Gödel Machine** [Zhang et al., 2025] replaces proof with *empirical validation on coding benchmarks*, maintaining an archive of agents and exploring open-endedly; the **Self-Improving Coding Agent (SICA)** [Robeyns et al., 2025] removes the meta-agent/target-agent split so one agent edits itself; **AlphaEvolve** [Novikov et al., 2025] evolves whole algorithms with LLM ensembles and automated evaluators, even improving the infrastructure that trains its own base models. **STOP** [Zelikman et al., 2023] and **Gödel Agent** [Yin et al., 2024] are closely related. All of these improve *capability or scaffolding*; none addresses lifelong memory.

### 2.2 The memory problem these systems inherit

**Catastrophic forgetting** [McCloskey & Cohen, 1989] is the tendency of neural networks to lose performance on old tasks when trained on new ones, because knowledge is stored in shared, overlapping weights. This is the central obstacle to *learning in the weights*.

**Test-time training (TTT).** Recent work learns *during inference*: TT-SI [Acikgoz et al., 2025] detects uncertain cases, synthesizes similar examples, and fine-tunes on them on the fly (reported +5.48% average accuracy with 68× fewer samples), then **resets the parameters after inference**. TTT-E2E [Sun & Choi / NVIDIA, 2026] compresses long context into weights via next-token prediction. These show fast, local weight updates are feasible — but TT-SI explicitly *discards* its adaptation afterward, throwing away exactly the kind of trace we wish to keep.

### 2.3 The biological memory blueprint

**Complementary Learning Systems (CLS)** [McClelland, McNaughton & O'Reilly, 1995; Kumaran, Hassabis & McClelland, 2016]. The brain resolves the *stability–plasticity dilemma* with two systems: a **fast hippocampus** that rapidly encodes pattern-separated episodic memories, and a **slow neocortex** that integrates them into stable, generalized knowledge. During sleep, the hippocampus **replays** experiences so the neocortex can consolidate them.

**Generative replay.** Deep Generative Replay [Shin et al., 2017] argues the hippocampus is "better paralleled with a generative model than a replay buffer," and trains a generator to synthesize past experiences for interleaved rehearsal — mitigating forgetting without storing raw data. Sleep-phase variants (e.g., *NeuroDream*, 2025) add an explicit offline "dream" stage. Modern CLS-inspired continual learners pair a generative component (VAE) with an associative store (Modern Hopfield Network) [Jun et al., 2025].

**The Ebbinghaus savings effect** [Ebbinghaus, 1885]. Memory decays along a *forgetting curve*, but relearning previously studied material is **markedly faster** than learning it the first time. The trace is not erased; a residue persists that makes re-acquisition cheap. This is the human counterpart of the user-facing phenomenon "I forgot it, but a quick glance brought it all back."

### 2.4 The gap

CLS-inspired continual learners fight forgetting but are not *self-improving agents*. Self-improving agents (DGM, SICA) have no human-like memory and no notion of savings. TTT methods adapt at test time but discard the adaptation. **No system combines (a) recursive self-improvement, (b) graceful value-weighted forgetting, and (c) savings-based near-instant relearning.** MNEMOSYNE targets exactly this intersection.

---

## 3. The Memory Dilemma in Recursive Self-Improvement

Consider an agent that improves itself over a long horizon, repeatedly acquiring, abandoning, and revisiting skills. It faces a trilemma:

- **Store skills in weights.** Internalizes skills and is fast at inference, but new learning overwrites old (catastrophic forgetting), and there is no cheap way to "set aside" a skill without losing it.
- **Store skills in an external buffer/archive.** Avoids overwriting (the DGM keeps an archive of agents) but never internalizes skills, must re-read large context each time (cost grows with history), and the archive grows monotonically.
- **Discard adaptations** (as TT-SI does after inference) — cheap, but the agent re-pays the full learning cost every time a situation recurs.

Humans escape the trilemma because forgetting in the brain is **lossy compression with a recoverable residue**, not deletion. The faded trace costs little to keep, yet re-exposure restores the skill at a fraction of the original cost. We contend that *engineering this residue* is the key to sustainable lifelong RSI.

---

## 4. Design Principles

MNEMOSYNE is guided by five principles, each traceable to a biological finding:

1. **Separate fast and slow learners** (CLS): a plastic episodic store and a stable consolidated core.
2. **Consolidate offline via generative replay** (sleep): move durable skills from fast to slow store without re-storing raw data, interleaving regenerated pseudo-experience to prevent forgetting.
3. **Forget on purpose, by value** (forgetting curve): bound memory by decaying low-value, low-use skills — but never to zero.
4. **Preserve savings** (Ebbinghaus): decay leaves a cheap, reconstructable **engram** so relearning is sublinear in the original cost.
5. **Make the memory policy itself improvable** (RSI): the rules governing 1–4 are part of the agent's editable code and are tuned empirically.

---

## 5. The MNEMOSYNE Architecture

MNEMOSYNE comprises four memory tiers and two background processes.

### 5.1 The four memory tiers

**(T1) Working Context (WC)** — *prefrontal working memory.* The live context window: lossless, volatile, and the most expensive per token. Holds only the current task.

**(T2) Episodic Store (ES)** — *hippocampus.* A fast-write, pattern-separated index of recent experiences and freshly acquired skills. Each entry pairs an embedding **key** (a pattern-separated code of the triggering context) with a **value** (a skill represented as a low-rank, LoRA-style adapter $\Delta W = BA$, or a retrievable episode). Writing is cheap; entries have a decaying retention score.

**(T3) Consolidated Core (CC)** — *neocortex.* The base model weights plus a slowly updated **adapter bank** of durable, generalized skills. Changes only during offline consolidation, protecting it from interference.

**(T4) Engram Residue Layer (ERL)** — *the long-term memory trace; the novel tier.* When a skill decays out of ES (or is demoted from CC), it is **not deleted**. It is compressed into an **engram** $\sigma$: a tiny seed consisting of (i) the top-$k$ singular directions of its adapter, (ii) a hash/sketch of its triggering-context key, and (iii) a scalar provenance/value tag. An engram is orders of magnitude smaller than the full adapter, so the ERL can hold a very large number of them at near-zero cost.

### 5.2 The two background processes

**Consolidation ("Sleep").** A scheduler periodically (a) selects high-value, frequently used ES skills, (b) uses a **generative replay** model to synthesize pseudo-experiences for both new and previously consolidated skills, and (c) fine-tunes the CC adapter bank on the interleaved set, moving skills ES → CC without catastrophic forgetting.

**Decay-to-Engram ("Graceful forgetting").** A learned policy assigns each ES/CC skill a **retention score** from usage frequency, recency, estimated future value, and redundancy. Skills below a threshold are compressed to engrams (full adapter dropped, $\sigma$ retained in ERL). This is the value-weighted forgetting curve that bounds the footprint of T2/T3.

A schematic of the flow:

```
  experience ──▶ [T1 Working Context] ──fast write──▶ [T2 Episodic Store]
                                                          │
                              (Sleep: generative replay)  │ consolidate
                                                          ▼
                                                  [T3 Consolidated Core]
                                                          │
                          (low value / unused)  decay-to-engram
                                                          ▼
                                                  [T4 Engram Residue Layer]
                                                          │
                          cue match ("quick glance") ──▶ SAVINGS re-consolidation
                                                          │  (warm-start + few TTT steps)
                                                          ▼
                                                back to [T2] / [T3]
```

---

## 6. The Savings Mechanism (Core Contribution)

### 6.1 Forgetting that preserves a recoverable trace

Let a skill be parameterized by a low-rank adapter $\Delta W^\* = B^\* A^\*$ learned over $WC$ and stored in ES. On decay, MNEMOSYNE discards $\Delta W^\*$ and stores only an engram
$$ \sigma = \big(\, U_k, \; h(\kappa), \; v \,\big), $$
where $U_k$ are the top-$k$ left singular vectors of $\Delta W^\*$ (with $k$ small), $h(\kappa)$ is a sketch of the skill's context key $\kappa$, and $v$ is a value tag. The storage cost of $\sigma$ is a small constant, independent of how expensive the skill was to learn originally.

### 6.2 Re-consolidation from a warm start

When a new context arrives whose pattern-separated key matches $h(\kappa)$ (the "quick glance"), a small **reconstructor network** $R_\theta$ produces a *warm-start* adapter:
$$ \widehat{\Delta W}_0 = R_\theta(\sigma, \kappa_{\text{new}}). $$
A few **test-time-training** gradient steps then refine $\widehat{\Delta W}_0$ on the current task to recover (or surpass) the original skill:
$$ \widehat{\Delta W}_{t+1} = \widehat{\Delta W}_t - \eta \, \nabla_{\Delta W} \, \mathcal{L}\big(\text{task}; \widehat{\Delta W}_t\big). $$
Because $\widehat{\Delta W}_0$ lies near the old optimum, the number of steps to reach criterion is small — this is the engineered analogue of biological savings. The reconstructor $R_\theta$ is itself trained (and continually improved) on the agent's own decay/re-exposure history, so the system *learns how to remember what it forgot*.

### 6.3 The Savings Ratio objective

Define the **Savings Ratio** for a skill:
$$ S \;=\; \frac{C_{\text{relearn}}}{C_{\text{first}}}, $$
where $C_{\text{first}}$ is the compute (e.g., gradient steps or tokens) to learn the skill the first time, and $C_{\text{relearn}}$ is the compute to recover it from its engram. A pure external store has $S \approx 1$ (no savings beyond raw retrieval) or unbounded memory; naïve forgetting (deletion) has $S = 1$ always. The MNEMOSYNE objective is
$$ \min_{\text{policy}} \; \mathbb{E}_{\text{skills}}[S] \quad \text{subject to} \quad \text{MemoryFootprint} \le M_{\max}. $$
This makes explicit the human trade-off: *forget enough to stay within a memory budget, but forget in a way that keeps re-learning nearly free.* The two knobs — the decay policy (what becomes an engram) and the reconstructor (how well an engram warm-starts) — are exactly what the recursive loop optimizes.

### 6.4 Why this is new

Generative replay preserves *old distributions* but offers no cheap-relearning guarantee for *deliberately forgotten* skills. TTT adapts quickly but **discards** the adaptation. Retrieval/RAG recalls verbatim text but does not *re-internalize a skill* cheaply. MNEMOSYNE's contribution is the **savings-preserving decay** primitive — a compressed, reconstructable residue that turns "forgotten" into "cheap to recover," embedded in a self-improving agent. We are not aware of prior work that unifies Ebbinghaus savings, low-rank adapter reconstruction, and recursive memory-policy optimization in this way.

---

## 7. Recursive Self-Improvement of the Memory Policy

Following the DGM/SICA paradigm, MNEMOSYNE can read and edit its own code. We extend the editable surface to the **memory machinery itself**:

- the **decay policy** (the retention-score function and threshold),
- the **engram codec** ($k$, what to keep in $\sigma$, quantization),
- the **reconstructor** $R_\theta$ and the number of re-consolidation steps,
- the **consolidation schedule** (when to "sleep," replay budget).

Each proposed modification is **empirically validated** on a held-out lifelong-learning suite (Section 8): a change is accepted only if it improves the joint objective (lower $\mathbb{E}[S]$ and/or higher retained accuracy at the same footprint). This yields a **second-order** improvement loop: the agent gets better at *managing its own memory*, which makes every future skill cheaper to acquire, forget, and recover — the compounding signature of true RSI, now applied to memory rather than only to task skill.

A subtle benefit: in open-ended self-improvement, agents constantly abandon strategies. Today those strategies are lost or left as dead weight in an archive. Under MNEMOSYNE each abandoned strategy leaves an **engram**, so revisiting it later is a "quick glance," not a rebuild — directly mitigating the **compute blow-up** that limits the depth of open-ended search.

---

## 8. Proposed Evaluation

*All experiments below are proposed; no results are claimed.*

### 8.1 Hypotheses

- **H1 (Savings).** Re-consolidation from an engram reaches criterion accuracy with $S \ll 1$ (target: an order-of-magnitude fewer steps than first learning).
- **H2 (Bounded footprint, retained competence).** At a fixed memory budget $M_{\max}$, MNEMOSYNE retains higher average accuracy across a long task stream than (a) fine-tuning, (b) replay-only, (c) RAG/external-store, and (d) TTT-with-reset baselines.
- **H3 (Recursive gain).** Allowing the agent to edit its own memory policy lowers $\mathbb{E}[S]$ over successive self-improvement rounds, versus a fixed policy.

### 8.2 Benchmarks

- **Lifelong skill streams:** long sequences of class-/domain-incremental tasks (e.g., Split-MNIST/CIFAR for sanity; then text/code skill streams) with **deliberate revisits** of earlier tasks to measure savings.
- **Agentic/coding:** SWE-Bench Verified and Polyglot run in a *recurring-issue* regime, where related issues reappear after long gaps, to test whether engrams accelerate re-solution.
- **A purpose-built "Savings Benchmark":** learn → forced decay → long delay → re-exposure, measuring $S$ directly. (This benchmark would be a contribution in its own right.)

### 8.3 Metrics

Average accuracy / backward transfer (forgetting), **Savings Ratio $S$**, memory footprint over time, compute per retained skill, and the trajectory of $\mathbb{E}[S]$ across self-improvement rounds.

### 8.4 Ablations

Remove the ERL (pure forget-by-deletion); replace the reconstructor with a cold start; freeze the memory policy (no recursion); vary engram rank $k$; vary the decay threshold. Each isolates one claim.

---

## 9. Safety Considerations

Self-improving systems must be sandboxed; every credible system in this lineage runs with isolation and human oversight (STOP measured sandbox-escape attempts; DGM and SICA mandate containerization). MNEMOSYNE adds memory-specific risks that require explicit controls:

- **Engram poisoning / false memories.** Biological reactivation can create false memories; an engram reconstructor could hallucinate a skill that was never validated. **Mitigation:** every re-consolidated skill must re-pass its acceptance test before use; engrams carry provenance tags and verification hashes.
- **Value drift in the decay policy.** A self-edited forgetting rule could discard safety-relevant knowledge. **Mitigation:** designate *protected memories* (safety constraints, alignment data) as non-decayable; require human sign-off for edits to the decay policy.
- **Unbounded recursion.** **Mitigation:** hard caps on self-improvement rounds and compute, append-only audit logs of every policy edit, and rollback.
- **Auditability.** Because skills live as discrete adapters/engrams with provenance, MNEMOSYNE is in principle *more* inspectable than monolithic weight updates — a potential safety advantage worth studying.

---

## 10. Limitations

This is a concept paper: the architecture is unimplemented and the experiments unrun. Open questions include whether a small reconstructor can reliably warm-start across diverse skill types; whether pattern-separated keys scale to millions of engrams without collision; the true storage/accuracy Pareto frontier of engram rank $k$; and whether recursive editing of the memory policy converges or destabilizes. The Savings Ratio target ($S \ll 1$) is a hypothesis, not a result. We present MNEMOSYNE to make these questions precise and testable.

---

## 11. Conclusion

Recursive self-improvement has crossed from theory into working software-engineering agents, but it has so far ignored the one capability that lets biological intelligence learn for a lifetime within a fixed skull: **the ability to forget gracefully and re-learn almost for free.** By copying the Ebbinghaus *savings effect* — compressing forgotten skills into cheap, reconstructable engrams and warm-starting their recovery with a few test-time steps — and by making the memory policy itself an object of recursive improvement, MNEMOSYNE aims to turn the costly explore-and-abandon loop of open-ended self-improvement into a sustainable, human-like cycle of learning, forgetting, and remembering. The central quantity to drive toward zero is the Savings Ratio. If forgetting can be made cheap *and* reversible, lifelong recursive intelligence becomes not just possible but economical.

---

## References

1. E. C. Acikgoz, C. Qian, H. Ji, D. Hakkani-Tür, G. Tur. *Self-Improving LLM Agents at Test-Time.* arXiv:2510.07841, 2025.
2. H. Ebbinghaus. *Über das Gedächtnis (Memory: A Contribution to Experimental Psychology).* 1885.
3. J. P. Jun, V. Marupudi, R. S. Shah, S. Varma. *A Neural Network Model of Complementary Learning Systems: Pattern Separation and Completion for Continual Learning.* arXiv:2507.11393, 2025.
4. D. Kumaran, D. Hassabis, J. L. McClelland. *What Learning Systems Do Intelligent Agents Need? Complementary Learning Systems Theory Updated.* Trends in Cognitive Sciences, 2016.
5. J. L. McClelland, B. L. McNaughton, R. C. O'Reilly. *Why There Are Complementary Learning Systems in the Hippocampus and Neocortex.* Psychological Review, 1995.
6. M. McCloskey, N. J. Cohen. *Catastrophic Interference in Connectionist Networks: The Sequential Learning Problem.* Psychology of Learning and Motivation, 1989.
7. A. Novikov et al. *AlphaEvolve: A Coding Agent for Scientific and Algorithmic Discovery.* arXiv:2506.13131, 2025.
8. M. Robeyns, M. Szummer, L. Aitchison. *A Self-Improving Coding Agent.* arXiv:2504.15228, 2025.
9. J. Schmidhuber. *Gödel Machines: Self-Referential Universal Problem Solvers Making Provably Optimal Self-Improvements.* arXiv:cs/0309048, 2003 (rev. 2006/2007).
10. H. Shin, J. K. Lee, J. Kim, J. Kim. *Continual Learning with Deep Generative Replay.* NeurIPS, 2017.
11. Y. Sun, Y. Choi, et al. *Reimagining LLM Memory: Test-Time Training with an End-to-End Formulation (TTT-E2E).* NVIDIA, 2026.
12. B. T. Tutuncuoglu. *NeuroDream: A Sleep-Inspired Memory Consolidation Framework for Artificial Neural Networks.* SSRN 5377250, 2025.
13. G. M. van de Ven, H. T. Siegelmann, A. S. Tolias. *Brain-Inspired Replay for Continual Learning with Artificial Neural Networks.* Nature Communications, 2020.
14. X. Yin, X. Wang, L. Pan, L. Lin, X. Wan, W. Y. Wang. *Gödel Agent: A Self-Referential Agent Framework for Recursive Self-Improvement.* arXiv:2410.04444, 2024.
15. E. Zelikman, E. Lorch, L. Mackey, A. T. Kalai. *Self-Taught Optimizer (STOP): Recursively Self-Improving Code Generation.* arXiv:2310.02304, 2023.
16. J. Zhang, S. Hu, C. Lu, R. Lange, J. Clune. *Darwin Gödel Machine: Open-Ended Evolution of Self-Improving Agents.* arXiv:2505.22954, 2025.

---

*Correspondence: babal@microsoft.com. This is an independent concept paper; views are the author's own. Prepared with AI research assistance. No empirical results are reported; Section 8 describes proposed experiments.*
