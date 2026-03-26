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
| qwen3-4b (3.6B dense, n=5) | 78.1 ± 8.4% | 99.1 ± 0.6% | +21.0% | turn 22-30 → turn 12-never |
| DeepSeek V3.2 (671B MoE, n=3) | 64.9 ± 5.8% | 92.5 ± 7.6% | +27.6% | turn 14-28 → turn 24-never |

qwen3-4b per-run details (5 runs):

|          | AN r0  | AN r1  | AN r2  | AN r3  | AN r4  | OURS r0 | OURS r1  | OURS r2 | OURS r3 | OURS r4 |
|----------|--------|--------|--------|--------|--------|---------|----------|---------|---------|---------|
| Overall  | 70.4%  | 80.8%  | 68.4%  | 88.0%  | 82.8%  | 98.4%   | 100.0%   | 98.8%   | 99.2%   | 99.2%   |
| R4       | 26%    | 100%   | 29%    | 100%   | 57%    | 100%    | 100%     | 100%    | 99%     | 100%    |
| 1st fail | t26    | t22    | t22    | t30    | t26    | t60     | never    | t61     | t12     | t28     |

DeepSeek V3.2 per-run details (3 runs):

|          | AN r0  | AN r1  | AN r2  | OURS r0 | OURS r1  | OURS r2 |
|----------|--------|--------|--------|---------|----------|---------|
| Overall  | 70.8%  | 59.2%  | 64.8%  | 92.8%   | 100.0%   | 84.8%   |
| R4       | 64%    | 29%    | 26%    | 91%     | 100%     | 81%     |
| 1st fail | t14    | t28    | t24    | t41     | never    | t24     |

AN shows high variance across both models (qwen3-4b: 68-88%, DeepSeek: 59-71%) with R4 adherence collapsing to 26-29% in worst runs. OURS is consistently higher (qwen3-4b: 98-100%, DeepSeek: 85-100%) with R4 ≥ 81% across all runs.

**In progress:**
- gpt-5.4-mini AN + OURS (closed-source model validation)
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
