# Repository Content Map

## Files

| File | Purpose |
|------|---------|
| `README.md` | **Canonical paper** (July 1, 2026) — "Beyond Autoregressive Generation: Breadth-First Architecture Compilation for Verified Code Synthesis." Original mathematical formulation, 13-agent Witness Council, breadth-first vs depth-first analysis. |
| `PAPER_ORIGINAL.md` | Backup of the original README.md before restructuring |
| `paper.md` | **Updated draft** (July 2, 2026) — Expanded evaluation section with 12-test results, production survival data, comparison with PlanCompiler/Compiled AI/Remy, component benchmarks, GRPO+Witness details |
| `arxiv.tex` | LaTeX source for arXiv submission |

## Version History

- **v1 (July 1):** Breadth-first mathematical model, 13-agent council, teacher-student pipeline
- **v2 (July 2):** Updated evaluation (12-test, 66.7%), deterministic benchmarks, production survival comparison, concurrent work analysis (PlanCompiler, Compiled AI, Remy)
- **v2.1 (July 4):** Baselines corrected from the preregistered n=50 protocol — base 62% parseable (was 0%), teacher 98%/54% (was ~90%/~85%); three-level metric split; orthogonal-failure finding + deterministic repair lift (+18pp) added. PAPER_ORIGINAL.md frozen as the archival record.
