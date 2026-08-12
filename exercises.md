# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Answer paraphrases retrieved context loosely (e.g. rounds "14 days" to "about two weeks") but every claim is still traceable to the context. Score dips to ~0.7 without inventing facts. | Answer states a warranty period, refund amount, or fee that does not appear in any retrieved chunk (e.g. assistant invents a "30-day extended warranty" OrbitTech never mentions). Score below 0.6 with fabricated specifics is a hallucination, not a phrasing issue. | Below 0.6: block release, inspect the generation prompt for missing grounding instructions and check whether the offending chunk was even retrieved. Above 0.6 but below 0.8: log for the next prompt-tuning pass. |
| Answer Relevance | Answer correctly addresses the core question but adds one extra, tangential sentence (e.g. answering a shipping-time question and also restating the return policy). Score ~0.65–0.75. | Answer responds to a different intent than what was asked (e.g. user asks about warranty claim steps, assistant explains how to place an order). Score below 0.6 signals intent/routing failure, not verbosity. | Below 0.6: treat as `irrelevant` failure, inspect retrieval query formation or prompt intent handling. 0.6–0.8: monitor, no release block. |
| Context Recall | Retriever misses one minor supporting clause (e.g. an exact fee amount) while the main policy is retrieved, so the answer is still usable but slightly under-cited. Score ~0.7. | Retriever fails to surface the document that contains the actual rule (e.g. a hard case about escalation timelines retrieves only the FAQ doc, not `09_escalation_and_policy_updates.md`). Score below 0.6 means the answer cannot possibly be complete regardless of generation quality. | Below 0.6: investigate retriever (BM25 query terms, chunk granularity, top-k) before touching the prompt — generation cannot fix missing evidence. |
| Context Precision | Retrieved chunks include one adjacent, mildly relevant paragraph ranked lower (e.g. general shipping info next to the specific delay policy). Score ~0.7–0.8, minor noise. | Highly relevant chunk is retrieved but ranked last among 5, or irrelevant chunks (different product category) dominate top ranks. Score below 0.6 means the generator is likely to anchor on noise instead of the right evidence. | Below 0.6: tune BM25 ranking/re-ranking or top-k; check for near-duplicate chunks diluting scores. |
| Completeness | Answer covers the main rule but omits a rarely-triggered exception (e.g. return policy stated correctly but the "final sale" exception for clearance items is left out). Score ~0.7. | Answer omits a condition that changes the outcome for the user's specific case (e.g. missing the "must have proof of purchase" requirement on a warranty claim). Score below 0.6 means the user could act on wrong/incomplete information. | Below 0.6: classify as `incomplete`, check both retrieval (was the exception's chunk retrieved?) and generation (was it in context but dropped?). |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy một tập câu hỏi OrbitTech (vd 10 câu) và hai answer candidates cho mỗi câu — một answer đúng/grounded (A) và một answer sai hoặc kém hơn rõ rệt (B), do con người gán nhãn trước "ground truth: A tốt hơn B". Chạy judge hai lần trên cùng cặp câu trả lời:
> - **Condition 1 (A trước, B sau):** prompt liệt kê "Answer 1 = A, Answer 2 = B".
> - **Condition 2 (đảo vị trí):** cùng nội dung nhưng "Answer 1 = B, Answer 2 = A".
>
> Nếu judge không có position bias, tỷ lệ chọn A thắng phải giống nhau ở cả hai condition (trong sai số thống kê). Nếu tỷ lệ "Answer 1 thắng" cao bất thường ở cả hai lần chạy bất kể nội dung, đó là bằng chứng trực tiếp của position bias. Có thể mở rộng thêm condition thứ ba là single-answer scoring độc lập (không so sánh cặp) để xem thứ hạng có đổi so với pairwise không.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric phải mô tả tiêu chí theo **nội dung/độ chính xác**, không theo độ dài, và nêu rõ "an answer that is concise and correct should score the same as or higher than a longer answer with the same correct content — extra length without new, verifiable information does not raise the score." Có thể thêm penalty rõ ràng cho câu trả lời dài dòng chứa filler, lặp ý hoặc thông tin không liên quan đến câu hỏi ("verbose but low-information answers should be penalized under Relevance/Conciseness"). Ngoài ra, cho judge xem cả hai answer ở dạng đã chuẩn hoá độ dài tương đối, hoặc thêm một dimension "Conciseness" riêng để tách bạch khỏi "Correctness", tránh việc độ dài âm thầm kéo điểm correctness lên.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM judge có thể tự tin (confident) nhưng sai theo cách hệ thống — ví dụ đánh giá cao câu trả lời "nghe chuyên nghiệp" dù thiếu evidence, hoặc favor câu trả lời giống văn phong của chính model đó (self-preference bias). Nếu không so sánh với nhãn con người, ta không biết judge có đang đo đúng "câu trả lời tốt cho khách hàng OrbitTech" hay chỉ đo "câu trả lời hợp gu của judge". Calibration (chạy judge trên một tập nhỏ đã có nhãn người, đo agreement/correlation) giúp: (1) phát hiện systematic bias trước khi tin tưởng judge trên toàn bộ dataset, (2) hiệu chỉnh threshold pass/fail cho đúng ngưỡng con người chấp nhận, (3) tạo confidence để dùng judge score làm CI/CD gate thay vì chỉ dùng cho báo cáo tham khảo.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.75 | Đây là metric rủi ro cao nhất về pháp lý/tin cậy — trả lời sai policy/số tiền/thời hạn có thể gây khiếu nại thật với khách hàng. Ngưỡng đặt trên mốc 0.6 "significant issues" của bài giảng để chặn sớm trước khi hallucination lọt ra production. |
| Answer Relevance | 0.7 | Trả lời lạc đề làm khách hàng phải hỏi lại, tăng chi phí support, nhưng ít rủi ro pháp lý hơn faithfulness nên ngưỡng thấp hơn một chút, vẫn nằm trong vùng "needs work" nếu thấp hơn. |
| Completeness | 0.7 | Thiếu điều kiện/ngoại lệ quan trọng (vd thiếu yêu cầu "proof of purchase" khi trả lời về bảo hành) khiến khách hàng hành động sai dựa trên thông tin thiếu, nên cần ngưỡng chặt tương đương relevance. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation** (RAGAS-style trên golden dataset, chạy trong CI): dùng mỗi khi có thay đổi prompt, retriever, chunking hoặc model — trước khi merge/deploy. Đây là gate bắt buộc vì nó lặp lại được, so sánh được giữa các version, và chạy tự động không cần người trực chờ.
> - **Online evaluation** (theo dõi real traffic, vd qua TruLens/Langfuse feedback functions): dùng sau khi đã deploy, để bắt các failure mode không xuất hiện trong golden dataset tĩnh — câu hỏi thật của khách hàng đa dạng hơn 20 test case, và policy/corpus có thể đổi theo thời gian (như `09_escalation_and_policy_updates.md` gợi ý). Chạy liên tục, thường lấy mẫu một phần traffic để giữ chi phí thấp.
> - **Human review**: dùng cho các case high-stakes hoặc cần calibration — ví dụ case liên quan đến an toàn dữ liệu/privacy (`08_accounts_privacy_and_security.md`), các adversarial case (prompt injection, false premise), hoặc định kỳ lấy mẫu để calibrate LLM judge (trả lời cho Exercise 1.2 câu 3). Human review chậm và tốn kém nên không dùng làm gate tự động cho mọi lần đổi code, mà dùng làm nguồn ground-truth để hiệu chỉnh offline/online evaluation.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
