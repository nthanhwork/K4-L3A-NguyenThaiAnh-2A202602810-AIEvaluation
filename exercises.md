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

| Metric            | Acceptable Low Score Scenario                                                                                                                | Critical Low Score Scenario                                                                                                                               | Action Required                                                                                                                                |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Faithfulness      | Khi câu hỏi mang tính xã giao/mở hoặc ngoài context và trợ lý từ chối lịch sự, hoặc câu trả lời bổ sung tri thức chung không sai lệch.       | Khi trợ lý bịa đặt (hallucination) chính sách bảo hành, giá bán hoặc thông số kỹ thuật không hề có trong tài liệu của OrbitTech Store.                    | Siết chặt system prompt yêu cầu chỉ dùng thông tin trong context, hạ temperature xuống 0.0, áp dụng prompt cảnh báo phạt lỗi bịa đặt.          |
| Answer Relevance  | Khi câu hỏi của người dùng mơ hồ/thiếu dữ kiện, trợ lý chủ động hỏi lại để làm rõ (clarification question) thay vì trả lời trực tiếp.        | Trợ lý trả lời lạc đề hoàn toàn, lặp lại câu hỏi hoặc trả lời thông tin của sản phẩm khác không liên quan đến thắc mắc của khách hàng.                    | Cải tiến query rewriting, phân loại rõ intent của người dùng trước khi sinh câu trả lời, tinh chỉnh prompt tập trung vào trọng tâm câu hỏi.    |
| Context Recall    | Khi câu hỏi đơn giản/cục bộ, retriever chỉ cần trích xuất 1 chunk trọng tâm là đủ dữ liệu trả lời, không cần kéo toàn bộ tài liệu liên quan. | Retriever bỏ sót các điều khoản loại trừ cốt lõi (ví dụ: điều kiện không được đổi trả tai nghe đã bóc seal), khiến trợ lý trả lời thiếu sót nghiêm trọng. | Tăng Top-K retrieval, bổ sung Hybrid Search (kết hợp Dense Embedding với Sparse BM25), điều chỉnh kích thước chunk size và chunk overlap.      |
| Context Precision | Khi kho tri thức có nhiều tài liệu tương tự nhau, một vài chunk phụ bị kéo theo ở cuối danh sách Top-K nhưng các chunk đầu vẫn chính xác.    | Các chunk rác hoặc tài liệu không liên quan bị xếp lên vị trí đầu (rank 1, 2), đẩy các chunk chứa câu trả lời đúng xuống vị trí thấp hoặc bị cắt bỏ.      | Áp dụng mô hình Reranking (Cross-Encoder / Cohere Rerank) để chấm điểm lại mức độ phù hợp sau khi retrieval, tối ưu hàm tính similarity score. |
| Completeness      | Khi người dùng chỉ hỏi một chi tiết nhỏ hoặc câu hỏi Yes/No đơn giản, không cần liệt kê toàn bộ quy trình dài dòng.                          | Trả lời thiếu các bước hướng dẫn bắt buộc trong quy trình khiếu nại/bảo hành, khiến khách hàng không biết phải làm gì tiếp theo.                          | Bổ sung few-shot examples trong prompt về tiêu chuẩn đầy đủ của câu trả lời, yêu cầu trợ lý kiểm tra checklist các ý cần có trước khi trả lời. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

- **Condition 1 (Original Order):** Đưa 2 câu trả lời A và B vào prompt đánh giá so sánh (pairwise) theo thứ tự: [Candidate 1: Answer A, Candidate 2: Answer B] và yêu cầu Judge chấm điểm hoặc chọn câu tốt hơn.
- **Condition 2 (Swapped Order):** Hoán đổi vị trí của hai câu trả lời: [Candidate 1: Answer B, Candidate 2: Answer A] với cùng một câu hỏi và rubric đánh giá.
- **Đánh giá bias:** Thống kê tỷ lệ phần trăm Judge chọn Candidate 1 ở cả hai lượt. Nếu tỷ lệ chọn vị trí 1 chênh lệch vượt trội (> 60%) bất kể nội dung là A hay B, kết luận Judge có Position Bias. Giải pháp là chạy song song cả hai lượt và lấy trung bình kết quả (swap-evaluation).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

- Trong rubric chấm điểm, thiết lập tiêu chí riêng biệt về "Độ súc tích & Mật độ thông tin" (Conciseness & Information Density).
- Định nghĩa rõ trong thang điểm: "Một câu trả lời dài dòng, lặp từ, chêm câu sáo rỗng hoặc thông tin thừa thãi không mang lại giá trị sẽ bị trừ điểm (tối đa chỉ đạt 3/5 điểm)".
- Hướng dẫn Judge chỉ tính điểm dựa trên số lượng luận điểm/sự thật chính xác (factual points), không cho thêm điểm chỉ vì câu trả lời dài hoặc có cấu trúc phức tạp.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

- LLM Judge không có nhận thức thực sự và dễ mắc các thiên kiến hệ thống (như tự chấm điểm cao cho model cùng họ - self-preference bias, hoặc chấm quá dễ dãi - leniency bias).
- Việc hiệu chỉnh (calibrate) với nhãn do con người gán (Human Ground Truth) giúp đo lường mức độ tương quan (Cohen's Kappa hoặc Spearman rank correlation), từ đó tìm ra ngưỡng tin cậy (confidence threshold), phát hiện sự chênh lệch (gap) và điều chỉnh lại prompt/rubric của Judge để phán đoán tiệm cận với tiêu chuẩn của chuyên gia con người.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric           | Threshold | Lý do                                                                                                                                                        |
| ---------------- | --------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Faithfulness     |   >= 0.85 | Đây là tiêu chí an toàn cốt lõi; trợ lý không được phép bịa đặt chính sách bảo hành, giá cả gây rủi ro pháp lý hoặc khiếu nại từ khách hàng.                 |
| Answer Relevance |   >= 0.80 | Đảm bảo câu trả lời luôn đi thẳng vào vấn đề khách hàng đang thắc mắc, không trả lời vòng vo hoặc lạc đề gây lãng phí thời gian.                             |
| Completeness     |   >= 0.75 | Đảm bảo cung cấp đầy đủ các bước hướng dẫn hành động thiết yếu; ngưỡng 0.75 cho phép linh hoạt câu từ súc tích nhưng không được bỏ sót điều kiện quan trọng. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

- **Offline Evaluation:** Dùng trong giai đoạn phát triển (development) và trong CI/CD pipeline trước khi deploy phiên bản mới. Chạy tự động trên bộ Golden Dataset cố định để kiểm tra lỗi hồi quy (regression testing) và đảm bảo chất lượng baseline mà không tốn chi phí rủi ro với người dùng thật.
- **Online Evaluation:** Dùng khi hệ thống đã đưa lên môi trường Production phục vụ người dùng thật. Đánh giá liên tục thông qua telemetry, phản hồi người dùng (thumbs up/down, CSAT), tỷ lệ chuyển giao cho nhân viên (escalation rate), latency và lấy mẫu ngẫu nhiên cho LLM Judge đánh giá theo thời gian thực.
- **Human Review:** Dùng định kỳ (hàng tuần/tháng) hoặc kích hoạt khi hệ thống cảnh báo các ca đánh giá có độ tự tin thấp (low confidence), các ca khách hàng khiếu nại gay gắt hoặc khi chuẩn bị ra mắt tính năng/chính sách mới. Chuyên gia thẩm định giúp cập nhật rubric, hiệu chỉnh LLM Judge và mở rộng thêm các ca khó vào Golden Dataset.

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

| Hạng mục                      | Kết quả       |
| ----------------------------- | ------------- |
| Tổng số records               | \_\_\_\_ / 20 |
| Easy                          | \_\_\_\_ / 5  |
| Medium                        | \_\_\_\_ / 7  |
| Hard                          | \_\_\_\_ / 5  |
| Adversarial                   | \_\_\_\_ / 3  |
| Source documents được sử dụng | \_\_\_\_ / 10 |
| Validator status              | PASS / FAIL   |

**Ba case đại diện cho quyết định thiết kế**

| ID  | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
| --- | ---------- | ------------------ | ----------------------------------------------- |
|     |            |                    |                                                 |
|     |            |                    |                                                 |
|     |            |                    |                                                 |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> _Câu trả lời:_

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

| ID  | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
| --- | ---------------- | ---------: | ------------: | -----------: | --------: | -----------: | ------: | ------- | ------------ |
| E01 |                  |            |               |              |           |              |         |         |              |
| E02 |                  |            |               |              |           |              |         |         |              |
| E03 |                  |            |               |              |           |              |         |         |              |
| E04 |                  |            |               |              |           |              |         |         |              |
| E05 |                  |            |               |              |           |              |         |         |              |
| M01 |                  |            |               |              |           |              |         |         |              |
| M02 |                  |            |               |              |           |              |         |         |              |
| M03 |                  |            |               |              |           |              |         |         |              |
| M04 |                  |            |               |              |           |              |         |         |              |
| M05 |                  |            |               |              |           |              |         |         |              |
| M06 |                  |            |               |              |           |              |         |         |              |
| M07 |                  |            |               |              |           |              |         |         |              |
| H01 |                  |            |               |              |           |              |         |         |              |
| H02 |                  |            |               |              |           |              |         |         |              |
| H03 |                  |            |               |              |           |              |         |         |              |
| H04 |                  |            |               |              |           |              |         |         |              |
| H05 |                  |            |               |              |           |              |         |         |              |
| A01 |                  |            |               |              |           |              |         |         |              |
| A02 |                  |            |               |              |           |              |         |         |              |
| A03 |                  |            |               |              |           |              |         |         |              |

**Aggregate Report**

- Overall pass rate: \_\_\_\_%
- Avg Context Recall: \_\_\_\_
- Avg Context Precision: \_\_\_\_
- Avg Faithfulness: \_\_\_\_
- Avg Relevance: \_\_\_\_
- Avg Completeness: \_\_\_\_
- Failure type distribution: \_\_\_\_

**Ba cases có Overall Score thấp nhất**

1. ID: \_**\_ | Score: \_\_** | Failure type: \_\_\_\_
2. ID: \_**\_ | Score: \_\_** | Failure type: \_\_\_\_
3. ID: \_**\_ | Score: \_\_** | Failure type: \_\_\_\_

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> _Câu trả lời:_

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
- [ ] Dimension khác: \***\*\_\_\*\***

| Score | Tiêu chí domain-specific | Ví dụ response |
| ----: | ------------------------ | -------------- |
|     5 |                          |                |
|     4 |                          |                |
|     3 |                          |                |
|     2 |                          |                |
|     1 |                          |                |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
| --------- | ----------------- | --------------------- |
|           |                   |                       |
|           |                   |                       |
|           |                   |                       |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> _Câu trả lời:_

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí                  | Framework 1: \_\_\_\_ | Framework 2: \_\_\_\_ |
| ------------------------- | --------------------- | --------------------- |
| Setup complexity          |                       |                       |
| Metrics available         |                       |                       |
| CI/CD integration         |                       |                       |
| Kết quả trên cùng dataset |                       |                       |
| Insight rút ra            |                       |                       |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> _Phân tích:_

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID      | Recall before | Recall after | Precision before | Precision after | Delta Precision |
| ------- | ------------: | -----------: | ---------------: | --------------: | --------------: |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
| **Avg** |               |              |                  |                 |                 |

**Tại sao Recall dự kiến không đổi?**

> _Câu trả lời:_

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> _Câu trả lời:_

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
