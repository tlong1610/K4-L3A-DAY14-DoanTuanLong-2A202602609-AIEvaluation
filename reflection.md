# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 50.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.845 | 0.520 | 1.000 | Tương đối ổn định, hầu hết retriever tìm đủ thông tin. |
| Context Precision | 0.934 | 0.700 | 1.000 | Retriever rất chính xác khi đưa ra evidence đúng ở ngay top chunks. |
| Faithfulness | 0.777 | 0.281 | 1.000 | Generative model phần lớn follow context, nhưng vẫn có hallucination nghiêm trọng (H02). |
| Relevance | 0.450 | 0.091 | 0.692 | Rất thấp. Heuristic đo relevance (overlap token) bị hạn chế khi câu hỏi ngắn nhưng answer dài. Dẫn tới đánh dấu sai nhiều câu (off_topic). |
| Completeness | 0.763 | 0.286 | 1.000 | Khá tốt. Hầu hết các câu model trả lời bao phủ được expected answer. |
| Overall Score | 0.686 | 0.319 | 0.872 | Dưới mong đợi (pass rate thấp) chủ yếu do bị kéo xuống bởi điểm Relevance. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 7 cases (chủ yếu là Easy và Medium có câu hỏi/trả lời chứa nhiều token trùng nhau).
- Metrics/cases ở mức Needs Work (0.6–0.8): 8 cases (như M01, H03, H05, bị điểm relevance kéo thấp xuống nhưng info vẫn đầy đủ).
- Metrics/cases ở mức Significant Issues (<0.6): 5 cases (như H04, A03, M03 bị lỗi thật sự: hallucination, incomplete).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5.0% |
| irrelevant | 4 | 20.0% |
| incomplete | 1 | 5.0% |
| off_topic | 7 | 35.0% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:* Dựa trên dữ liệu đánh giá, vấn đề chính nằm ở **Evaluation heuristic** và quá trình **Generation**. Hệ thống retrieval hoạt động tương đối tốt với Context Precision đạt 1.0 và Recall trung bình > 0.75. Tuy nhiên, điểm Relevance trung bình chỉ ở mức 0.450 dẫn đến nhiều câu bị phân loại là off_topic (35%). Điều này xuất phát từ hạn chế của token overlap metric khi không đánh giá được sự tương đồng về mặt ngữ nghĩa (semantic relevance). Ngoài ra, phần generation vẫn còn tồn tại các lỗi nghiêm trọng, cụ thể là 1 trường hợp hallucination và 1 trường hợp incomplete.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:** H04 - I activated OrbitPlus after placing my order. Can the free shipping benefit be applied retroactively to that order?
**Expected answer:** No. The OrbitPlus membership benefit must be active when the order is placed. Activating OrbitPlus after placing an order does not retroactively change the price or shipping fee.
**Actual answer:** OrbitTech sells many devices.
**Scores:** Context Recall: 0.895 | Context Precision: 1.000 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000
**Evidence inspection:** Retriever lấy đúng chunks nhưng LLM trả lời hoàn toàn bịa đặt và không liên quan. Mọi điểm đều 0.

| Level | Question | Answer |
|---|---|---|
| Symptom | Câu trả lời không chứa bất cứ thông tin nào liên quan đến OrbitPlus hay shipping fee. |
| Why 1 | LLM sinh ra text ngẫu nhiên ("OrbitTech sells many devices"). |
| Why 2 | Có thể do nhiệt độ (temperature) của model cao hoặc prompt injection ngầm. |
| Why 3 | System prompt không strict buộc LLM phải dùng evidence khi sinh câu trả lời. |
| Why 4 | Thiếu validation sau generation để block trả lời off-topic hoàn toàn. |
| Why 5 | Prompt instruction yếu và lack of guardrails for answer grounding. |

**Root cause từ `find_root_cause()`:**
`Generation Error: Model generated ungrounded/hallucinated text (Faithfulness < 0.3).`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
Đồng ý với phân tích này. Mặc dù Context Recall đạt 0.895, cho thấy retriever đã trích xuất đủ tài liệu cần thiết, nhưng điểm Faithfulness lại bằng 0. Điều này chứng tỏ model đã bỏ qua thông tin trong context và tự tạo ra một câu trả lời không liên quan.

**Proposed fix cụ thể:**
Cần tinh chỉnh lại system prompt để kiểm soát output của model. Có thể bổ sung chỉ thị: "Chỉ được trả lời dựa trên tài liệu được cung cấp. Nếu thông tin không có trong tài liệu, hãy trả lời 'Tôi không biết'." Ngoài ra, việc tích hợp thêm LLM guardrail sau bước generation sẽ giúp chặn các phản hồi hallucination trước khi gửi đến người dùng.

### Failure 2

**ID và question:** A03 - My NovaBook 14 is covered by OrbitTech lifetime warranty, right? I need to claim it now.
**Expected answer:** The NovaBook 14 does not have a lifetime warranty. OrbitTech provides a 24-month limited hardware warranty...
**Actual answer:** The NovaBook 14 has a lifetime warranty and will be replaced immediately.
**Scores:** Context Recall: 0.469 | Context Precision: 1.000 | Faithfulness: 0.375 | Relevance: 0.333 | Completeness: 0.125 | Overall: 0.278
**Evidence inspection:** Context Recall thấp (0.469), retriever lấy chunk bị thiếu thông tin quan trọng. Model dính bẫy adversarial nên xác nhận "lifetime warranty" sai sự thật.

| Level | Question | Answer |
|---|---|---|
| Symptom | Answer xác nhận sai sự thật "lifetime warranty" từ user. |
| Why 1 | Model có xu hướng làm hài lòng user (sycophancy) khi bị hỏi câu gài bẫy. |
| Why 2 | Thiếu guardrail chống "false premise" trong system prompt. |
| Why 3 | Retrieval cũng không cung cấp đủ context để phủ định (Context Recall = 0.469). |
| Why 4 | Document chỉ nói "24-month" mà không có chữ "lifetime" nên BM25 có thể xếp hạng thấp. |
| Why 5 | Retriever yếu (BM25) khi query có từ trái nghĩa với tài liệu + Prompt không dặn dò việc từ chối assumption sai. |

**Root cause và proposed fix:**
Root cause: Retrieval thiếu khả năng tìm kiếm ngữ nghĩa để xử lý các câu hỏi phức tạp, kết hợp với hiện tượng sycophancy (xu hướng đồng tình với người dùng) của model khi gặp câu hỏi gài bẫy.
Fix: Nâng cấp retriever sang phương pháp Semantic Search để cải thiện độ chính xác ngữ nghĩa. Đồng thời, bổ sung yêu cầu ràng buộc vào prompt: "Cần kiểm chứng các giả định của người dùng bằng cách đối chiếu trực tiếp với chính sách, không tự động đồng tình với thông tin sai lệch".

**ID và question:** M03 - My package was delivered yesterday but accessories are missing. What should I do?
**Expected answer:** Visible shipping damage or missing items must be reported within 48 hours after confirmed delivery. You should keep the packaging and provide photographs of the label, box, and contents when contacting support.
**Actual answer:** Visible shipping damage must be reported within 48 hours.
**Scores:** Context Recall: 0.840 | Context Precision: 1.000 | Faithfulness: 1.000 | Relevance: 0.000 | Completeness: 0.320 | Overall: 0.440
**Evidence inspection:** Context cung cấp đủ (Recall 0.84). Câu trả lời chính xác nhưng quá vắn tắt, thiếu các bước "keep packaging", "provide photographs".

| Level | Question | Answer |
|---|---|---|
| Symptom | Câu trả lời quá ngắn, không hướng dẫn đủ các bước tiếp theo. |
| Why 1 | Model có xu hướng trả lời trực tiếp ý chính mà bỏ qua chi tiết (verbosity thấp). |
| Why 2 | Không có instruction "Hãy liệt kê toàn bộ các bước khách hàng cần làm theo policy". |
| Why 3 | System prompt chung chung không nhấn mạnh tính "Actionability". |
| Why 4 | Đánh giá trước đây chỉ tập trung vào độ chính xác (faithfulness), chưa đo completeness chặt chẽ. |
| Why 5 | Prompt thiếu instruction yêu cầu trích xuất toàn bộ điều kiện và quy trình. |

**Root cause và proposed fix:**
Root cause: Model instruction hiện tại chưa nhấn mạnh đủ về yêu cầu Completeness và Actionability trong câu trả lời.
Fix: Cập nhật system prompt để yêu cầu chi tiết hơn: "Khi tư vấn về quy trình, phải liệt kê đầy đủ các điều kiện, thời hạn và các loại giấy tờ hoặc bằng chứng cần thiết theo chính sách".

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Lỗi Generation Heuristics (Relevance đánh giá sai) | E01, M01, M02, M04, M06, H01, H02, A01 | High |
| 2 | Lỗi Hallucination & Prompt Injection | H04, A02, A03 | High |
| 3 | Lỗi Thiếu Completeness (Câu trả lời cụt lủn) | M03 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**
Ưu tiên hàng đầu sẽ là xử lý Cluster 2 (Lỗi Hallucination & Prompt Injection). Nguyên nhân là do các lỗi này tác động trực tiếp đến độ tin cậy và rủi ro an toàn (trust & safety) của hệ thống. Nếu tác tử cung cấp thông tin sai lệch về chính sách bảo hành, điều này có thể dẫn đến thiệt hại về tài chính và pháp lý. Ngược lại, các vấn đề ở Cluster 1 chủ yếu xuất phát từ giới hạn của các phương pháp đánh giá (heuristic evaluator) chứ không hẳn do chất lượng sinh văn bản thực tế của model.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Implement a faithfulness guardrail to filter answers not grounded in retrieved context | Open |
| F002 | irrelevant | Answer does not address the question — improve prompt clarity | Improve prompt clarity to ensure the agent directly addresses the user question | Open |
| F003 | off_topic | Answer does not address the question — improve prompt clarity | Strengthen intent detection to prevent off-topic responses | Open |
| F004 | off_topic | Answer does not address the question — improve prompt clarity | No suggestion available | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | No suggestion available | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | No suggestion available | Open |
| F007 | irrelevant | Answer does not address the question — improve prompt clarity | No suggestion available | Open |
| F008 | off_topic | Answer does not address the question — improve prompt clarity | No suggestion available | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | No suggestion available | Open |
| F010 | off_topic | Answer is missing key information — increase context window or improve generation | No suggestion available | Open |

**Ba improvement suggestions ưu tiên**
1. Thay thế Token Overlap heuristic bằng LLM-as-a-Judge nhằm đánh giá chính xác hơn về ngữ nghĩa cho metric Relevance.
2. Áp dụng System Guardrails thông qua các ràng buộc prompt để model có thể nhận diện và từ chối các yêu cầu vi phạm an toàn hoặc chứa thông tin sai lệch (Prompt Injection).
3. Tích hợp Vector Search và tối ưu hóa chiến lược chunking để thay thế hoặc kết hợp với BM25, qua đó xử lý tốt hơn các câu hỏi chứa từ đồng nghĩa hoặc được diễn đạt lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Dùng LLM Judge thay Token Overlap | Relevance | Chạy lại pipeline, theo dõi % pass rate và điểm Relevance trung bình xem có sát với human rating không. |
| System Guardrail cho False Premises | Faithfulness (trong Adversarial) | Test lại với tập dataset chuyên về Adversarial, đếm số lần "refusal" thành công. |
| Sửa System Prompt yêu cầu Completeness | Completeness | Đo điểm Completeness trung bình trên các câu Medium/Hard sau khi đổi prompt. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**
Quá trình này nên được tự động hóa trong CI/CD pipeline mỗi khi có Pull Request thay đổi mã nguồn, đặc biệt là các thay đổi liên quan đến thuật toán chunking, logic của retriever, system prompt, hoặc khi cập nhật phiên bản model mới.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**
Mức giảm 0.05 là phù hợp để theo dõi chất lượng tổng thể, vì nó đủ nhạy để phát hiện sự suy giảm hiệu suất mà không tạo ra quá nhiều cảnh báo sai (false alarms). Tuy nhiên, đối với Faithfulness, cần thiết lập ngưỡng nghiêm ngặt hơn (ví dụ: 0.02) do yêu cầu không khoan nhượng đối với hiện tượng hallucination trong lĩnh vực hỗ trợ khách hàng.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**
- **Block deployment:** Faithfulness (nhằm ngăn chặn hallucination) và Context Recall (nếu thiếu tài liệu gốc, rủi ro sinh sai thông tin sẽ rất cao).
- **Chỉ alert:** Relevance (các heuristics thường đánh giá thấp các câu trả lời dài nhưng mang tính cung cấp thêm ngữ cảnh) và Context Precision (kết quả hiển thị không ở vị trí tối ưu nhưng vẫn chứa đủ thông tin).

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Eval (Golden Dataset)] → [Regression Testing] → [Human Review (nếu failed)] → Deploy
```

> *Giải thích:* Bước Offline evaluation sử dụng `run_full_eval` để tính toán các metrics. Tiếp theo, `run_regression` đối chiếu kết quả này với baseline. Nếu phát hiện hiệu suất suy giảm vượt ngưỡng (drop > 0.05), hệ thống sẽ chặn quá trình deploy và yêu cầu Human Review để phân tích root cause thông qua `FailureAnalyzer`.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thêm LLM-as-a-Judge | Relevance | Điểm Relevance sẽ tăng (phản ánh đúng semantic), giảm false negatives. |
| 2 | Cải thiện System Prompt Guardrails | Faithfulness | Chặn được các prompt injection, nâng điểm Faithfulness trên Adversarial. |
| 3 | Tối ưu retrieval (Reranking) | Context Precision | Chunk quan trọng nhất sẽ lên rank #1, tiết kiệm LLM context window. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**
Cần bổ sung các truy vấn sử dụng từ đồng nghĩa hoặc cách diễn đạt khác không trùng khớp trực tiếp với tài liệu gốc (ví dụ: dùng "cost" thay vì "price", hoặc "broken" thay vì "defects"). Ngoài ra, nên thêm các câu hỏi phức tạp đòi hỏi tổng hợp thông tin từ 3 đến 4 tài liệu riêng biệt để kiểm tra khả năng reasoning.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**
Kết quả đo Context Precision của thuật toán BM25 đạt 1.0 là một điểm khá bất ngờ. Điều này có thể được lý giải bởi tập dataset hiện tại sử dụng hệ thống từ vựng rất sát với tài liệu gốc, tạo điều kiện thuận lợi cho phương pháp keyword matching. Ngược lại, phương pháp Heuristic Evaluation dựa trên word-overlap hoạt động kém hiệu quả hơn dự kiến, dẫn đến điểm Relevance trung bình chỉ đạt khoảng 0.375.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**
Hạn chế lớn nhất của word-overlap heuristics là không có khả năng hiểu ngữ nghĩa, từ đồng nghĩa hoặc các cấu trúc câu paraphrase. Một câu trả lời ngắn gọn, đầy đủ ý nhưng dùng từ vựng khác với câu hỏi vẫn bị chấm điểm thấp. Khi đưa hệ thống vào production, cần thay thế phương pháp này bằng `LLM-as-a-Judge` (ví dụ như các metrics của RAGAS) để đánh giá Faithfulness và Answer Relevance một cách chính xác hơn. Phương án khác là sử dụng các mô hình như Sentence Transformer hoặc Cross Encoder để đo lường độ tương đồng ngữ nghĩa.
