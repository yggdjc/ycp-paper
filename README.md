# Assemble, Don't Accumulate: Deterministic Context Management for Long-Horizon LLM Agents

**Author:** Ting-Chieh Chu, YGG Team

## Abstract

Long-horizon LLM agents treat context as mutable state -- content accumulates, instructions lose attention share monotonically, and behavioral guarantees erode. All existing approaches (compaction, paging, retrieval, compression) share a common computational model: context is state that grows and must be reduced.

We propose a different computational model: context is not stored; it is computed. Working Memory is a deterministic function of structured session memory -- WM = f(SessionMemory, Config). Each LLM call begins with empty working memory; a deterministic assembler renders context from structured session memory using LLM-declared metadata.

We prove accumulate attention share decays as O(1/n) while assembly maintains a constant floor Omega(|C|/B), and connect this to information capacity collapse. We implement this in ycp (ygg-context-proxy), an open-source context proxy that interposes between agent and LLM.

On synthetic 100-turn scenarios and SWE-bench, across GPT-5 and qwen3-4b, assembly maintains instruction adherence where accumulate-compact and RAG-select degrade; achieves 0% unauthorized token-level cross-task leakage; and matches RAG on downstream metrics at lower per-call cost.

## Core Contributions

1. **Attention dilution analysis.** Formal proof that accumulate attention decays O(1/n) (Theorem 1) while assembly maintains constant floor (Theorem 2), with architecture-aware bounds under RoPE and ALiBi.
2. **The assembly principle and its mechanisms.** Four deterministic mechanisms: constitution partitioning, budget-driven episode selection, decision pinning via LLM-declared metadata (mlock pattern), and structural task isolation.
3. **Empirical validation.** 5 baselines (AN/AC/AX/RS/OURS), 2 models (GPT-5, qwen3-4b), synthetic scenarios + SWE-bench-lite with full statistical rigor.

## Repository Structure

```
paper-spec.md      # Full design spec
README.md          # This file
LICENSE            # CC BY 4.0
```

## License

This work is licensed under [Creative Commons Attribution 4.0 International](LICENSE).
