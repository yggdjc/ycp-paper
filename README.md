# Assemble, Don't Accumulate: Deterministic Context Management for Long-Horizon LLM Agents

English | [中文](README_zh.md)

**Author:** Ting-Chieh Chu, YGG Team

## Abstract

Long-horizon LLM agents treat context as mutable state -- content accumulates, instructions lose attention share monotonically, and behavioral guarantees erode. All existing approaches (compaction, paging, retrieval, compression) share a common computational model: context is state that grows and must be reduced.

We propose a different computational model: context is not stored; it is computed. Working Memory is a deterministic function of structured session memory -- WM = f(SessionMemory, Config). Each LLM call begins with empty working memory; a deterministic assembler renders context from structured session memory using LLM-declared metadata. This is not an optimization of accumulate; it is a replacement.

We prove accumulate attention share decays as O(1/n) while assembly maintains a constant floor Omega(|C|/B), and connect this to information capacity collapse. Any system that maintains context as mutable state belongs to the accumulate class and inherits this decay. Any system that constructs context as a bounded function belongs to the assembly class and admits constant attention guarantees. No hybrid escapes this classification.

We implement this in ycp (ygg-context-proxy), an open-source context proxy that interposes between agent and LLM.

## Core Contributions

1. **Attention dilution analysis.** Formal proof that accumulate attention decays O(1/n) (Theorem 1) while assembly maintains constant floor (Theorem 2), with architecture-aware bounds under RoPE and ALiBi. This is a classification result: it partitions all context management strategies into two formally distinct categories.

2. **The assembly principle and its mechanisms.** Four deterministic mechanisms: constitution partitioning, budget-driven episode selection, decision pinning via LLM-declared metadata (mlock pattern), and structural task isolation.

3. **Empirical validation.** 5 baselines (AN/AC/AX/RS/OURS), multiple models spanning 3.6B dense to 671B MoE, synthetic 100-turn scenarios + SWE-bench-lite with full statistical rigor.

## Status

Paper in preparation. Early experimental results on S1 (instruction adherence, 100 turns, 32K budget):

| Model | AN (accumulate) | OURS (assembly) | Delta | First Fail (AN → OURS) |
|-------|-----------------|-----------------|-------|------------------------|
| qwen3-4b (3.6B dense, n=3) | 73.2 ± 6.6% | 99.1 ± 0.8% | +25.9% | turn 14-22 → turn 60-never |
| DeepSeek V3.2 (671B MoE) | 70.8% | 92.8% | +22.0% | turn 14 → turn 41 |

qwen3-4b multi-run details (3 runs):

|          | AN r0  | AN r1  | AN r2  | OURS r0 | OURS r1  | OURS r2 |
|----------|--------|--------|--------|---------|----------|---------|
| Overall  | 70.4%  | 80.8%  | 68.4%  | 98.4%   | 100.0%   | 98.8%   |
| R4       | 64%    | 100%   | 29%    | 100%    | 100%     | 100%    |
| R5       | 64%    | 100%   | 29%    | 100%    | 100%     | 100%    |
| 1st fail | t14    | t22    | t22    | t60     | never    | t61     |

AN shows high variance (68-81%) with R4/R5 adherence collapsing to 29% in the worst run. OURS is stable (98-100%) with R4/R5 at 100% across all runs. Assembly's first failure occurs 38-87 turns later than accumulate's, or not at all.

**In progress:**
- gpt-5.4-mini AN + OURS (closed-source model validation)
- DeepSeek V3.2 multi-run variance confirmation
- AC (LLM compaction), AX (LLMLingua compression), RS (RAG retrieval) baselines
- S2 (canary survival), S3 (decision consistency), S4 (task isolation), S5 (deep backtracking)
- Attention share analysis (direct extraction on qwen3-4b, logprobs on gpt-5.4-mini)

## Repository Structure

```
README.md          # English version
README_zh.md       # 中文版本
paper-spec.md      # Full paper design specification
LICENSE            # CC BY 4.0
```

Evaluation framework and ycp source code will be released upon paper completion.

## License

This work is licensed under [Creative Commons Attribution 4.0 International](LICENSE).
