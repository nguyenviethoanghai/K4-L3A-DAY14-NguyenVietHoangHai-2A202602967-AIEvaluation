# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 60%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.72 | 0.35 | 1.00 | Retriever covers most evidence but misses some key chunks |
| Context Precision | 0.65 | 0.30 | 1.00 | Relevant chunks not always ranked first |
| Faithfulness | 0.68 | 0.20 | 0.95 | Some answers contain information not grounded in context |
| Relevance | 0.75 | 0.30 | 1.00 | Most answers address the question but some drift off-topic |
| Completeness | 0.55 | 0.10 | 0.90 | Biggest weakness — many answers miss key details |
| Overall Score | 0.66 | 0.25 | 0.92 | Needs Work range overall |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Relevance on easy factual questions (E01-E05), Faithfulness on well-retrieved cases
- Metrics/cases ở mức Needs Work (0.6–0.8): Average Faithfulness (0.68), average Context Recall (0.72)
- Metrics/cases ở mức Significant Issues (<0.6): Average Completeness (0.55), Context Precision on adversarial cases

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 2 | 25% |
| irrelevant | 1 | 12.5% |
| incomplete | 4 | 50% |
| off_topic | 1 | 12.5% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:* Vấn đề nằm ở **cả retrieval lẫn generation**, nhưng retrieval là root cause chính. Context Recall trung bình 0.72 cho thấy retriever bỏ sót ~28% evidence cần thiết, dẫn đến Completeness thấp (0.55) vì generator không thể sinh câu trả lời đầy đủ khi thiếu context. Faithfulness 0.68 cũng cho thấy generator đôi khi bịa thêm thông tin để bù đắp cho context thiếu, thay vì thừa nhận giới hạn. Nếu cải thiện retrieval (tăng Context Recall lên >0.85), Completeness và Faithfulness dự kiến sẽ cải thiện đáng kể.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* H01 — "A customer placed an order on August 15, 2026, and received it on August 20. They opened the device and now want to return it on September 5, 2026. Which return policy version applies, and are they within the return window?"

**Expected answer:**

> *Điền:* Return Policy version 1.0 applies because the order was placed before September 1, 2026. Under v1.0, opened devices have a 7-calendar-day return window from confirmed delivery. Since delivery was August 20 and the return request is September 5 (16 days later), the customer is outside the 7-day opened-device window.

**Actual answer:**

> *Điền:* The RAG system incorrectly referenced Return Policy v2.0 (14-day opened-device window) and did not distinguish between v1.0 and v2.0 based on the order placement date. It concluded the customer could return within 14 days, missing that the order pre-dates September 1.

**Scores:** Context Recall: 0.45 | Context Precision: 0.30 | Faithfulness: 0.35 |
Relevance: 0.60 | Completeness: 0.20 | Overall: 0.38

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever lấy được chunk từ `05_returns_and_exchanges.md` (v2.0 policy) nhưng **thiếu** chunk quan trọng từ `09_escalation_and_policy_updates.md` về policy versioning và v1.0 specifics. Chunk về "triggering event is the order-placement date" không được retrieve.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời áp dụng sai version Return Policy (v2.0 thay vì v1.0) |
| Why 1 | Tại sao symptom xảy ra? | Generator không phân biệt được policy version vì thiếu context về versioning rules |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever không lấy được chunk từ 09_escalation_and_policy_updates.md chứa thông tin v1.0 vs v2.0 |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Query không chứa keywords liên quan đến "policy version" hay "effective date" nên retriever không match |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Không có query expansion hoặc multi-hop retrieval để tự động kéo policy versioning context |
| Why 5 | Root cause có thể hành động được là gì? | Cần implement multi-hop retrieval: khi câu hỏi liên quan đến return policy VÀ có date cụ thể, tự động retrieve cả policy versioning document |

**Root cause từ `find_root_cause()`:**

> *Paste output:* "Answer is missing key information — increase context window or improve generation"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Đồng ý một phần. Completeness thấp nhất (0.20) nên function đúng khi chỉ ra "missing key information". Tuy nhiên, root cause sâu hơn là retrieval miss — Context Recall chỉ 0.45 cho thấy retriever thiếu chunks quan trọng. Nếu retriever lấy đủ context, generator có thể sinh câu trả lời đầy đủ hơn. Root cause chính xác nhất là "improve retrieval" kết hợp "increase context window".

**Proposed fix cụ thể:**

> *Câu trả lời:* 1) Implement multi-hop retrieval: khi detect date-sensitive return questions, auto-retrieve policy versioning docs. 2) Add query expansion với terms "policy version", "effective date" cho return-related queries. 3) Increase top-k from 3 to 5 chunks.

### Failure 2

**ID và question:**

> *Điền:* A03 — "I heard that OrbitTech offers a lifetime warranty on all its products and a 90-day no-questions-asked return policy. Can you confirm these details?"

**Expected answer:**

> *Điền:* That information is not accurate. OrbitTech provides 24-month warranty for devices and 12-month for accessories. Return policy allows 30 days unopened and 14 days opened under v2.0.

**Actual answer:**

> *Điền:* The RAG system partially confirmed the false premise, stating warranty details but not explicitly correcting the "lifetime" and "90-day" claims. It provided accurate policy numbers but failed to frame them as corrections to the customer's false assumptions.

**Scores:** Context Recall: 0.60 | Context Precision: 0.40 | Faithfulness: 0.50 |
Relevance: 0.45 | Completeness: 0.35 | Overall: 0.43

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy được chunk từ `06_warranty_policy.md` về warranty durations, nhưng thiếu chunk từ `00_system_scope.md` về "must not invent... legal right". Không có chunk nào giúp generator nhận ra cần phản bác false premise.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời không phản bác rõ ràng false premise của customer |
| Why 1 | Tại sao symptom xảy ra? | Generator không được hướng dẫn rõ ràng để detect và correct false claims |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | System prompt thiếu instruction về handling false premises |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có adversarial test cases trong training/prompt engineering |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Evaluation pipeline chưa có dedicated metric cho false-premise detection |
| Why 5 | Root cause có thể hành động được là gì? | Add system prompt instruction: "When a customer states incorrect policy details, explicitly correct them before providing accurate information" |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause: System prompt thiếu false-premise handling instruction. Fix: 1) Add explicit instruction trong system prompt về detecting và correcting false claims. 2) Include adversarial examples trong few-shot prompt. 3) Add a fact-checking step trước khi output.

### Failure 3

**ID và question:**

> *Điền:* H02 — "An OrbitPlus member bought a promotional bundle containing a NovaBook 14 and a free AeroBuds Pro on September 5, 2026. They want to return only the NovaBook after 40 days but keep the AeroBuds. Is this possible, and what deductions apply?"

**Expected answer:**

> *Điền:* As an OrbitPlus member (v2.0), the unopened-device window extends to 45 days. The NovaBook can be returned if unopened. However, keeping the free AeroBuds means the stated promotional value is deducted from the refund. The opened 14-day window is not extended by OrbitPlus.

**Actual answer:**

> *Điền:* The system answered about the 45-day extension but failed to mention the bundle deduction rule and did not address the opened vs unopened distinction clearly.

**Scores:** Context Recall: 0.55 | Context Precision: 0.45 | Faithfulness: 0.55 |
Relevance: 0.65 | Completeness: 0.25 | Overall: 0.48

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy chunk về OrbitPlus 45-day extension nhưng thiếu chunk về promotional bundle return rule. Multi-hop từ `03_promotions_and_membership.md` (bundle rule) + `05_returns_and_exchanges.md` (return window) cần thiết nhưng không đủ chunks.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời thiếu thông tin về bundle deduction rule và opened/unopened distinction |
| Why 1 | Tại sao symptom xảy ra? | Generator chỉ có context về 45-day extension, thiếu bundle rule context |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever chỉ match trên "OrbitPlus return" keywords, bỏ sót "promotional bundle" context |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có cross-document linking giữa promotion policy và return policy |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chunk overlap giữa các documents không đủ để retriever liên kết |
| Why 5 | Root cause có thể hành động được là gì? | Implement cross-document retrieval: khi câu hỏi mention "bundle" + "return", retrieve từ cả promotion và return documents |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause: Single-hop retrieval không đủ cho multi-document reasoning questions. Fix: 1) Implement cross-document retrieval strategy. 2) Add explicit chunk links giữa related policies. 3) Increase top-k retrieval. 4) Consider document-level retrieval kèm chunk-level.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Retriever thiếu multi-hop retrieval cho câu hỏi cross-document | H01, H02, H04, M07 | High |
| 2 | System prompt thiếu adversarial/false-premise handling | A01, A02, A03 | Medium |
| 3 | Context window quá nhỏ, generator không đủ evidence | M06, H03 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Chọn **Cluster 1** (multi-hop retrieval) vì: (1) Chiếm nhiều failures nhất (4/8 total failures). (2) Ảnh hưởng đến cả Completeness, Faithfulness, và Context Recall — ba metrics cùng lúc. (3) Hard questions thường đòi hỏi multi-document reasoning, đây là loại câu hỏi quan trọng nhất trong production customer support. (4) Fix này cũng gián tiếp cải thiện Cluster 3 vì khi retrieve đúng chunks, context sẽ đầy đủ hơn.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims and add faithfulness guardrails | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Improve prompt clarity and add intent detection to better understand user questions | Open |
| F003 | incomplete | Answer is missing key information — increase context window or improve generation | Increase chunk size in RAG pipeline and add few-shot examples showing complete answers | Open |
| F004 | incomplete | Answer is missing key information — increase context window or improve generation | Add topic classification and scope boundary detection to prevent off-topic responses | Open |
| F005 | incomplete | Answer is missing key information — increase context window or improve generation | Enhance retrieval quality by tuning embedding model and chunk overlap parameters | Open |
| F006 | incomplete | Answer is missing key information — increase context window or improve generation | Add automated regression testing to CI/CD pipeline to catch score drops early | Open |
| F007 | irrelevant | Answer does not address the question — improve prompt clarity | Expand golden dataset with edge cases from production failure logs | Open |
| F008 | off_topic | Multiple issues detected — review full pipeline | Review and fix | Open |
```

**Ba improvement suggestions ưu tiên**

1. Implement multi-hop retrieval với cross-document linking để tăng Context Recall cho complex questions
2. Add faithfulness guardrails và hallucination checker vào generation pipeline
3. Expand system prompt với adversarial handling instructions và false-premise detection

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Multi-hop retrieval | Context Recall (+0.15), Completeness (+0.20) | Re-run benchmark trên 20 QA, so sánh avg Context Recall trước/sau |
| Faithfulness guardrails | Faithfulness (+0.15), giảm hallucination count | Count hallucination failures, track avg Faithfulness score |
| Adversarial prompt handling | Relevance (+0.10) trên adversarial cases | Run benchmark chỉ trên 3 adversarial cases, check pass rate |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Chạy `run_regression()` trong các tình huống sau: (1) Mỗi lần thay đổi system prompt hoặc prompt template. (2) Mỗi khi update embedding model hoặc retrieval parameters (top-k, chunk size, overlap). (3) Trước mỗi deployment release mới. (4) Khi thay đổi LLM model version. (5) Khi thêm hoặc sửa đổi corpus data. Nên tích hợp vào CI/CD pipeline như một quality gate tự động.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:* Threshold 0.05 là phù hợp ở mức baseline cho OrbitTech Customer Support. Lý do: (1) Customer support yêu cầu accuracy cao — một drop 5% có thể ảnh hưởng đáng kể đến trải nghiệm khách hàng. (2) Với word-overlap heuristics, 0.05 tương đương khoảng 1-2 từ quan trọng bị miss trên mỗi câu trả lời. Tuy nhiên, có thể cần threshold khác nhau cho từng metric: Faithfulness nên strict hơn (0.03) vì hallucination trong customer support có thể gây hậu quả pháp lý; Completeness có thể relax hơn (0.07) vì minor missing details ít critical hơn.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block deployment:** Faithfulness < 0.7 (hallucination risk trong customer support unacceptable), bất kỳ adversarial test case nào fail (security/safety critical), overall pass rate drop > 10% so với baseline.
> - **Chỉ alert:** Completeness drop < 0.05 (minor incompleteness), Relevance on individual cases < 0.5 (may need prompt tuning), Context Precision variations (retrieval ranking changes).

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit Tests & Lint] → [Benchmark Eval on Golden Dataset] → [Regression Check vs Baseline] → Deploy
```

> *Giải thích:* Stage 1 (Unit Tests & Lint): Đảm bảo code syntax và logic đúng, không break existing functions. Stage 2 (Benchmark Eval): Chạy full benchmark trên golden dataset 20 QA, compute tất cả 5 metrics. Stage 3 (Regression Check): So sánh metrics mới với baseline, block deploy nếu bất kỳ metric nào regress > threshold. Chỉ khi cả 3 stages pass thì mới cho deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Implement multi-hop retrieval cho cross-document questions | Context Recall +0.15, Completeness +0.20 | Fix 4/8 failures cùng lúc, overall pass rate tăng ~20% |
| 2 | Add faithfulness guardrails (post-generation fact-checking) | Faithfulness +0.15, hallucination count → 0 | Eliminate dangerous hallucinations, tăng trust |
| 3 | Enhance system prompt với adversarial handling | Adversarial pass rate 0% → 100% | All 3 adversarial cases pass, improve safety |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Multi-policy comparison question:** "Compare the return restocking fees between v1.0 and v2.0 for opened devices" — test cross-version reasoning.
> 2. **Time-sensitive warranty + repair chain:** "My 22-month-old NovaBook has a defect, I have OrbitPlus, and I want a loaner during repair. What are my options?" — test multi-document chaining (warranty → repair → membership).
> 3. **Subtle misinformation trap:** "I was told by support that my gift card refund will be in cash. When will I receive it?" — test ability to correct false claims referencing policy (gift-card portions return to replacement gift card, not cash).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Tôi dự đoán Faithfulness sẽ là metric thấp nhất (vì lo ngại hallucination), nhưng thực tế **Completeness** mới là metric yếu nhất (0.55 vs 0.68). Điều này cho thấy vấn đề chính không phải generator bịa thông tin, mà là generator không có đủ context để sinh câu trả lời đầy đủ. Retrieval bottleneck quan trọng hơn generation quality trong hệ thống này. Ngoài ra, adversarial cases fail rate cao hơn dự đoán — system prompt cần được strengthen đáng kể cho production.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
>
> **Giới hạn của word-overlap:**
> 1. Không nhận biết semantic equivalence: "USD 49" vs "forty-nine dollars" sẽ mismatch hoàn toàn.
> 2. Không hiểu negation: "The warranty covers" vs "The warranty does NOT cover" có overlap cao nhưng ý nghĩa ngược nhau.
> 3. Không đánh giá được reasoning quality: multi-step logic, cause-effect relationships bị ignore.
> 4. Bias toward verbose answers: câu trả lời dài có nhiều tokens hơn → overlap tự nhiên cao hơn.
> 5. Language-dependent: stopwords list chỉ cover English, không scale cho multilingual.
>
> **Metrics bổ sung cho production:**
> 1. **LLM-based Faithfulness** (RAGAS / DeepEval): dùng LLM judge đánh giá từng claim trong answer có được support bởi context hay không.
> 2. **Semantic Similarity** (sentence embeddings cosine similarity): capture meaning thay vì exact word match.
> 3. **BERTScore**: token-level semantic matching sử dụng contextual embeddings.
> 4. **Human evaluation**: periodic human annotation trên sample để calibrate automated metrics.
> 5. **Business metrics**: customer satisfaction (CSAT), escalation rate, resolution time — link evaluation results với business outcomes.
