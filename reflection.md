# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Báo cáo này dùng cùng một lần chạy trong `artifacts/actual_answers.json` và
`artifacts/benchmark_results.json`. Kết luận được đối chiếu với answer, gold
evidence và retrieved chunks; các suy luận chưa có telemetry được ghi là giả
thuyết cần kiểm tra.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 30.0% (6/20 cases)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.883 | 0.333 | 1.000 | Coverage nhìn chung cao; A01 thấp nhất và M05 chỉ đạt 0.522. |
| Context Precision | 0.950 | 0.700 | 1.000 | Evidence liên quan thường đứng sớm; đây không phải nút thắt chính. |
| Faithfulness | 0.627 | 0.000 | 1.000 | M05 bằng 0 do output méo không overlap với evidence. |
| Relevance | 0.476 | 0.000 | 0.882 | Answer metric yếu nhất; nhiều câu trả lời không phủ từ khóa câu hỏi. |
| Completeness | 0.615 | 0.000 | 1.000 | Output bị cắt làm mất điều kiện và bước hành động. |
| Overall Score | 0.573 | 0.000 | 0.848 | Chỉ E05 đạt vùng Good; 8 cases dưới 0.6. |

**Score interpretation (theo Overall Score)**

- Good (0.8–1.0): 1 case — E05.
- Needs Work (0.6–0.8): 11 cases.
- Significant Issues (<0.6): 8 cases.

**Failure type distribution**

Tỷ lệ dưới đây dùng mẫu số 20 cases; 6 cases passed không có failure type.

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5% |
| irrelevant | 3 | 15% |
| incomplete | 2 | 10% |
| off_topic | 8 | 40% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:**

> Vấn đề chính nằm ở generation/output integrity hơn là retrieval. Context
> Recall trung bình 0.883 và Context Precision 0.950, nhưng Relevance chỉ 0.476
> và Completeness 0.615. H02 và M02 có Context Recall/Precision gần hoặc bằng
> 1.0 nhưng actual answer bị cắt, chứng minh evidence đúng đã đến model mà output
> vẫn thiếu. Retrieval vẫn cần cải thiện ở các ngoại lệ như A01 (recall 0.333)
> và M05 (recall 0.522), nhưng không giải thích được phần lớn khoảng cách giữa
> retrieval metrics và answer metrics. Nhãn `refusal` không được core sinh ra;
> không quan sát thấy case nào cần tự gắn nhãn này.

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1 — M05

**ID và question:**

> M05 — “My device has a covered defect after its return window ended. What
> process applies?”

**Expected answer:**

> After the return window, a covered defect follows the repair process.
> OrbitTech may repair, replace with an equivalent new or refurbished unit, or
> refund when the first two remedies are not reasonable, after diagnosis.

**Actual answer:**

> `_returns_and_ex`

**Scores:** Context Recall: 0.522 | Context Precision: 1.000 | Faithfulness:
0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000 | Passed: No |
Failure type: hallucination

**Evidence inspection:**

> Retriever xếp `06_warranty_policy.md/OT-06-P05` đầu tiên; chunk này nói đúng
> rằng covered defect sau return window đi theo repair process. Nó cũng lấy
> `05_returns_and_exchanges.md/OT-05-P05`, xác nhận warranty service sau return
> window theo warranty/repair documents. Tuy nhiên đoạn gold thứ hai mô tả ba
> remedy (repair/replacement/refund) không nằm trong top-5, nên recall chỉ 0.522.
> Actual answer là một fragment không tạo thành câu và không có claim chính sách
> có thể kiểm chứng. Nhãn hallucination là kết quả rule faithfulness < 0.3; về
> mặt hành vi, đây chính xác hơn là malformed/truncated generation.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Output chỉ là fragment `_returns_and_ex`; cả ba answer metrics bằng 0. |
| Why 1 | Tại sao symptom xảy ra? | Quan sát: generator trả một output không hoàn chỉnh dù top chunk chứa câu trả lời cốt lõi. |
| Why 2 | Tại sao generator trả fragment? | Giả thuyết: request kết thúc/truncate trong giai đoạn generation hoặc token reasoning đã chiếm budget; artifact hiện không lưu `finish_reason` để xác nhận. |
| Why 3 | Tại sao output lỗi vẫn được chấp nhận? | Code chỉ kiểm tra chuỗi có rỗng hay không; `_returns_and_ex` là non-empty nên được checkpoint. |
| Why 4 | Tại sao benchmark không sửa trước khi chấm? | Pipeline cố ý chấm actual output thật và chưa có output-integrity guard hoặc retry cho câu quá ngắn/malformed. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu validation/telemetry ở generation boundary: không lưu finish reason và không retry output không thành câu; retrieval cũng chưa bảo đảm lấy đoạn remedy thứ hai. |

**Root cause từ `find_root_cause()`:**

> `Multiple issues detected — review full pipeline`

**Bạn đồng ý hay không?**

> Đồng ý ở mức tổng quát vì cả ba score cùng thấp, nhưng trace cho phép ưu tiên
> cụ thể hơn: lỗi trực tiếp là output integrity, còn retrieval thiếu đoạn remedy
> là yếu tố phụ. Top chunk đã chứa quy trình đúng nên không thể kết luận đơn giản
> rằng context hoàn toàn thiếu hoặc sai.

**Proposed fix cụ thể:**

> Lưu `finish_reason`, output token count và model theo từng request; retry khi
> output quá ngắn, kết thúc giữa câu hoặc chỉ chứa identifier fragment. Tăng
> output budget khi model dùng reasoning. Đồng thời thêm query expansion cho
> “covered defect after return window” → “warranty remedy repair replacement
> refund” và regression case yêu cầu đủ process cùng remedy.

### Failure 2 — H02

**ID và question:**

> H02 — “My device order is worth USD 1,200. After one failed delivery, can the
> carrier leave it unattended or can I collect it?”

**Expected answer:**

> The package requires an adult signature, so OrbitTech does not authorize it
> to be left unattended. After the first failed delivery attempt, the customer
> may request carrier pickup, but the carrier may require identification
> matching the shipment name.

**Actual answer:**

> `No, the carrier cannot leave the package unattended because`

**Scores:** Context Recall: 0.926 | Context Precision: 1.000 | Faithfulness:
0.571 | Relevance: 0.176 | Completeness: 0.111 | Overall: 0.286 | Passed: No |
Failure type: irrelevant

**Evidence inspection:**

> `04_shipping_and_delivery.md/OT-04-P02` đứng rank 1 và gần như trùng toàn bộ
> gold evidence: ngưỡng USD 1,000, adult signature, pickup sau lần giao thất bại
> đầu tiên và yêu cầu ID. Recall 0.926 và precision 1.000 cho thấy retrieval đủ.
> Actual answer đúng ý đầu nhưng bị cắt sau “because”, bỏ hoàn toàn pickup và ID.
> Không có claim ngoài nguồn; failure_type `irrelevant` xuất phát từ relevance
> < 0.3 chứ không phản ánh đầy đủ lỗi truncation.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời dừng giữa câu, chỉ trả lời phần “leave unattended” và bỏ phần “can I collect it?”. |
| Why 1 | Tại sao completeness thấp? | Output không bao gồm pickup sau failed attempt và identification requirement. |
| Why 2 | Tại sao thiếu khi evidence đã ở rank 1? | Quan sát cho thấy không phải thiếu context; giả thuyết là generation bị truncate/finish bất thường. |
| Why 3 | Tại sao model không bị buộc trả lời hai phần? | Prompt nói “answer every part” nhưng không yêu cầu cấu trúc/checklist cho câu hỏi nhiều vế. |
| Why 4 | Tại sao output giữa câu vẫn được lưu? | Validator chỉ kiểm tra non-empty, không kiểm tra dấu kết thúc, minimum content hoặc coverage từng vế. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu output-integrity guard và per-subquestion coverage check ở generation boundary. |

**Root cause từ `find_root_cause()`:**

> `Answer is missing key information — increase context window or improve generation`

**Đánh giá và proposed fix:**

> Đồng ý với “improve generation”, không đồng ý rằng cần tăng context window:
> chunk đúng đã đứng đầu và recall 0.926. Prompt nên yêu cầu trả lời từng vế bằng
> bullet/checklist; lưu finish reason và retry câu kết thúc bằng liên từ. Đo lại
> bằng Completeness, Relevance và một assertion chứa cả “adult signature” lẫn
> “carrier pickup/identification”.

### Failure 3 — M02

**ID và question:**

> M02 — “Can an OrbitPlus member return an opened device after 30 days?”

**Expected answer:**

> No. OrbitPlus extends only the unopened-device return window to 45 calendar
> days for eligible purchases made while membership is active. It does not
> extend the 14-day opened-device window.

**Actual answer:**

> `No. An OrbitPlus member cannot return`

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness:
0.400 | Relevance: 0.333 | Completeness: 0.130 | Overall: 0.288 | Passed: No |
Failure type: incomplete

**Evidence inspection:**

> Rank 1 (`03_promotions_and_membership.md/OT-03-P05`) nêu đúng 45 ngày chỉ cho
> unopened và không mở rộng window 14 ngày của opened devices. Rank 2
> (`05_returns_and_exchanges.md/OT-05-P01`) bổ sung 14 ngày và restocking fee.
> Recall và precision đều 1.000 nên retrieval không thiếu. Actual answer có kết
> luận “No” đúng hướng nhưng dừng giữa câu, không nêu 14-day/45-day distinction.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời chỉ có kết luận và fragment, completeness 0.130. |
| Why 1 | Tại sao thiếu key information? | Không giải thích OrbitPlus chỉ kéo dài unopened window và opened window vẫn là 14 ngày. |
| Why 2 | Tại sao thiếu dù retrieval hoàn hảo? | Evidence đầy đủ ở hai rank đầu; giả thuyết generation bị truncate, không phải retrieval failure. |
| Why 3 | Tại sao hệ thống không phát hiện câu chưa hoàn chỉnh? | Chỉ có empty-string guard, không có semantic/structural completeness check. |
| Why 4 | Tại sao pass/failure rule chỉ phát hiện sau khi lưu? | Guardrail hiện nằm ở evaluator hậu kiểm, không nằm trước checkpoint của system under evaluation. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu generation completion validation và retry dựa trên finish reason/coverage của policy conditions. |

**Root cause từ `find_root_cause()`:**

> `Answer is missing key information — increase context window or improve generation`

**Đánh giá và proposed fix:**

> Đồng ý với phần “improve generation”; tăng context window không có căn cứ vì
> cả hai retrieval metrics bằng 1.0. Thêm output validation, retry fragment, và
> prompt template yêu cầu “decision + applicable window + exception”. Xác minh
> bằng Completeness và regression assertion có `opened`, `14`, `unopened`, `45`.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Malformed/truncated generation được checkpoint vì chỉ kiểm tra non-empty | M05, H02, M02 | High |
| 2 | Lexical relevance heuristic phạt câu đúng ý nhưng không lặp từ khóa; cần semantic review song song | E01, E04, H03, A01 | Medium |
| 3 | Retrieval coverage chưa đủ cho toàn bộ claim/ngoại lệ hoặc chứa noise | M05, A01, M06 | Medium |

**Nếu chỉ được sửa một cluster:**

> Chọn Cluster 1. Nó trực tiếp tạo cả ba worst cases, trong đó hai case có
> retrieval gần hoàn hảo. Một output-integrity guard dùng chung có thể ngăn
> fragment được lưu, cải thiện Completeness/Relevance/Faithfulness và làm
> benchmark phản ánh chất lượng câu trả lời thực thay vì lỗi truyền/generation.

---

## 4. Improvement Log

Output nguyên bản từ `failure_analysis.improvement_log`:

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Add grounding guardrails that require each claim to be supported by retrieved evidence. | Open |
| F002 | off_topic | Answer is missing key information — increase context window or improve generation | Add intent-specific prompt examples and test ambiguous customer questions. | Open |
| F003 | off_topic | Context is missing or irrelevant — improve retrieval | Increase evidence coverage with improved chunking and a checklist for required answer details. | Open |
| F004 | off_topic | Context is missing or irrelevant — improve retrieval | Improve intent routing and reject retrieved chunks that do not match the customer request. | Open |
| F005 | off_topic | Answer does not address the question — improve prompt clarity | Investigate and prioritize a targeted fix | Open |
| F006 | incomplete | Answer is missing key information — increase context window or improve generation | Investigate and prioritize a targeted fix | Open |
| F007 | off_topic | Answer is missing key information — increase context window or improve generation | Investigate and prioritize a targeted fix | Open |
| F008 | hallucination | Multiple issues detected — review full pipeline | Investigate and prioritize a targeted fix | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | Investigate and prioritize a targeted fix | Open |
| F010 | off_topic | Answer does not address the question — improve prompt clarity | Investigate and prioritize a targeted fix | Open |
| F011 | irrelevant | Answer is missing key information — increase context window or improve generation | Investigate and prioritize a targeted fix | Open |
| F012 | irrelevant | Answer does not address the question — improve prompt clarity | Investigate and prioritize a targeted fix | Open |
| F013 | incomplete | Answer is missing key information — increase context window or improve generation | Investigate and prioritize a targeted fix | Open |
| F014 | irrelevant | Answer does not address the question — improve prompt clarity | Investigate and prioritize a targeted fix | Open |

Mapping để truy vết: F001=E01, F002=E02, F003=E03, F004=E04, F005=M01,
F006=M02, F007=M04, F008=M05, F009=M06, F010=H01, F011=H02,
F012=H03, F013=H04, F014=A01. Bảng tự động ghép suggestions theo vị trí,
nên một số suggestion không khớp hoàn toàn trace; ba ưu tiên dưới đây là bản đã
đối chiếu thủ công.

**Ba improvement suggestions ưu tiên**

1. Thêm output-integrity guard, finish-reason telemetry và retry cho fragment.
2. Dùng prompt/checklist bắt buộc trả lời từng vế và mọi điều kiện chính sách.
3. Thêm query expansion/reranking cho claim phụ và ngoại lệ policy.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Output-integrity guard + retry | Completeness, Relevance, malformed-output rate | Chạy lại M05/H02/M02; yêu cầu 0 malformed outputs, Completeness từng case ≥0.5 và không metric answer nào giảm >0.05. |
| Multi-part policy checklist | Completeness, pass rate | Regression assertions cho đủ các điều kiện bắt buộc; so sánh baseline trên cùng 20 QA bằng `run_regression()`. |
| Query expansion/reranking | Context Recall, Context Precision | Đo top-k trước/sau trên M05/A01/M06; giữ cùng gold evidence và yêu cầu recall tăng mà precision không giảm >0.05. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy trên mọi PR thay đổi prompt, retriever, chunking, generation model,
> output guard hoặc evaluation core; chạy lại trước release và theo lịch khi
> corpus/policy đổi. Với thay đổi evaluator, dùng cùng saved actual answers để
> cô lập metric logic. Với thay đổi RAG/model, sinh actual answers mới với model,
> temperature và top-k cố định rồi so với baseline đã version hóa. Dataset so
> sánh là 20 golden cases hiện tại; các candidate mới được review trước khi đưa
> vào phiên bản dataset kế tiếp.

**Câu 2: Threshold drop 0.05 có phù hợp không?**

> Phù hợp như quality gate chung vì đúng contract code và đủ nhạy với suy giảm
> trung bình đáng kể. Tuy nhiên 0.05 không đủ cho safety/privacy: một regression
> nghiêm trọng ở A02 có thể bị trung bình che khuất. Vì vậy giữ gate `drop > 0.05`
> cho từng answer metric, đồng thời thêm zero-tolerance gate theo case cho prompt
> injection, credential/privacy và hazardous-device guidance. Dùng nhiều lần
> chạy hoặc confidence interval nếu model không deterministic để tránh block do
> noise ngẫu nhiên.

**Câu 3: Metric/failure nào block deployment, metric nào chỉ alert?**

> Block khi Faithfulness, Relevance hoặc Completeness trung bình giảm hơn 0.05
> so với baseline; khi Faithfulness dưới 0.80; khi xuất hiện malformed output;
> hoặc bất kỳ adversarial safety/privacy case nào fail. Block nếu Context Recall
> giảm >0.05 ở nhóm policy-critical. Alert cho Context Precision giảm nhỏ nhưng
> answer metrics ổn, một lexical `off_topic` có human review xác nhận câu trả lời
> đúng nghĩa, hoặc thay đổi pass rate nằm trong variance đã hiệu chuẩn. Alert
> phải có owner và deadline, không được dùng để bỏ qua lỗi safety.

**Câu 4: Evaluation flow**

```text
Code/prompt/retrieval change → Offline golden evaluation → Regression comparison → Quality gate + targeted human review → Deploy
```

> Offline evaluation tạo artifact có trace; regression comparison dùng baseline
> cùng dataset; quality gate áp ngưỡng và kiểm tra case-critical. Chỉ deploy khi
> gate pass, sau đó tiếp tục online monitoring cho escalation, feedback và lỗi
> phân phối mới.

---

## 6. Continuous Improvement Loop

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Validate finish reason/cấu trúc output và retry fragment | Completeness, Relevance, Faithfulness | Loại ba worst failures do output bị cắt/méo. |
| 2 | Prompt checklist cho câu nhiều vế và điều kiện policy | Completeness, pass rate | Giữ đủ thời hạn, ngoại lệ và bước hành động. |
| 3 | Query expansion + rerank cho remedy/exception terms | Context Recall, Context Precision | Lấy đủ evidence phụ mà vẫn đặt chunk chính lên đầu. |

**Cases đề xuất cho benchmark vòng tiếp theo:**

> (1) Một câu hai vế về signature-required delivery và carrier pickup, với
> assertion output phải trả lời cả hai; (2) một case warranty sau return window
> yêu cầu liệt kê repair/replacement/refund; (3) một case OrbitPlus phân biệt
> opened 14 ngày với unopened 45 ngày. Đây là candidate cho phiên bản dataset
> tiếp theo; dataset nộp hiện tại vẫn giữ đúng 20 slots.

---

## 7. Final Reflection

**Điều gì trái với dự đoán ban đầu?**

> Retrieval tốt hơn dự đoán: precision 0.950 và recall 0.883, nhưng pass rate chỉ
> 30%. Hai case retrieval hoàn hảo vẫn nằm trong bottom three vì actual answer bị
> cắt. Điều này cho thấy tối ưu retriever trước tiên sẽ không xử lý được failure
> cluster quan trọng nhất; generation boundary và observability cần được sửa
> trước.

**Giới hạn của word-overlap và metric production:**

> Word overlap không hiểu phủ định, quan hệ điều kiện, ngày áp dụng hay paraphrase.
> Một câu lặp nhiều từ nguồn vẫn có thể áp dụng sai policy; ngược lại câu ngắn
> đúng nghĩa có thể bị relevance thấp. Nó cũng gán M05 là hallucination dù output
> thực tế là malformed. Trong production, tôi sẽ bổ sung claim-level
> entailment/groundedness, semantic answer relevance, policy-condition checklist,
> LLM-as-a-Judge đã calibrate với human labels, safety/privacy assertions và
> task-success metrics. Retrieval cần Recall@K/nDCG hoặc labeled relevance; mọi
> score tự động vẫn phải liên kết tới trace để human review các case rủi ro cao.
