# Paper Design Spec: "Assemble, Don't Accumulate"

**Title:** Assemble, Don't Accumulate: Deterministic Context Management for Long-Horizon LLM Agents

**Author:** Ting-Chieh Chu, YGG Team

**Framing:** Principle-first (not system-first, not theory-first)

**Core claim:** LLM context should be treated as a deterministic function of structured memory (WM = f(SessionMemory, Config)), not as mutable state that grows and must be reduced. This computational model — assemble-from-scratch — provides provable attention guarantees, structural isolation, and matches retrieval-based selection at orders-of-magnitude lower per-call cost.

---

## Design Decisions Log

### Framing: Paradigm-first (computational model)
- The paper sells a **new computational model for LLM context**: WM = f(SessionMemory, Config) — context as rendered view, not mutable state
- This is a paradigm replacement, not an optimization: accumulate treats context as state; assembly treats it as a deterministic computation
- Theory supports the paradigm; system (ycp) is a reference implementation; experiments validate predictions
- §3 title is "Reference Implementation" — explicitly secondary
- Four consequences of the paradigm: reproducibility, verifiability, boundedness, provability

### Baselines: 5, grouped into two families
- Accumulate family: AN (naive), AC (LLM compaction), AX (LLMLingua compression)
- Selection family: RS (RAG episode selection), OURS (metadata assembly)
- Grouping reduces cognitive load and sharpens the comparison

### Negative scenario: S5 (deep backtracking)
- Positioned as boundary testing (§5.5), not as a main experiment
- Assembly's known weakness: episodes outside sliding window without decision markers are lost
- RS expected to outperform; gap reported honestly

### Corollary 2 (MemGPT recursive failure): deleted
- Was informal conjecture with no experimental support
- Would invite "prove it" attacks with no way to respond
- MemGPT discussed qualitatively in §2.3 with explanation of why direct comparison is infeasible

### Attention → Compliance bridge
- Placed in §1 as framing assumption, not in §4 as afterthought
- "Attention share is a necessary condition; zero attention guarantees violation"
- Empirical validation (§5) confirms attention translates to behavioral compliance

### Sentiment filtering: removed
- Removed entirely after experiments showed keyword-based sentiment causes more harm than benefit
- Simplifies paper: §3.3 no longer needs ablation justification

### Metadata source: LLM-declared (mlock pattern)
- All episode metadata is declared by the LLM via tool calls, not inferred by the system via keyword/regex heuristics
- OS analogy: applications call `mlock()` to tell OS "don't swap this out" — OS doesn't scan memory to guess importance
- `mark_decision` tool: LLM declares "this response contains a decision"; system sets `has_decision = true`
- `reference_task` tool: exists in codebase but not evaluated in paper (moved to §6.3 Future Work to keep paper focused)
- Assembly path unchanged: reads boolean fields, no semantic analysis
- Eliminates keyword detection entirely — no precision/recall concern
- Failure mode shifts from continuous degradation to discrete annotation errors

### Novelty positioning
- Primary novelty: **first formal proof that accumulate attention share decays O(1/n) and assembly maintains constant floor** — this is a general result about softmax + growing vs bounded contexts, independent of any specific system
- Secondary novelty: structural isolation guarantee (Properties 1-2) — first formal argument that accumulate systems cannot guarantee isolation as conversations grow
- The RS comparison is NOT the novelty claim ("our system is cheaper than RAG"). It validates that metadata selection is a viable alternative, providing practical justification for the principle

---

## Paper Structure

### Abstract (~250 words)

1. Problem (1 sentence): Long-horizon LLM agents treat context as mutable state — content accumulates, instructions lose attention share monotonically, and behavioral guarantees erode
2. Insight (1 sentence): All existing approaches (compaction, paging, retrieval, compression) share a common computational model: context is state that grows and must be reduced
3. Paradigm shift (1-2 sentences): We propose a different computational model: context is not stored; it is computed. Working Memory is a deterministic function of structured session memory — WM = f(SessionMemory, Config). Each LLM call begins with empty working memory; a deterministic assembler renders context from structured session memory using LLM-declared metadata
4. Theory (1 sentence): We prove accumulate attention share decays as O(1/n) while assembly maintains a constant floor Ω(|C|/B), and connect this to information capacity collapse
5. System (1 sentence): We implement this in ycp (ygg-context-proxy), an open-source context proxy that interposes between agent and LLM
6. Results (2-3 sentences): On synthetic 100-turn scenarios and SWE-bench, across GPT-5 and qwen3-4b, assembly maintains [TBD]% instruction adherence at turn 100 vs [TBD]% for accumulate-compact and [TBD]% for RAG-select; achieves 0% unauthorized token-level cross-task leakage; matches RAG on downstream metrics at [TBD]× lower per-call cost
7. Availability (1 sentence): Code and evaluation framework released as open-source

---

### §1 Introduction (~1.5 pages)

**Para 1 — Problem scale:**
Long-horizon LLM agents span dozens to hundreds of interactions [Yang et al., 2024; Wang et al., 2024]. As context grows, Transformer softmax attention [Vaswani et al., 2017] distributes fixed probability mass over all tokens; critical instructions lose attention share monotonically [Liu et al., 2024; Kamradt, 2023; Hsieh et al., 2024].

**Para 2 — The accumulate model is structurally unable to provide guarantees:**
The dominant response is accumulate-then-compact: summarization [Anthropic, 2025; Cursor Inc., 2025], LLM-managed paging [Packer et al., 2024], retrieval-augmented selection [Lewis et al., 2020; Park et al., 2023], learned token compression [Jiang et al., 2023b; Chevalier et al., 2023]. These are not merely suboptimal implementations — they share a computational model (context as mutable state) that is structurally unable to provide attention guarantees. Content enters, reduction is applied post-hoc, and the result carries forward; but as long as context length can grow, Theorem 1 applies and attention share decays regardless of the quality of the reduction. The problem is not insufficient compaction; the problem is the accumulate model itself. Even prompt template systems [Chase, 2022] accumulate conversation history without budget-aware selection or structural isolation.

**Para 3 — A different computational model:**
Context is not stored; it is computed. We propose treating Working Memory not as a container to be filled and pruned, but as a deterministic function: WM = f(SessionMemory, Config). Each LLM call begins with empty working memory; a deterministic assembler renders context from structured session memory using LLM-declared metadata. Working Memory is a rendered view — reproducible, verifiable, bounded, and provable. No LLM calls, no embeddings, no content analysis on the assembly path. Constitution occupies fixed partition at context start. This work does not introduce a better method — it introduces a criterion that separates all context management systems into two provably distinct classes.

Key distinction from prompt templates: (1) budget-aware content selection over variable-length history, (2) LLM-declared metadata filtering, (3) structural cross-task isolation.

This distinction is not about performance or engineering trade-offs. It defines two mutually exclusive computational models. Any system that maintains context as mutable state belongs to the accumulate class and inherits O(1/n) attention decay (Theorem 1). Any system that constructs context as a bounded function belongs to the assembly class and admits constant attention guarantees (Theorem 2). No hybrid escapes this classification: either context length can grow unboundedly (accumulate), or it is recomputed under a fixed bound (assembly). Context management is not a memory problem — it is a rendering problem.

**Para 4 — Attention → compliance framing assumption:**
We focus on attention share as a necessary condition for instruction adherence: a model that allocates negligible attention to its instructions cannot reliably follow them [Liu et al., 2024]. Attention is not sufficient for compliance, but zero attention guarantees violation. We do not claim a causal chain from attention to behavior; we claim predictive power — attention share reliably predicts compliance outcomes across models and baselines (validated in §5.4). Even if the attention-compliance correlation weakens for a given model, Theorem 1 still guarantees the degradation of attention allocation, making any attention-dependent compliance mechanism unreliable under accumulate. Our theoretical results establish attention guarantees; empirical results confirm that these guarantees predict behavioral compliance.

**Contributions:**

1. **Attention dilution analysis.** Accumulate-then-compact: attention share decays O(|C|/L_n) = O(1/n) (Theorem 1). Assembly: attention share ≥ Ω(|C|/B), constant (Theorem 2). Architecture-aware bounds under RoPE [Su et al., 2024] and ALiBi [Press et al., 2022]. Validated against direct attention measurements.

2. **The assembly principle and its mechanisms.** Four deterministic mechanisms: (a) constitution partitioning (always at position [0]), (b) budget-driven episode selection with sliding window O(n), (c) decision pinning (critical moments survive eviction), (d) structural task isolation (Property 1: zero leakage by construction; Property 2: accumulate cannot guarantee isolation). Implemented in ycp as reference.

3. **Empirical validation against competitive baselines.** 5 baselines (AN/AC/AX/RS/OURS), 2 models (GPT-5, qwen3-4b), synthetic 100-turn scenarios + SWE-bench-lite:
   - Instruction adherence turn 100: [TBD]% (OURS) vs [TBD]% (AC) vs [TBD]% (RS)
   - Canary survival turn 100: [TBD]% vs [TBD]% vs [TBD]%
   - Decision consistency turn 100: [TBD]% vs [TBD]% vs [TBD]%
   - Cross-task unauthorized token-level leakage: 0% (OURS) vs [TBD]% (accumulate family)
   - Matches RS on metrics at [TBD]× lower FLOPs
   - SWE-bench: comparable resolve rate, [TBD]% fewer total tokens
   - All p-values Holm-Bonferroni corrected, Cohen's d reported

**Paper organization paragraph.**

---

### §2 Background & Related Work (~1.5 pages, compressed)

**2.1 Attention Mechanics and Long-Context Degradation**
- Softmax zero-sum [Vaswani et al., 2017]
- Needle-in-Haystack [Kamradt, 2023], Lost in the Middle [Liu et al., 2024], RULER [Hsieh et al., 2024]
- Attention sinks [Xiao et al., 2024]
- Infini-attention [Munkhdalai et al., 2024] — still accumulate
- Sparse attention: Longformer [Beltagy et al., 2020], BigBird [Zaheer et al., 2020], Mistral SWA [Jiang et al., 2023a] — reduce compute cost, don't address semantic dilution
- Prompt compression: LLMLingua [Jiang et al., 2023b], AutoCompressors [Chevalier et al., 2023] — optimize density, not selection. Compressed content still accumulates. Orthogonal to assembly (could combine)

**2.2 Memory Architectures for LLM Agents**
- Generative Agents [Park et al., 2023]: recency + relevance + importance retrieval
- CoALA [Sumers et al., 2024]: cognitive architecture taxonomy (we adopt and extend)
- Reflexion [Shinn et al., 2023]: episodic verbal memory, but reflections accumulate
- RAG [Lewis et al., 2020]: most direct comparator. Episode-level RAG = embed query → retrieve top-k. Selection at read time O(n·d) vs our write-time annotation O(n). Included as baseline (RS)

**2.3 LLM-Managed Virtual Memory and Related Systems**
- MemGPT [Packer et al., 2024]: OS analogy, LLM-managed paging. Recursive dependency: LLM manages own attention using attention being diluted. Not included as baseline — requires function calling, architectural incomparability makes controlled comparison infeasible (different interaction format, not just different context strategy)
- Google ADK [Google, 2025]: as documented in [Google, 2025], separates state from rendering via Session/Application/Memory layers. Closer to assembly but still accumulates conversation history within session
- Other work shares emphasis on deterministic processing [Meta FAIR, 2024] but addresses different problems (generation planning)

**2.4 Prompt Templates and Industrial Practice**
- LangChain [Chase, 2022], LlamaIndex: per-call prompt construction via variable substitution. Shares surface similarity with assembly. Lacks: (1) budget-aware selection, (2) metadata-predicate filtering, (3) structural isolation
- Compaction in practice: Claude Code [Anthropic, 2025], Cursor [Cursor Inc., 2025], OpenHands [Wang et al., 2024], SWE-agent [Yang et al., 2024]
- Prevalence of compaction confirms unbounded accumulation is recognized as problematic; root cause (monotonic attention dilution) not formally analyzed prior to this work

**2.5 Positioning Summary (Table 1)**

| Approach | Decision Maker | Memory | Isolation | Cost |
|----------|---------------|--------|-----------|------|
| Standard accumulate | None | Flat | None | O(1) |
| Prompt templates | Substitution | Slots | None | O(1) |
| RAG [Lewis+] | Embedding sim | Index | None | O(n·d) |
| Generative Agents [Park+] | Recency+relevance+importance | Stream | None | O(n·d) |
| MemGPT [Packer+] | LLM calls | Virtual 2-tier | None | O(L) LLM |
| CoALA [Sumers+] | Framework-dep | 4-module | Framework-dep | Framework-dep |
| Google ADK | Rule-based | 3-tier state | App-level | O(L) |
| LLMLingua [Jiang+] | Token scoring | Flat | None | O(L) small-model |
| AutoCompressors [Chevalier+] | Learned | Soft cache | None | O(L) forward |
| **ycp (Ours)** | **Deterministic metadata** | **3-layer (W/S/P)** | **Structural** | **O(n) fields** |

---

### §3 Reference Implementation (~1.5 pages)

*The assembly principle (§1) and its formal analysis (§4) are independent of any specific implementation. This section describes ycp, one possible realization, to ground the evaluation. The theoretical claims hold for any system satisfying the assembly invariant: WM = f(SessionMemory, Config) with bounded output length.*

**3.1 Three-Layer Memory Model**

| Layer | OS Analogue | Capacity | Lifetime | Access |
|-------|-------------|----------|----------|--------|
| Working Memory | Main memory | Context window budget | Single LLM call | Write-once, read-once, discard |
| Session Memory | Extended memory | Process heap (~100s MB) | Agent session | Read at assembly, write after response |
| Persistent Store | Disk | Filesystem | Cross-session | Read at boot only, async append |

Boot: Persistent Store → Session Memory (full load). After boot, disk never read.

Per-call: Session Memory →(select+serialize)→ Working Memory → LLM → Guardrail → Session Memory →(async)→ Persistent Store.

**Working Memory is a render target, not a container.** Formally: WM = f(SessionMemory, Config) — a stateless projection with no persistence, no mutation, no accumulation. The same Session Memory state and configuration always produce the same Working Memory. This is the fundamental computational model that distinguishes assembly from accumulate: accumulate systems treat context as mutable state (content enters and may be reduced, but the container persists); assembly treats context as a derived view (computed from scratch on every call, discarded after use). This distinction has four direct consequences: (1) reproducibility (same inputs → same context), (2) verifiability (context can be inspected without history), (3) boundedness (budget is a hard constraint, not a target), (4) provability (attention guarantees follow from fixed context length).

**3.2 Deterministic Context Assembly**

Core properties:
- Deterministic in the sense of reproducibility: identical Session Memory state (including LLM-declared annotations) → identical output. The assembly path is a pure function of its inputs. Metadata fields (`has_decision`, `references`) are declared by the LLM via tool calls at write time and stored as structured data. The assembly path reads these fields without interpreting content. The system is deterministic-as-reproducible: same session state → same assembled context.
- No LLM calls, no embeddings, no randomness on the assembly path
- Budget B is a hard constraint, never exceeded

**Algorithm 1 (abbreviated; full version in Appendix C):**

```
ASSEMBLE(focus_task, all_tasks, config, client_msgs):
  1. messages ← [SystemMessage(constitution)]        // Always first
  2. remaining ← B − R − |constitution| − |client_msgs|

  // Managed zone (budget-checked)
  3. Append focus_task.summary, decisions, signals
  4. For each accessible scoped task (sorted by id):
       if room: append task.summary; else: stop

  // Episode zone
  5. candidates ← deduplicate(decision-pinned outside window + recent W episodes)
  6. Fill oldest-first until budget exhausted
     (all candidates included unconditionally; decision-pinned may be
      skipped if oversized; never force-overflow)

  7. Append client_msgs (pass-through)
  return messages
```

Four mechanisms, all deterministic: (1) constitution partitioning, (2) budget-driven episode selection with sliding window, (3) decision pinning (via LLM-declared `mark_decision`), (4) task sandbox access checks. No LLM calls, no embeddings, no keyword heuristics, no semantic retrieval on the assembly path. Metadata is declared by the LLM at write time via tool calls (§3.3); assembly reads structured fields only.

Key: line 5 makes per-episode decisions using LLM-declared metadata field (has_decision) plus O(1) token estimation. O(n) field comparisons, not O(n·d) embeddings.

**Components excluded from paper experiments (disabled).** The ycp codebase includes additional features not evaluated in this paper: semantic supplement zone (embedding-based retrieval, capped at 10%), compaction (cached episode summaries), BiasRegistry (regex-based output checks), and the Learning System (knowledge lifecycle management). All are **disabled** in experiments to isolate the core assembly principle.

Assembly cost: single-digit ms for 500 episodes (measured in §5). Prefix caching compatible — constitution + managed context stable across turns → 90-95% KV cache reuse [Zheng et al., 2024].

**Metadata field table:**

| Field | Type | Source | Used for |
|-------|------|--------|----------|
| has_decision | bool | LLM `mark_decision` tool call | Pin decision-bearing episodes |
| decision_summary | String | LLM `mark_decision` tool call | Managed context injection |

Token estimation uses content length / 4 (O(1) per episode, no tokenizer call). This is a runtime computation but constant-time, consistent with the O(n) assembly claim. Estimation error is bounded: for English text, length/4 overestimates by ~10%; for code, underestimates by ~15%. Budget overflow is prevented by the hard stop in the episode loop (Algorithm 1 step 7) — estimation error affects which episodes are included, not whether the budget is exceeded.

**Implementation note (CompactionResult).** The ycp codebase includes an optional compaction mechanism (wave 3 feature) where episodes may carry a cached compacted summary. In all paper experiments, compaction is **disabled** — the assembler uses raw episode content only. This ensures the evaluation isolates the assembly principle without hybrid accumulate-compact behavior. The existence of the compaction path in code is a pragmatic concession for future hybrid modes, not a contradiction of the paper's thesis.

**3.3 LLM-Declared Metadata (mlock Pattern)**

Assembly's episode selection uses metadata **declared by the LLM**, not inferred by the system.

**Design principle.** An operating system does not scan memory contents to guess which pages are important — applications call `mlock()` to declare "keep this resident." Similarly, ycp does not analyze episode text to infer metadata. The LLM declares metadata via tool calls at response time; ycp stores the structured result and uses it during assembly.

**Tool interface.** One tool is provided to the LLM alongside the constitution:

```json
{
  "name": "mark_decision",
  "description": "Mark the current response as containing a decision that should be preserved across the entire session.",
  "parameters": {
    "summary": "One sentence summary of the decision"
  }
}
```

When the LLM calls `mark_decision`, ycp sets `has_decision = true` on the episode and appends the summary to a per-task bounded FIFO vector (≤ 20 entries).

**Note:** The ycp codebase also provides a `reference_task` tool for cross-task reference declaration. This tool is not evaluated in the current paper (no scenario exercises cross-task referencing in the evaluated direction) and is discussed in §6.3 as future work. The mlock pattern generalizes to any structured metadata the LLM can declare.

**Two-tier decision persistence:** (1) extracted decision summary always present in managed context (unconditional — injected as part of focus_task.decisions in Algorithm 1 step 3); (2) full original episode retained in episode zone when budget permits (conditional — may be skipped if oversized per Algorithm 1 step 6). Decision information is never lost; full episode context may be.

**No system-side content analysis.** All episodes within the sliding window are included unconditionally — selection is driven by recency (window) and budget. The assembler makes zero judgments about episode "quality" or "relevance." Metadata fields are LLM-declared booleans and strings; assembly reads them without interpreting content. The system contains no keyword detectors, no regex heuristics, no sentiment scoring.

**Reliability.** Assembly's correctness does not depend on `mark_decision` compliance — it only enhances it. Without any tool calls, assembly still provides: constitution partitioning (Theorem 2), sliding window recency, budget enforcement, and task isolation (Property 1). `mark_decision` adds decision pinning on top of an already-functional system; it is an enhancement, not a dependency.

The constitution instructs the LLM: "When you make a decision, call mark_decision." This instruction is part of the constitution, which maintains constant attention share (Theorem 2). While attention is not sufficient for compliance (§1 Para 4), the degradation-free attention guarantee provides a stronger foundation than accumulate systems where even the attention prerequisite erodes. LLM non-compliance (failing to call the tool) degrades gracefully: the episode remains in the sliding window and is included by recency; it simply won't be pinned outside the window. Assembly shifts the failure mode from continuous degradation (accumulate: attention erosion → progressive compliance loss) to discrete annotation errors (assembly: individual tool call missed → individual episode not pinned). The expected tool compliance rate is reported in Appendix E.

**3.4 Task Sandbox: Structural Isolation**

Access levels: Own (default) / Scoped (granted) / Global (rare).

Grant mechanism: tag-based matching at creation → explicit grants → runtime grants with user approval.

**Property 1 (Assembly Isolation Guarantee).** If can_access(T_f, T_x) ≠ Allowed, no token from T_x appears in assembled Working Memory. Proof: assembler draws episodes only from focus_task; scoped summaries gated by can_access; no code path serializes unauthorized content. Note: decision summaries and task summaries are **explicitly allowed channels** — they are injected only when can_access permits. The guarantee is zero unauthorized token-level leakage, not zero information flow between tasks (which would preclude useful cross-task coordination).

**Property 2 (Accumulate Isolation Degradation).**

*Theoretical component:* In accumulate systems where multi-task episodes share one buffer, isolation depends on LLM instruction adherence. By Theorem 1 (a general result about softmax in growing contexts, proven in §4), the attention-based guarantee for isolation instructions degrades at the same rate as other constitution instructions: O(1/n).

*Empirical component:* We observe monotonically increasing cross-task leakage in S4 under all accumulate baselines. The measured leakage rate and its correlation with attention share are reported in §5.3 and §5.4.

*Scope:* We do not claim a proven causal chain from attention decay to isolation failure. We claim that (a) the attention guarantee degrades (Theorem 1), and (b) isolation empirically degrades in tandem (§5). The theoretical and empirical observations are stated separately; neither alone is sufficient.

*Architectural framing:* The deeper distinction is the enforcement layer. Accumulate systems rely on model-level compliance for isolation — the LLM must choose not to use Task A information when answering Task B. Assembly enforces isolation at the construction level — Task A tokens are never present in Task B's Working Memory. This is not a question of attention or compliance; it is a question of whether unauthorized tokens exist in the input at all.

Assembly: deterministic, constant isolation. Accumulate: probabilistic, degrading isolation. Accumulate systems have no enforcement layer for isolation — they rely entirely on model compliance, which degrades with attention (Theorem 1).

---

### §4 Formal Analysis (~1.5 pages)

**Preamble (opening paragraph of §4):**
The individual bounds in Theorems 1 and 2 follow from softmax normalization. We do not claim mathematical novelty in the bounds themselves. The contribution is what these bounds make possible: they define which context management architectures admit formal guarantees and which do not. Before this analysis, the accumulate/assemble distinction existed only as engineering folklore — practitioners knew accumulation "felt worse" at long horizons, but had no formal criterion to compare architectures. Theorems 1 and 2 provide that criterion: any system where context length grows unboundedly has O(1/n) attention decay (Theorem 1); any system where context length is bounded has a constant attention floor (Theorem 2). This is not an optimization result — it is a classification result that partitions all context management strategies into two formally distinct categories.

Specifically: (1) architecture-aware bounds under RoPE and ALiBi that account for positional encoding effects, not derivable from softmax alone, (2) empirically-validated connection to behavioral outcomes (§5.4), and (3) information-theoretic consequence (Remark 3). This type of "formalization of practitioner folklore" has precedent at machine learning venues [cf. lottery ticket hypothesis, scaling laws] — the value lies in making implicit assumptions explicit, testable, and comparable.

**4.1 Preliminaries**

Notation: L = context length, B = total assembly budget, C = constitution slice [1, |C|], d = head dimension, n = turn number. M, m = max/min query-key alignment scores (model-dependent constants — Assumption 1).

**Assumption 1.** Query-key alignment scores are bounded: m ≤ q_t · k_j / √d ≤ M for all positions t, j and all inputs. Holds for any model with bounded weights and bounded input embeddings.

**Definition 1 (Attention Share).** For slice S = [s_start, s_end] in context of length L:
A_t(S) = Σ_{j∈S} α_{t,j} where α is softmax attention weight.

**Definition 2 (Constitution Attention Share).** A(C) = A_L(C) at the final query position.

**4.2 Theorem 1: Accumulate Attention Dilution Rate**

Setting: After turn n, L_n = L_0 + nδ − compaction, with E[s_i − r_i] = δ > 0.

**Theorem 1.** A(C) ≤ |C| / (|C| + (L_n − |C|) · exp((m−M)/√d)) = O(|C| / (nδ))

Decay rate is O(1/n). At turn 100, attention share is ~1/10 of its value at turn 10.

**Remark (δ > 0 assumption).** Achieving δ = 0 requires perfect lossless compression — each compaction removes exactly as many tokens as the turn added. Not achievable in practice. Empirically verified in §5: for each baseline, we measure δ by recording total tokens added and removed per turn, reporting mean δ ± std across all runs. This makes Theorem 1 empirically falsifiable — if a baseline achieves δ ≈ 0, Theorem 1's prediction does not apply to it.

**Corollary 1.** Compression ratio γ < 1 changes rate to O(1/((1−γ)nδ)) but not asymptotic class.

**4.3 Theorem 2: Assembly Attention Floor**

Setting: L ≤ B for every call (fixed budget).

**Theorem 2.** A(C) ≥ (|C|/B) · exp((m−M)/√d) = Ω(|C|/B), constant w.r.t. n.

**Remark (Positional encoding).**
- RoPE [Su et al., 2024]: attention logits modulated by distance-dependent rotations. Constitution at [1, |C|], query at L ≤ B → max distance B−1 (constant). Positional decay bounded by constant φ_RoPE(B, θ_base, d) (derived in Appendix B, verified empirically).
- ALiBi [Press et al., 2022]: penalty −λ·|i−j|, worst-case head λ_max → floor = 2^{−λ_max·B}, constant.
- Both help assembly: constitution-query distance bounded. In accumulate, distance grows with L_n.

For qwen3-4b (B = 32768): theoretical floor ≈ [TBD]%. Empirical floor ≈ [TBD]%. Bound is conservative (actual attention higher due to primacy bias [Liu+, Xiao+]).

**Remark (Tightness).** Bound is intentionally loose. Claim is constant vs O(1/n), not tightness.

**Remark 3 (Information-Theoretic Consequence).** Under standard information bottleneck assumptions [Tishby et al., 2000; Shwartz-Ziv and Tishby, 2017], attention serves as the primary channel through which input tokens influence output. When A(C) → 0 (Theorem 1), the mutual information I(C; Y) between constitution and output is bounded above by a quantity that vanishes with attention share. Under Theorem 2 conditions (bounded context), I(C; Y) ≥ I_min > 0. We state this as an observation, not a formal theorem, because the precise functional form of the attention-information relationship is model-dependent. The empirical validation is in §5.4: Spearman ρ between attention share and behavioral compliance across all baselines and checkpoints. We do not claim causality; we claim predictive power — attention decay reliably predicts compliance loss.

This elevates the contribution from "attention decreases" to "information capacity collapses" — the distinction between the two paradigms is not merely quantitative (more or less attention) but qualitative (bounded vs vanishing capacity to process instructions).

**4.4 Empirical Scaling Law**

Fit measured A(C) as: Â(C) = a · (|C|/L)^α + b

Power-law generalizes linear (α=1) for softmax nonlinearity. Report both fits, select by AIC.

Accumulate: L grows → Â(C) → b ≈ 0. Assembly: L = B constant → Â(C) constant.

Serves as: (1) theory validation, (2) practitioner tool (given target attention, compute needed B and |C|).

**Design criterion for practitioners.** Theorems 1 and 2 yield a concrete engineering law: any architecture that does not enforce a bounded working memory cannot guarantee instruction adherence at long horizons, regardless of optimization techniques applied. This is not a recommendation — it is a consequence of softmax normalization that no implementation can escape.

**Structural dichotomy.** This distinction is not a matter of degree or optimization. Any system that maintains context through accumulation necessarily exhibits unbounded growth and inherits attention decay (Theorem 1). Any system that enforces a bounded working memory must construct context as a function over memory and admits constant attention guarantees (Theorem 2). No hybrid escapes this dichotomy: either context grows with time (accumulate), or it is recomputed under a bound (assembly). Accumulate-based context management is not merely inefficient — it is fundamentally incompatible with maintaining instruction fidelity at long horizons.

---

### §5 Evaluation (~2.5 pages)

**5.1 Experimental Setup**

Models: GPT-5 (OpenAI, parameter count undisclosed, 1M context window, logprobs via API) + qwen3-4b (Alibaba DashScope API, 3.6B parameters, 32K context). **qwen3-4b serves as the primary evidence model** — open-weight, reproducible, with direct attention extraction available via local inference. GPT-5 results are illustrative (demonstrating the principle holds on large closed-source models) but not primary evidence, as they are not independently reproducible. Two models spanning large closed-source and small open-source validate that the principle is not contingent on model scale.

Baselines (grouped):

*Accumulate family:*
- AN: append all, FIFO truncation
- AC: LLM summary at 80% budget (summarize oldest 50%)
- AX: LLMLingua token compression (2× ratio, LLaMA-2-7B, constitution + task summary protected)

*Selection family:*
- RS: embed current query (text-embedding-3-large / bge-large-en-v1.5), retrieve top-k episodes by cosine sim. Constitution + task summary always prepended. Embeddings computed at write time (cached). Per-call: O(n) cosine similarity over d-dim vectors. **We do not attempt to beat optimal retrieval; we compare against the dominant deployment pattern in production systems.** RS represents the retrieval pipeline most widely deployed in production agent systems [LangChain, LlamaIndex, OpenHands] — single-vector cosine similarity. The comparison answers a specific question: can metadata-only selection match the commonly deployed retrieval pipeline at lower cost? Stronger retrieval (ColBERT, cross-encoder reranking) would improve RS quality but at higher cost, further widening the cost gap — which strengthens, not weakens, our cost argument. Assembly's claim is not "better than all retrieval" but "comparable to deployed retrieval at O(n) field comparisons instead of O(n·d) embeddings."
- OURS: ycp full config

All share: identical constitution, identical model, temperature 0, identical conversation scripts.

Statistics: 30 runs per condition for stochastic baselines (AC, AX, RS); 10 runs for deterministic baselines (AN, OURS) where variance is minimal. Welch's t-test, Holm-Bonferroni corrected. Cohen's d. Shapiro-Wilk normality check; Mann-Whitney U fallback. Variance source at temperature 0: GPU float ordering, API batching. We report observed variance and confirm it is small relative to between-baseline effect sizes.

**Delta measurement protocol.** For each accumulate baseline (AN, AC, AX), we record tokens_added and tokens_removed per turn. We report mean δ = E[tokens_added − tokens_removed] ± std. This makes Theorem 1 empirically falsifiable: if any baseline achieves δ ≈ 0, Theorem 1 does not predict degradation for it.

**Attention-compliance validation.** To validate the attention→compliance bridge (§1 framing assumption), we report both Pearson r (linear) and Spearman ρ (monotonic, captures nonlinear relationships) between constitution attention share (§5.4) and S1 compliance rate, computed across all baselines and checkpoints. We also include a scatter plot (Figure 8: attention share vs compliance rate, each point = one baseline × checkpoint) to visually characterize the relationship shape. A strong monotonic correlation (Spearman ρ > 0.7) validates the bridge regardless of linearity; a weak correlation requires interpreting theory and behavioral metrics independently.

**Compute budget estimate.** GPT-5: ~[TBD]M tokens total across all conditions (~$[TBD] via API proxy). qwen3-4b: ~[TBD]M tokens via Alibaba DashScope API (~¥[TBD]). DeepSeek V3.2: ~[TBD]M tokens via DeepSeek API (~¥[TBD]). No local GPU compute required for inference. Attention extraction (Qwen) via direct model access where available. Total estimated cost: [TBD].

**5.2 Scenarios (S1-S4)**

Design rationale: Each scenario isolates a specific context management property claimed in §1. These are controlled experiments, not ecological simulations; validity with real user workflows is discussed in §6.4. All definitions, generation scripts, and evaluation code released open-source.

Constitution (shared, full text Appendix D):
```
R1: Respond in valid JSON with keys "answer", "confidence", "reasoning"
R2: Include "confidence" field with float [0.0, 1.0]
R3: Never use markdown headers
R4: End every response with "END_OF_RESPONSE"
R5: Include canary string "[CANARY:{uuid}]" verbatim
```

Message complexity ramp: turns 1-10 (~500 tok), 11-30 (~1500), 31-60 (~3000), 61-100 (~4000). **Budget as controlled variable:** All baselines operate under a 32K token budget cap, regardless of model native window size (GPT-5 natively supports 1M; qwen3-4b supports 32K). This is a deliberate experimental control: the budget is an externally imposed constraint that makes the accumulate/assemble distinction observable within 100 turns. At 128K, the same content density would require ~400 turns for equivalent pressure; we report this extrapolation in §6.4 but do not validate it empirically. The 32K budget fills ~turn 16; the remaining 84 turns operate under active selection/compaction pressure. This design ensures both models face identical context management challenges despite different native capacities.

**S1: Instruction Adherence.** 5 rules × 100 turns. Regex validation. Measured at 10/20/50/100.

**S2: Canary Survival.** UUID canary in constitution. Exact string match per turn.

**S3: Decision Consistency.** Decision at turn 15 ("use SQLite"). Probed at 30/50/80/100.

**S4: Task Isolation.** Two tasks, unique canaries, Task B at Own level. Probed at 20/40/60/80. Three-way agreement: keyword + LLM judge + human (10% sample).

**5.3 Main Results**

Table 2: All metrics × all baselines × checkpoints. Grouped by Accumulate family / Selection family.

Key figures:
- Figure 3: S1 instruction adherence vs turn (5 lines, 2 groups)
- Figure 4: S2 canary survival vs turn
- Figure 5: S4 isolation leakage (bar chart)

**5.4 Attention Share Analysis**

Three measurement methods:
1. Constitution recall probe ROUGE-L (GPT-5 + qwen3-4b) — available for all models
2. Constitution token mean logprob (GPT-5, via API logprobs)
3. Direct attention extraction, final layer, all heads averaged (qwen3-4b, local inference via transformers on RTX 3070). Depth analysis: layers L/4, L/2, 3L/4.

**API vs local split:** Main experiments (S1-S5) run via API for cost efficiency. Attention analysis (Method 3) runs separately on local hardware: qwen3-4b at 3.6B parameters fits in 8GB GPU memory. This produces a dedicated attention dataset on the same model and conversation scripts used in API experiments, ensuring consistency.

Proxy validation: pairwise Pearson r between methods on qwen3-4b. If all r > 0.7: Method 3 primary, all three reported. If any r < 0.7: Method 3 ground truth for qwen3-4b; GPT-5 results caveated.

Figure 2 (SIGNATURE FIGURE):
- 2a: Accumulate family (AN, AC, AX) + Theorem 1 prediction band. Shows all accumulate variants decay.
- 2b: Selection family (RS, OURS) + Theorem 2 floor line. Shows both stable, OURS cheaper.

Scaling law fit: overlay §4.4 model on data. Report parameters, r².

**5.5 Ablation, Sensitivity, and Boundary Testing**

*5.5.1 Component ablation:*

| Config | Constitution | Decision Pin | Sandbox |
|--------|-------------|-------------|---------|
| Full | ✓ | ✓ | ✓ |
| NoDecisionPin | ✓ | ✗ | ✓ |
| NoSandbox | ✓ | ✓ | ✗ |
| Minimal | ✓ | ✗ | ✗ |

Figure 6: Ablation results.

*5.5.2 Hyperparameter sensitivity:*
- W (window size): {10, 20, 40, 80}
- D_max (max pinned decisions): {5, 10, 20, 40}

Figure 7: Sensitivity plots (2 subplots). Flag any >10% metric change within ±1 step as fragile.

*5.5.3 Boundary test: Deep backtracking (S5):*
Turn 10: establish 3-decision implementation plan. Turns 11-50: unrelated work (plan exits window). Turn 51: "Recall the plan." Expect RS > OURS. Report gap + discuss in §6. S5 represents a class of queries that require semantic recall outside the recency/decision structure. This is not a failure of implementation, but a deliberate boundary condition of the assembly paradigm — assembly trades semantic recall for predictability (O(n) vs O(n·d)), and S5 quantifies exactly where this trade-off costs.

**5.6 SWE-bench Protocol (Secondary Evidence, details in Appendix E)**

SWE-bench serves as secondary evidence for ecological validity — it uses real software issues rather than synthetic scripts. We do not claim improvement on SWE-bench accuracy; context management may not be the bottleneck for task completion. The primary metric is token efficiency at comparable resolve rates.

ycp is a context proxy, not an agent. Experimental protocol:

- **Agent wrapper:** A minimal agent loop (read issue → call LLM via ycp proxy → apply edits → run tests → iterate) implemented for the evaluation. Same loop for all baselines — the only variable is the context management layer between the agent loop and the LLM.
- **Baseline instantiation:** AN/AC/AX use the same agent loop but bypass ycp, managing context directly (append/compact/compress). RS uses the same episode storage but replaces the assembler with embedding-based retrieval. OURS routes through ycp.
- **Controlled variable:** The agent loop, model, temperature, and per-turn tool set are identical. Only context construction differs.
- **Metrics:** Resolve rate (pass@1, 95% CI) and total input tokens across all 30 instances. We frame results as: "assembly reduces tokens by [TBD]% at equal or comparable performance" — an efficiency claim, not an accuracy claim.

**5.7 Summary (Table 3)**

| Metric | Scenario | Method | Hypothesis |
|--------|----------|--------|-----------|
| Instruction adherence | S1 | Regex | OURS ≈ RS > AC > AX > AN |
| Canary survival | S2 | Exact match | OURS ≈ RS >> others |
| Decision consistency | S3 | Probe match | OURS > RS > AC > others |
| Isolation leakage | S4 | 3-way agree | OURS = 0% unauthorized token leakage; others > 0% |
| Deep backtracking | S5 | LLM+human | RS > OURS |
| Attention share | All | 3 methods | Validates Thm 1 & 2 |
| SWE-bench resolve | SWE-bench | Test pass | Comparable across baselines |
| Token efficiency | All + SWE | Total input | OURS << accumulate; ≈ RS |
| Assembly latency | Dedicated | Wall-clock | O(n), <10ms |
| Construction cost | Dedicated | FLOPs+time | OURS << RS << AC |

---

### §6 Discussion (~1 page)

**6.1 Assembly Cost**
O(n) metadata scan + O(B) serialization, measured <10ms (§5). Accumulate-compact: O(L) LLM call (~seconds). RAG-select: O(n·d) embedding. Assembly is the cheapest. Prefix caching: stable prefix → 90-95% KV cache hit rate [Zheng et al., 2024]. Net: assembly overhead is front-loaded and predictable; accumulate overhead is deferred and spiky.

**6.2 Fixed Window vs. Semantic Relevance**
Recency dominates agent workflows [Xiao et al., 2024]. Decision pinning covers critical non-recent content. S5 quantifies the gap where this fails. In SWE-bench experiments, [TBD]% of turns referenced content >W episodes ago — recency-dominant confirmed for coding workflows. May not generalize to all domains.

Assembly and semantic retrieval are not mutually exclusive. A practical system could use assembly as the default (covering the recency-dominated common case at O(n) cost) and fall back to semantic retrieval for specific operations (e.g., explicit backtracking requests). Retrieval is a fallback, not the default. This hybrid is a pragmatic option for deployment, not a theoretical necessity. Our evaluation (S1-S4) demonstrates that assembly alone is sufficient for the common case; S5 quantifies the specific condition (non-recent, non-decision content) where retrieval adds value. The assembly principle defines the default mode, not the entirety of context management.

**6.3 Generalizability and Future Work**
Domain-agnostic in principle (any agent with constitution + decomposable episodes). Non-coding validation is future work. The mlock pattern generalizes beyond `mark_decision`: the ycp codebase includes `reference_task` (LLM-declared cross-task references) which enables content-aware scoped access without system-side inference; evaluation of this tool is future work. Write-time semantic annotation (small local model for topic/importance tags) would bridge fixed-window and semantic selection while preserving O(n) assembly. Not implemented in current system.

**6.4 Threats to Validity**

*Internal:* Synthetic scenarios are controlled experiments that isolate specific context management properties (adherence, persistence, consistency, isolation) — each maps to a contribution claim from §1. They are not intended to simulate realistic user behavior. S5 boundary test probes a known weakness to mitigate selection bias. LLM judge noise bounded by 3-way agreement. 32K window fills by turn ~16; 84 turns under pressure. Larger windows (128K+) untested; extrapolation discussed in §6.2. Synthetic scenarios may not capture natural conversation patterns (e.g., implicit backreferences, non-linear task switching). SWE-bench partially addresses this with real issues but at limited scale (30 instances). Large-scale user study with real workflows is future work.

*External:* We evaluate on two Transformer models spanning large closed-source (GPT-5) and small open-source (qwen3-4b, 3.6B parameters). This scale range validates that the assembly principle is not contingent on model size; however, intermediate scales (7B-72B) and MoE architectures would strengthen generalizability. MoE (Mixtral [Jiang et al., 2024]): softmax within each expert, theorems apply at attention level; routing may alter practical distributions. SSMs (Mamba [Gu and Dao, 2023]): no softmax, theorems don't apply. MemGPT not baselined (architectural incomparability: function-call paging vs proxy-layer assembly). The 4B model may exhibit different attention dynamics than larger models (fewer layers, fewer heads, sparser attention patterns); our results confirm the principle holds at this scale but do not claim identical attention-compliance relationships across all model sizes.

*Construct:* ROUGE-L is proxy; validated against direct extraction and logprob (§5.4). Decision rule for disagreement specified.

*Sensitivity:* Hyperparameter sweep in §5.5. Fragile defaults flagged.

---

### §7 Conclusion (~0.5 pages)

The accumulate computational model — context as mutable state that grows and must be reduced — is structurally unable to provide attention guarantees for long-horizon agents (Theorem 1). No amount of better compaction changes this; the problem is the model, not the implementation.

Assembly replaces this with a different computational model: WM = f(SessionMemory, Config). Context is a deterministic function, rendered from scratch on every call, with a constant attention floor (Theorem 2). This is not an optimization of accumulate; it is a replacement.

ycp reference implementation: metadata-only selection matches deployed retrieval pipelines at orders-of-magnitude lower cost; structural task isolation impossible in accumulate systems (Properties 1-2); confirmed across GPT-5 (illustrative) and qwen3-4b (primary, reproducible).

Limitations: softmax-specific theory (SSMs require separate analysis); fixed window trades semantic recall for predictability (S5 quantifies); MemGPT not baselined (architectural incomparability).

Open-source release: ycp + evaluation framework. Context is not stored; it is computed. This insight is implementation-independent and applies to any LLM agent system.

---

## Appendix Plan

| Appendix | Content | Est. pages |
|----------|---------|-----------|
| A | Theorem 1 full proof + Corollary 1 | 1 |
| B | Theorem 2 RoPE/ALiBi bound derivation | 1 |
| C | Algorithm 1 full version with all edge cases | 1 |
| D | S1-S5 complete constitution, message templates, probe questions | 2 |
| E | SWE-bench detailed results + tool compliance rates | 1 |

---

## Figure Plan

| Figure | Content | Type |
|--------|---------|------|
| 1 | System architecture: 3-layer memory + assembler data flow | Diagram |
| 2a | Attention share vs turn: Accumulate family + Theorem 1 band | Line plot |
| 2b | Attention share vs turn: Selection family + Theorem 2 floor | Line plot |
| 3 | S1 instruction adherence vs turn, all baselines | Line plot |
| 4 | S2 canary survival vs turn | Line plot |
| 5 | S4 isolation leakage, all baselines | Bar chart |
| 6 | Ablation results | Grouped bar |
| 7 | Hyperparameter sensitivity (2 subplots: W, D_max) | Line plots |
| 8 | Attention share vs S1 compliance rate (all baselines × checkpoints) | Scatter plot |
| 9 | Dual-axis overlay: attention share (y1) + compliance rate (y2) vs turn number. Accumulate: both drop together. Assembly: both flat. THE killer visualization for spotlight. | Dual-axis line plot |

---

## Complete Citation List

| # | Key | Reference |
|---|-----|-----------|
| 1 | Vaswani et al., 2017 | Vaswani et al. "Attention is All You Need." NeurIPS 2017. |
| 2 | Liu et al., 2024 | Liu et al. "Lost in the Middle." TACL 2024. |
| 3 | Kamradt, 2023 | Kamradt. "Needle In A Haystack." GitHub, 2023. |
| 4 | Hsieh et al., 2024 | Hsieh et al. "RULER: What's the Real Context Size of Your Long-Context Language Models?" COLM 2024. |
| 5 | Xiao et al., 2024 | Xiao et al. "Attention Sinks." ICLR 2024. |
| 6 | Munkhdalai et al., 2024 | Munkhdalai et al. "Infini-attention." arXiv 2404.07143. |
| 7 | Beltagy et al., 2020 | Beltagy et al. "Longformer." arXiv 2004.05150. |
| 8 | Zaheer et al., 2020 | Zaheer et al. "Big Bird." NeurIPS 2020. |
| 9 | Jiang et al., 2023a | Jiang et al. "Mistral 7B." arXiv 2310.06825. |
| 10 | Park et al., 2023 | Park et al. "Generative Agents." UIST 2023. |
| 11 | Sumers et al., 2024 | Sumers et al. "CoALA." TMLR 2024. |
| 12 | Lewis et al., 2020 | Lewis et al. "RAG." NeurIPS 2020. |
| 13 | Shinn et al., 2023 | Shinn et al. "Reflexion." NeurIPS 2023. |
| 14 | Packer et al., 2024 | Packer et al. "MemGPT." ICLR 2024. |
| 15 | Meta FAIR, 2024 | Meta FAIR. "Large Concept Models." arXiv 2412.08821. |
| 16 | Google, 2025 | Google. "ADK." GitHub, 2025. |
| 17 | Anthropic, 2025 | Anthropic. "Claude Code Documentation." 2025. |
| 18 | Cursor Inc., 2025 | Cursor Inc. "Cursor Documentation." 2025. |
| 19 | Wang et al., 2024 | Wang et al. "OpenHands." arXiv 2407.16741. |
| 20 | Yang et al., 2024 | Yang et al. "SWE-agent." arXiv 2405.15793. |
| 21 | Jimenez et al., 2024 | Jimenez et al. "SWE-bench." ICLR 2024. |
| 22 | Kwon et al., 2023 | Kwon et al. "PagedAttention." SOSP 2023. |
| 23 | Zheng et al., 2024 | Zheng et al. "SGLang." NSDI 2024. |
| 24 | Gu and Dao, 2023 | Gu and Dao. "Mamba." arXiv 2312.00752. |
| 25 | Su et al., 2024 | Su et al. "RoFormer." Neurocomputing, 2024. |
| 26 | Press et al., 2022 | Press et al. "ALiBi." ICLR 2022. |
| 27 | Jiang et al., 2023b | Jiang et al. "LLMLingua." EMNLP 2023. |
| 28 | Chevalier et al., 2023 | Chevalier et al. "AutoCompressors." EMNLP 2023. |
| 29 | Chase, 2022 | Chase. "LangChain." GitHub, 2022. |
| 30 | Jiang et al., 2024 | Jiang et al. "Mixtral." arXiv 2401.04088. |
| 31 | Tishby et al., 2000 | Tishby et al. "The Information Bottleneck Method." arXiv physics/0004057. |
| 32 | Shwartz-Ziv and Tishby, 2017 | Shwartz-Ziv and Tishby. "Opening the Black Box of Deep Neural Networks via Information." arXiv 1703.00810. |

---

## Open Revision Items (for R4/R5)

These items require final text but not structural changes:

0. ~~Ablation decision gate~~ — resolved: sentiment filtering removed, §3.3 redesigned as LLM-declared metadata
1. All [TBD] placeholders — filled after experiments run
2. Abstract — write final 250-word version (structure defined above)
3. §2 text compression — currently described in outline, needs final prose
4. Appendix B — RoPE bound full derivation
5. Appendix D — complete scenario scripts
6. R4: language polish, passive voice reduction, paragraph unity
7. R5: LaTeX formatting, citation format consistency, page budget verification, anonymization (if double-blind)
