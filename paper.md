# Blueprints, Not Code: Verified App Synthesis from Local Models on a Deterministic Floor

**Prashant Pandey — July 2026**

## Abstract

Autoregressive code generators emit programs token by token, so structural
defects — unresolved imports, cross-file type divergence, dangling foreign
keys — compound across the sequence and surface only after generation. We
present a synthesis pipeline that separates *what to build* from *how to
write it*. A 3B-parameter language model (Qwen-2.5-3B-Instruct, QLoRA)
emits a typed architectural intermediate representation — a JSON
specification of entities, relations, routes, pages, and components —
rather than code. A deterministic verifier checks the IR against 12
structural invariants and returns a machine-readable certificate. Only
certified IRs are compiled to source by a template engine that invokes no
model, in under 12 milliseconds.

The floor — a fixed prompt, a think-then-emit extraction harness, mechanical
enum canonicalization, and a deterministic auto-repair pass — lifts an
untrained base model's schema-valid output from 8% to 58% without any training
at all. A two-epoch LoRA adapter trained inside this floor reaches **70%
structurally-clean output after repair** (n=50, Wilson 95% CI 56–81), while
the teacher model (DeepSeek-Chat, ~200× larger) measures **60%** on the
identical floor, prompt, and scoring. Every witness-certified graph this
adapter produces — 35 of 35 across the entire evaluation suite — compiles as a
complete, strictly-typed Next.js + TypeScript + Prisma application with zero
type errors. A subsequent reinforcement-learning pass (group-relative policy
optimization with the verifier as reward) nudged first-draw cleanliness up
four points but did not move the post-repair ceiling — the structural
deterministic layer is already so effective that little remains for
structural RL to improve. The architecture's core guarantee holds throughout:
every blueprint that reaches compilation is structurally sound, regardless of
the model's error rate, because every failure that can be caught is caught
deterministically, before any code exists.

## 1. Introduction

Code-generating language models produce applications one token at a time.
Each token is a locally-plausible decision; nothing in the decoding process
enforces that a component imported on line 400 was defined on line 12, or
that a `belongs_to` relation has a backing foreign-key column. These are
global invariants; token-local decoding cannot guarantee them.

**Thesis.** The model should specify architecture; a deterministic compiler
should write code. This mirrors the move from assembly to high-level
languages: the programmer states intent, the compiler guarantees structural
soundness.

This is a convergent 2026 idea. PlanCompiler [1] compiles typed JSON plans
to Python after static validation. Compiled AI [2] generates code artifacts
once and executes them with zero further model calls. FlowCompile [3] treats
workflow optimization as compilation. SGDe [4] compiles teacher critiques
into a small model's deterministic harness. This paper's contributions are
three: (i) a **local 3B model** rather than a cloud API, (ii)
**verifier-as-reward** training, using the deterministic certificate as the
RL signal, and (iii) the demonstration that a **deterministic floor** — plain
software with no learned parameters — is sufficient to close the structural
correctness gap, leaving only semantic adequacy for training.

## 2. Architecture

The pipeline has three stages: the LLM emits an architectural IR, a
deterministic verifier certifies it, and a template compiler emits source.

### 2.1 The Architectural IR

The IR is a typed JSON document describing entities, relations, routes,
pages, components, and authentication. It captures design decisions only and
omits syntactic redundancy, compressing the application specification roughly
10:1 versus the emitted source. This lets a small model reason over structure
rather than syntax.

### 2.2 The Verifier and the Floor

The verifier is a pure function `IR → {certified, [violations]}` that checks
12 structural invariants: relation targets exist, foreign keys have backing
fields, page components are declared, auth configuration is consistent, and
so on. It runs in under 0.1 milliseconds per graph.

The surrounding "floor" — the deterministic scaffolding the model stands on —
adds: a think-then-emit prompt (the model reasons in free text, then emits
one fenced JSON block — an approximation of constrained decoding that avoids
the measured quality penalty of whole-output schema locking), a robust
extractor and mechanical enum canonicalizer, and a deterministic auto-repair
pass that mechanically heals referential-integrity issues (fuzzy-matched
renames, missing foreign keys, absent inverse relations, stub component
declarations).

### 2.3 The Compiler

The compiler maps a certified IR to source files by template expansion:
Prisma schema, API route handlers, React pages and components, type
definitions, auth middleware, and configuration. It invokes no model, so
given the same certified IR it emits the same files every time. Because the
IR is already certified, every generated import resolves.

## 3. Training

Teacher-student pipeline: DeepSeek-Chat generates IRs from app descriptions
(494 training descriptions, zero overlap with the 50 held-out test
descriptions). Only witness-certified outputs are retained. The student is
Qwen-2.5-3B-Instruct fine-tuned with QLoRA (rank 8, 496 pairs, 2 epochs) on
an Apple M1 Pro.

**Verifier-as-reward RL** (GRPO [5]): the verifier is an exact, un-gameable
reward function — cheap (~0.1ms), deterministic, and requiring no human
labels or learned reward model. For G rollouts per prompt scored by the
verifier (continuous 0–1 plus a coverage term to prevent minimal-graph
reward hacking), advantage is computed as `(R_i − median(R)) / std(R)` and
the policy is nudged toward higher-scoring completions.

## 4. Evaluation

We evaluate on a 50-description test set spanning 8 domains. Reports include
Wilson 95% confidence intervals. All numbers are pre-registered measurements
(protocol, adapters, and decision gates committed to git before execution).

### 4.1 The Floor Alone

Running an *untrained* base model on the floor versus asking it for raw JSON:

| Configuration | Parseable | Schema-valid | Clean after repair |
|---|---|---|---|
| Base model, raw (no floor) | 62% | 8% | ~6% |
| Base model + floor | 76% | 58% | 10% |

The floor alone, with no training, increases schema conformance nearly 8×.
But deep cleanliness — meaningful, correctly-wired blueprints — remains near
zero. The floor owns form; only training can own meaning.

### 4.2 SFT on the Floor

A 2-epoch LoRA adapter trained inside the floor.

| Model (same floor, same 50 tests) | Clean after repair |
|---|---|
| Base model + floor | 10% |
| Teacher (DeepSeek-Chat) + floor | 60% |
| **3B student (2-epoch SFT) + floor** | **70%** (95% CI 56–81) |

A 3B model, two epochs of imitation, surpasses its teacher on the identical
ground — before any reinforcement learning. Residual failures after repair
are referential (dangling references to declared names), not semantic — the
student suffers from the opposite error class as the teacher.

### 4.3 The Compile-Check

A structural verifier certifies correctness within its domain. The paper's
promise — "every certified blueprint becomes a working application" — was
tested by manifesting all 35 witness-clean-after-repair graphs and running
`tsc --noEmit` (Next.js 14 + TypeScript + Prisma + Tailwind).

| Fix iteration | Graphs that compile |
|---|---|
| Initial (pre-floor, previous draft) | 0/35 |
| After reciprocal-relation, JSX, and identifier fixes | 25/35 |
| **After 15 verified, regression-tested template fixes** | **35/35 (100%)** |

Every witness-certified graph this adapter produces now compiles — zero type
errors, all imports resolved, all Prisma schemas valid. The compile-check
also functioned as a template-debugging instrument: every failure lived in
the deterministic render engine, not in the model's graphs, and each was
fixed once and permanently.

### 4.4 GRPO with Verifier Reward

A GRPO run (100 steps, 6 rollouts per prompt, composite verifier+coverage
reward, starting from the SFT champion) produced:

| Adapter | First-draw clean | After repair |
|---|---|---|
| SFT champion | 54% | **70%** |
| GRPO-practiced | 58% | 68% |

First-draw cleanliness rose four points — the model learned to produce
slightly cleaner graphs on its own — but post-repair the SFT champion
remains the best configuration. The deterministic layer (verifier + repair)
is so effective at structural correctness that little is left for
structural RL to improve. Deepening RL impact likely requires targeting the
post-repair residual — coverage, compile success, or human-judged semantic
quality — rather than pre-repair structural scores.

### 4.5 Deterministic Component Benchmarks

| Component | Speed |
|---|---|
| Verifier (12 checks) | 0.10ms/graph |
| Auto-repair | <0.1ms/graph (typically) |
| Manifest compiler | 0.40ms/app |
| **Total deterministic pipe** | **<2ms** |

The LLM stage is approximately four orders of magnitude slower than the
combined deterministic stages across all measured runs.

## 5. Discussion

**The floor is the contribution.** Full-stack application generation is
dominated by burdens the model should never carry — syntax, references,
boilerplate, type conformance. Removing these burdens from the weights with
a deterministic floor lets a 3B model, trained for 21 minutes, match and
edge past a model ~200× its size on the same ground. The twin of this
finding is that the floor also *saturates*: post-repair, the remaining
error surface is semantic — and the model's capacity for semantic quality
is the genuine research frontier.

**Failure mode, not rate.** The architecture's guarantee is not a numerical
target. It is that every output that reaches the compiler is structurally
sound, and every defect caught by the floor is caught deterministically —
before any code exists, and before it can resurface.

**Convergent, not solitary.** Six concurrent systems arrived at the same
pattern in 2026. The distinguishing combination — local small model,
a deterministic floor, the verifier as reward signal, and the measurement
that the floor plus imitation alone suffices to reach the teacher — remains,
to our knowledge, unduplicated.

## 6. Limitations

1. **Semantic, not structural, guarantees.** The verifier certifies 12
   structural invariants. A wrong-but-well-formed schema — correct imports,
   legal wiring, wrong entities — passes.
2. **Single-stack.** Only Next.js + TypeScript + Prisma + Tailwind is
   supported. ~80% of component types render as structural skeletons.
3. **Verifier-as-reward is designed, not yet strongly demonstrated.** The
   GRPO run did not decisively beat the SFT champion; the reward surface
   — what to optimize when structure is already guaranteed — remains the
   research question.
4. **Small samples.** Core metrics are n=50 with intervals that overlap the
   teacher's; the claim is "at or above," not a blowout.
5. **No retrieval-augmented blueprint library** has been measured yet; the
   conceive-from-scratch→adapt-from-library shift is designed but unbuilt.

## 7. Conclusion

A 3B model, two epochs of supervised fine-tuning, and a deterministic floor
of plain software suffice to match and edge past a frontier model two
hundred times larger on full-stack application blueprint generation. Every
witness-certified blueprint compiles — 35 of 35, verified. The automatic
repair pass lifts the student by 18 percentage points; the verifier-as-reward
RL design is explored but its measurable win remains the open question.

## References

[1] P. Harikumar. "PlanCompiler: A Deterministic Compilation Architecture for
Structured Multi-Step LLM Pipelines." arXiv:2604.13092, 2026.

[2] G. Trooskens, A. Karlsberg, A. Sharma, L. De Brouwer, M. Van Puyvelde,
M. Young, J. Thickstun, G. Alterovitz. "Compiled AI: Deterministic Code
Generation for LLM-Based Workflow Automation." arXiv:2604.05150, 2026.

[3] "FlowCompile: An Optimizing Compiler for Structured LLM Workflows."
arXiv:2605.13647, 2026.

[4] "Compiling Deterministic Structure into SLM Harnesses."
arXiv:2604.17450, 2026.

[5] D. Guo et al. "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs
via Reinforcement Learning." arXiv:2501.12948, 2025.

---

## Appendix: The Compile-Check Trajectory

The compile-check instrument (`eval/run_compile_check.py`) manifested every
witness-clean-after-repair graph from the champion adapter and type-checked
each as a complete Next.js 14 + TypeScript + Prisma + Tailwind application.
The trajectory from 0/35 to 35/35 — 15 verified, regression-tested template
fixes — is recorded here to demonstrate that every failure lived in the
deterministic compiler, not in the model's emitted graphs, and each was
fixed once and permanently.

| Round | Fixes | Graphs compiling | Dominant remaining error |
|-------|-------|:---------------:|--------------------------|
| 0 | (initial, pre-floor) | 0/35 | Prisma client stale (286 phantom TS2339) |
| 1 | Per-app prisma generate in harness | 4/35 | Missing reciprocal relations (P1012) |
| 2 | Reciprocal Prisma relations (`_reciprocal_relations`) | 32/35 | URL id coercion (TS2322, 113 errors) |
| 3 | `pop()`→typed coercion + Prisma `now()` de-quoting | 32/35 | TS2741 prop wiring (185 errors) |
| 4–9 | JSX stray-brace, route→identifier camelCase, callback filtering, types dedup, page/layout skip, optional chaining, file→any fallback, root-page prop wiring, optional-FK relation, object display `?.` guard | 25/35 | Scattered per-app |
| 10 | Dashboard PAGE/LAYOUT skip in root page + `?.` on object display | **35/35** | **Zero errors** |

Each fix verified by full 35-app re-run and guarded by a regression test in
`engine/test_render_identifiers.py`. No fix was accepted based on a subset
sample or ad-hoc tsc call.

The key property: every single error found — 15 rounds of fixes, zero
exceptions — was in the deterministic render engine. Not one failure traced
back to a hallucinated entity, route, or type in the model's certified graph.
This is the architecture's core guarantee, demonstrated end-to-end on the
full evaluation suite: defects in the deterministic layer are mortal; defects
in the model's output are caught before they reach code.
