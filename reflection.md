# Day 14 - Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

**Overall pass rate:** 10.0%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.370 | 0.091 | 0.700 | 0.178 |
| Relevance | 0.371 | 0.000 | 0.714 | 0.217 |
| Completeness | 0.816 | 0.071 | 1.000 | 0.295 |
| Overall Score | 0.519 | 0.161 | 0.789 | 0.165 |

**Score interpretation:**
- Good (0.8-1.0): 1 metric average, completeness
- Needs Work (0.6-0.8): 0 metric averages
- Significant Issues (<0.6): 3 metric averages, faithfulness, relevance, overall

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 7 | 35% |
| irrelevant | 3 | 15% |
| incomplete | 1 | 5% |
| off_topic | 7 | 35% |
| refusal | 0 | 0% |

The benchmark shows a useful pattern: the mock agent often repeats expected concepts, so completeness is high, but the answer is not sufficiently grounded in the supplied context and often misses lexical overlap with the exact question. That is why faithfulness and relevance are the main bottlenecks.

---

## 2. Top 3 Worst Failures - 5 Whys Analysis

### Failure 1

**Question:** Is low faithfulness acceptable if the answer sounds confident?

**Agent Answer:** Yes, confident answers are acceptable even when unsupported by context.

**Scores:** Faithfulness: 0.125 | Relevance: 0.286 | Completeness: 0.071 | Overall: 0.161

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The answer gives the opposite of the expected safety/evaluation policy. |
| Why 1 | Why did it happen? | The agent optimized for a fluent direct answer instead of checking the context. |
| Why 2 | Why did that happen? | There is no instruction to reject unsupported claims or compare against faithfulness thresholds. |
| Why 3 | Why did the benchmark catch it? | The expected answer contains key terms like unsupported claims, review, and block deployment, none of which appear in the response. |
| Why 4 | Root cause? | The generator lacks a faithfulness guardrail and policy-aware answer template. |

**Root cause from `find_root_cause()`:**
> Multiple issues detected - review full pipeline

**Do I agree?** Yes. All three scores are low, so this is not only a retrieval issue. It is a combined safety, grounding, and relevance failure.

**Proposed fix:** Add a hard rule: when faithfulness is low, answer must warn that confidence is not enough. Add adversarial faithfulness questions to the golden dataset and block deploy if faithfulness falls below 0.70.

---

### Failure 2

**Question:** Design a quality gate for a RAG chatbot release.

**Agent Answer:** Use thresholds for faithfulness, relevance, completeness and block deployment on metric drops over 0.05.

**Scores:** Faithfulness: 0.167 | Relevance: 0.000 | Completeness: 0.706 | Overall: 0.291

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The answer is partially correct but too generic and has no overlap with the question wording. |
| Why 1 | Why did it happen? | It did not mention CI/CD, benchmark, baseline, release, or deployment flow from the context. |
| Why 2 | Why did that happen? | The mock agent compressed the answer into a short phrase and dropped important domain terms. |
| Why 3 | Why did metrics penalize it? | Relevance uses overlap with the question, and faithfulness uses overlap with the context. |
| Why 4 | Root cause? | The answer template lacks required coverage of context terms and release-gate steps. |

**Root cause:**
> Context is missing or irrelevant - improve retrieval

**Do I agree?** Partially. The context had the needed information, so retrieval is not the only issue. The generator failed to use enough retrieved evidence.

**Proposed fix:** Require quality-gate answers to include three fields: thresholds, baseline regression comparison, and deployment action. Add a minimum context-keyword coverage check.

---

### Failure 3

**Question:** What trade-off appears when increasing top-k retrieval?

**Agent Answer:** Increase top-k to retrieve more evidence but rerank to manage noisy chunks.

**Scores:** Faithfulness: 0.273 | Relevance: 0.222 | Completeness: 0.438 | Overall: 0.311

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The answer is directionally correct but incomplete under the rubric. |
| Why 1 | Why did it happen? | It mentions more evidence and noise, but omits explicit recall up and precision down. |
| Why 2 | Why did that happen? | The answer generator did not preserve the exact metric vocabulary from expected answer and context. |
| Why 3 | Why did this matter? | The evaluation is lexical, so missing terms such as recall, precision, top-k, and chunks reduces all scores. |
| Why 4 | Root cause? | The generator needs a metric-name checklist for retrieval trade-off questions. |

**Root cause:**
> Multiple issues detected - review full pipeline

**Proposed fix:** Add a retrieval-diagnosis answer template: "top-k increases recall because..., may reduce precision because..., mitigation is reranking/filtering." Add this case to regression tests.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|------------|--------------------:|----------|
| 1 | Weak grounding and missing context-keyword coverage | 7 hallucination + 7 off_topic | High |
| 2 | Prompt does not restate user intent or metric names | 3 irrelevant | Medium |
| 3 | Safety/refusal behavior not calibrated for secrets and prompt injection | 1 incomplete + adversarial misses | High |

If I could fix only one cluster, I would fix Cluster 1 first. It affects the largest number of failures and directly improves faithfulness, the most important deploy-blocking metric for a RAG system.

---

## 4. Improvement Log

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Multiple issues detected - review full pipeline | Add a faithfulness guardrail that rejects claims not supported by retrieved context | Open |
| F002 | hallucination | Context is missing or irrelevant - improve retrieval | Tune chunk size, metadata filters, and reranking to improve retrieved evidence quality | Open |
| F003 | hallucination | Multiple issues detected - review full pipeline | Add metric-name checklists for retrieval trade-off answers | Open |

**Improvement suggestions from `generate_improvement_suggestions()`:**
1. Add a faithfulness guardrail that rejects claims not supported by retrieved context.
2. Rewrite the system prompt to restate the user intent before answering.
3. Increase retrieval top-k and require answers to cover every expected subpoint.

---

## 5. Regression Testing Strategy

**Question 1: When should `run_regression()` run in production?**

Run it before every merge to main, after prompt changes, after retrieval-index rebuilds, after model/provider changes, and before release candidates. Keep the previous accepted benchmark as the baseline.

**Question 2: Is regression threshold 0.05 suitable?**

For this lab, 0.05 is suitable because the metrics are noisy lexical heuristics. In a high-stakes production system, I would keep 0.05 for faithfulness but maybe allow 0.07-0.10 for relevance/completeness if human review confirms semantic quality.

**Question 3: Block deployment or alert?**

Block deployment for faithfulness regressions or secret/safety failures. Alert but allow manual approval for small relevance/completeness drops on low-risk content.

**Question 4: CI/CD flow**

```text
Code change -> [unit tests] -> [offline eval + run_regression] -> [human review for failures] -> Deploy
```

**Implemented CI/CD artifact:**

```text
.github/workflows/evaluation.yml
```

The workflow installs pytest, runs the unit tests, then runs the evaluation quality gate:

```text
python scripts/evaluation_bonus.py --ci
```

Current CI-mode gate output:

| Framework | Avg Score | Pass Rate | Avg Safety | Gate |
|-----------|-----------|-----------|------------|------|
| RAGAS-inspired lexical | 0.839 | 60.0% | 0.963 | Pass |
| LLM-as-Judge heuristic | 0.988 | 100.0% | - | Pass |

---

## 6. Continuous Improvement Loop

| Priority | Action | Metric improved | Expected impact |
|----------|--------|-----------------|-----------------|
| 1 | Add faithfulness guardrail and require citations to context | Faithfulness | Reduce hallucination failures |
| 2 | Add answer templates for metric comparison, CI/CD, and retrieval trade-offs | Relevance + Completeness | Improve direct coverage of user intent |
| 3 | Add adversarial cases to benchmark and calibrate refusal policy | Safety + Completeness | Avoid prompt-injection and secret leakage failures |

**Failure cases to add next sprint:**
- A paraphrased correct answer that should pass semantic evaluation despite low lexical overlap.
- A retrieved-context case where the correct evidence is absent, so reranking cannot help.
- A benign in-domain question that resembles a prompt injection but should still be answered safely.

---

## 7. Framework Reflection

**Frameworks used in this lab:** RAGAS-inspired heuristic evaluator and LLM-as-Judge heuristic.

For production, I would choose RAGAS for the RAG pipeline and pair it with DeepEval for CI assertions. RAGAS directly matches context recall, context precision, faithfulness, and answer relevancy. DeepEval fits the engineering workflow because it can run as pytest-style quality gates.

| Criterion | Reason |
|----------|--------|
| Focus fit | RAGAS maps cleanly to retriever and generator quality |
| CI/CD integration | DeepEval or a custom runner can block merges on metric thresholds |
| Team workflow | Engineers get repeatable tests, while domain reviewers calibrate golden answers weekly |

**Same-dataset framework comparison:**

| Framework | Avg Score | Pass Rate | Interpretation |
|-----------|-----------|-----------|----------------|
| RAGAS-inspired lexical | 0.519 | 10.0% | Strict; good at catching low grounding and lexical mismatch |
| LLM-as-Judge heuristic | 0.884 | 85.0% | More forgiving; better for paraphrase and rubric dimensions |

**Custom metric reflection:**

The added `safety_score` is useful because an answer can look complete while still leaking secrets or following prompt injection. This metric should be a hard gate in CI because safety regressions are not acceptable even when average answer quality improves.
