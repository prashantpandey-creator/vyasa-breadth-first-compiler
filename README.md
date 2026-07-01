# Puranic Architecture — Research Paper

**Vyasa: Structured Architectural Graphs with Deterministic Witness for Prompt-to-Code Generation**

Prashant Pandey — July 2026

## Files

| File | Description |
|------|-------------|
| `paper.md` | Full research paper (Markdown) |
| `arxiv.tex` | LaTeX source for arXiv submission |

## Abstract

We present Vyasa, a three-stage compiler architecture for prompt-to-code generation: Conception (LLM → Architectural Graph) → Witness (12 deterministic structural checks) → Manifest (compiler → working code). The Architectural Graph compresses application specification ~13:1. A 3B adapter achieves 66.7% valid JSON. Deterministic stages execute in <2ms. We introduce GRPO+Witness — a training flywheel where the Witness itself is the reward function.

## Related Repos

- **Engine:** [vyasa-manifestation-engine](https://github.com/prashantpandey-creator/vyasa-manifestation-engine)
- **Models:** [huggingface.co/PRASSANT](https://huggingface.co/PRASSANT)

## References

- PlanCompiler (arXiv 2604.13092, 2026) — typed node registry + static graph validation
- Compiled AI (arXiv 2604.05150, 2026) — compile-time generation, zero-token execution
- Appycodes Audit (2026) — 31 AI codebases, 38-50% production survival

## Status

- [ ] arXiv submission
- [ ] Full 50-test eval
- [ ] GRPO+Witness execution
- [x] Paper draft complete
- [x] GitHub published

---
**Jai Gurudev.**
