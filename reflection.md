# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Dưới đây là kết quả tổng hợp thu được từ Exercise 3.2:

**Overall pass rate:** **10.0**%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.32 | 0.00 | 1.00 | 0.30 |
| Relevance | 0.31 | 0.00 | 0.75 | 0.21 |
| Completeness | 0.34 | 0.00 | 1.00 | 0.36 |
| Overall Score | 0.32 | 0.00 | 0.83 | 0.28 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8–1.0)? **0**
- Bao nhiêu metrics ở Needs Work (0.6–0.8)? **0**
- Bao nhiêu metrics ở Significant Issues (<0.6)? **3** (Cả 3 metrics Faithfulness, Relevance, Completeness đều rơi vào ngưỡng báo động đỏ do agent mock trả về lệch mục tiêu ngữ nghĩa).

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 9 | 50.0% |
| irrelevant | 3 | 16.7% |
| incomplete | 2 | 11.1% |
| off_topic | 4 | 22.2% |
| refusal | 0 | 0.0% |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

### Failure 1

**Question:** *What is the default relevance threshold in RAGASEvaluator?*

**Agent Answer:** *RAG stands for Retrieval-Augmented Generation. It combines retrieval search with LLM generation of text to ground the answers in context.*

**Scores:** Faithfulness: 0.00 | Relevance: 0.00 | Completeness: 0.00 | Overall: 0.00

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent trả lời sai hoàn toàn nội dung câu hỏi về cấu hình threshold, thay vào đó trả lời định nghĩa RAG. |
| Why 1 | Tại sao xảy ra? | Bộ phân loại ý định (intent classifier) định tuyến sai câu hỏi chứa từ khóa "RAGASEvaluator" thành câu hỏi tìm định nghĩa "RAG". |
| Why 2 | Tại sao Why 1 xảy ra? | Vì logic định tuyến trong mã nguồn agent sử dụng so khớp chuỗi thô (`if "RAG" in question`), dẫn đến việc từ khóa "RAG" trong "RAGASEvaluator" kích hoạt luồng trả lời RAG trước. |
| Why 3 | Tại sao Why 2 xảy ra? | Chưa xây dựng bộ phân loại ngữ nghĩa hoặc phân loại từ khóa theo thứ tự ưu tiên từ cụ thể đến bao quát. |
| Why 4 | Root cause là gì? | Bộ định tuyến ý định (Intent Router) của agent thiếu kiểm soát ngữ nghĩa và bị lỗi logic trùng lặp từ khóa. |

**Root cause (from `find_root_cause()`):**
> *Multiple issues detected — review full pipeline*

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> **Đồng ý**. Điểm số của cả 3 metrics đều rơi về 0.0 do agent trả lời lạc sang một chủ đề khác hoàn toàn so với tài liệu ngữ cảnh được cấp. Lỗi định tuyến ở pha đầu làm sụp đổ toàn bộ pipeline phía sau (lấy sai context và trả lời sai).

**Proposed fix (cụ thể, actionable):**
> Nâng cấp bộ định tuyến intent bằng cách sử dụng bộ phân loại ngữ nghĩa dựa trên Embeddings hoặc sắp xếp lại thứ tự kiểm tra từ khóa chi tiết (ví dụ: kiểm tra `"RAGASEvaluator"` trước `"RAG"`).

---

### Failure 2

**Question:** *What does BM25 stand for?*

**Agent Answer:** *BM25 stands for Best Matching 25. It is a ranking function used in document retrieval.*

**Expected:** *BM25 stands for Best Matching 25, a classic ranking function used in document retrieval.*

**Scores:** Faithfulness: 0.50 | Relevance: 0.25 | Completeness: 0.91 | Overall: 0.55

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Điểm Relevance cực thấp (0.25) mặc dù câu trả lời thực tế rất chính xác và liên quan sát sườn đến câu hỏi. |
| Why 1 | Tại sao xảy ra? | Do số lượng từ khóa trùng khớp trực tiếp giữa câu trả lời và câu hỏi thô là rất ít sau khi lọc bỏ stopwords. |
| Why 2 | Tại sao Why 1 xảy ra? | Từ khóa chung duy nhất chỉ là "BM25" và "stand", các từ khác trong câu hỏi như "What", "does", "for" đều bị loại bỏ bởi stopword list. |
| Why 3 | Tại sao Why 2 xảy ra? | Thuật toán đánh giá Relevance sử dụng lexical word-overlap thô thay vì đo lường sự tương đồng ngữ nghĩa thực sự của câu hỏi và câu trả lời. |
| Why 4 | Root cause là gì? | Hạn chế của công cụ đo lường (Metric Limitation): Word-overlap heuristic không phản ánh đúng Relevance thực tế. |

**Root cause:**
> *Answer does not address the question — improve prompt clarity* (đề xuất từ code dựa trên điểm số thấp nhất).

**Bạn có đồng ý không? Tại sao?**
> **Không đồng ý hoàn toàn**. Code đề xuất sửa prompt clarity, nhưng thực tế lỗi nằm ở bộ đo lường (heuristic metric). Câu trả lời của agent rất tốt. Thay đổi prompt không làm thay đổi bản chất của metric so khớp từ thô. Ta cần nâng cấp metric Relevance lên dạng so khớp ngữ nghĩa.

**Proposed fix:**
> Sử dụng embeddings similarity (như cosine similarity) hoặc LLM-as-Judge để đánh giá Answer Relevancy thay vì dùng đếm số từ trùng lặp.

---

### Failure 3

**Question:** *Should I use RAG or fine-tuning for my legal document analysis chatbot?*

**Agent Answer:** *You should use fine-tuning because legal chatbots need to write like lawyers.*

**Expected:** *For legal documents, use RAG because legal queries require exact citation of source texts, factual accuracy, and the documents update frequently. Fine-tuning should only be used to adapt the tone or legal writing style.*

**Scores:** Faithfulness: 0.25 | Relevance: 0.45 | Completeness: 0.25 | Overall: 0.32

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent khuyên dùng giải pháp kỹ thuật sai lệch hoàn toàn so với Expected (khuyên dùng fine-tuning thay vì RAG). |
| Why 1 | Tại sao xảy ra? | LLM Generator bị ảo giác (hallucination), đưa ra kiến thức cũ trong trọng số của nó thay vì trích xuất từ tài liệu ngữ cảnh. |
| Why 2 | Tại sao Why 1 xảy ra? | Prompt của agent chưa bắt buộc nghiêm ngặt mô hình phải bám sát ngữ cảnh được cung cấp và từ chối nếu không tìm thấy căn cứ. |
| Why 3 | Tại sao Why 2 xảy ra? | Thiếu hệ thống kiểm tra chéo (hallucination checker) để rà soát câu trả lời trước khi trả về cho người dùng. |
| Why 4 | Root cause là gì? | LLM Generator thiếu tính grounding vào tài liệu tìm kiếm và thiếu cơ chế phát hiện ảo giác tự động. |

**Root cause:**
> *Multiple issues detected — review full pipeline*

**Proposed fix:**
> Cải tiến System Prompt của agent để ép buộc mô hình bám sát tài liệu context: *"Chỉ sử dụng thông tin trong ngữ cảnh được cung cấp để trả lời câu hỏi. Không được bịa đặt."*. Đồng thời tích hợp một bộ phân loại hallucination tự động ở ngõ ra.

---

## 3. Failure Clustering

Dựa trên kết quả benchmark của 18 ca lỗi, ta chia thành các cụm lỗi sau:

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| **1** | Lỗi định tuyến ý định câu hỏi (Intent Routing Errors) | E01, E02, M06, M10, A20 | High |
| **2** | Ảo giác của LLM Generator & Lấy thiếu context chi tiết (Hallucination / Poor Grounding) | M09, M12, H13, H14, H15, H16, A18, A19 | High |
| **3** | Giới hạn kỹ thuật của metric so khớp từ khóa thô (Metric Limitation) | E05, M07, M08, M11, H17 | Medium |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn **Cluster 2 (Hallucination / Poor Grounding)**. Vì đây là cụm lỗi lớn nhất chiếm tới 8 ca lỗi (44.4% lượng lỗi) và ảo giác thông tin kỹ thuật là lỗi nghiêm trọng nhất làm giảm sút lòng tin của người dùng. Sửa lỗi này sẽ giải quyết tận gốc các ca lỗi nặng của mức Hard và Adversarial.

---

## 4. Improvement Log (from `generate_improvement_log`)

Dưới đây là bảng Improvement Log xuất ra từ `generate_improvement_log()`:

```
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Multiple issues detected — review full pipeline | Refine retriever weights or search query expansion to fetch better context | Open |
| F003 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | off_topic | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F005 | incomplete | Answer is missing key information — increase context window or improve generation | Improve prompt clarity and relevance constraints by adding explicit guidelines | Open |
| F006 | irrelevant | Answer does not address the question — improve prompt clarity | Refine intent detection routing to match queries with the correct agent tools | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F008 | off_topic | Context is missing or irrelevant — improve retrieval | Refine retriever weights or search query expansion to fetch better context | Open |
| F009 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F011 | hallucination | Multiple issues detected — review full pipeline | Improve prompt clarity and relevance constraints by adding explicit guidelines | Open |
| F012 | hallucination | Multiple issues detected — review full pipeline | Refine intent detection routing to match queries with the correct agent tools | Open |
| F013 | hallucination | Multiple issues detected — review full pipeline | Implement hallucination checker to filter unsupported claims | Open |
| F014 | hallucination | Answer is missing key information — increase context window or improve generation | Refine retriever weights or search query expansion to fetch better context | Open |
| F015 | incomplete | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F016 | hallucination | Multiple issues detected — review full pipeline | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F017 | hallucination | Multiple issues detected — review full pipeline | Improve prompt clarity and relevance constraints by adding explicit guidelines | Open |
| F018 | off_topic | Answer does not address the question — improve prompt clarity | Refine intent detection routing to match queries with the correct agent tools | Open |
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. **Implement hallucination checker to filter unsupported claims** (Ngăn chặn các câu trả lời bịa đặt không có căn cứ từ context nguồn).
2. **Refine intent detection routing to match queries with the correct agent tools** (Sửa lỗi phân loại nhầm lẫn tài liệu ngữ cảnh).
3. **Increase chunk size in RAG pipeline to reduce context fragmentation** (Tăng tính đầy đủ và toàn diện cho câu trả lời thông qua context thấu đáo hơn).

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Tích hợp chạy tự động `run_regression()` trên tập Golden Dataset trong CI/CD pipeline tại các thời điểm:
> - Trước khi gộp nhánh phát triển vào nhánh chính (`pre-merge to main` pull requests).
> - Mỗi khi có cập nhật hoặc chỉnh sửa prompt template.
> - Khi thay đổi cấu hình tìm kiếm RAG (ví dụ: thay đổi thuật toán xếp hạng, đổi Rerank model hoặc thay đổi chunk size/overlap).

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> **Phù hợp**. Độ lệch chuẩn khi đánh giá RAG thường dao động trong khoảng 0.02 - 0.03. Thiết lập ngưỡng 0.05 giúp loại bỏ các biến động ngẫu nhiên không đáng kể của LLM nhưng đủ nhạy để phát hiện ngay lập tức các sự cố suy giảm chất lượng nghiêm trọng trước khi deploy.

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> **Block deployment**. Nếu chỉ alert, các phiên bản kém chất lượng vẫn có thể lọt ra production gây ảnh hưởng xấu đến trải nghiệm người dùng cuối. Việc block deployment đóng vai trò như một chốt chặn chất lượng nghiêm ngặt. Trade-off: Điều này có thể làm chậm tốc độ release sản phẩm nếu xảy ra trường hợp false alarm do sự trồi sụt ngẫu nhiên của API mô hình.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```
Code change → [Run Unit Tests] → [Run Offline Eval (20 QAs)] → [Run Regression vs Baseline] → Deploy
                 (bước 1)                 (bước 2)                        (bước 3)
```

---

## 6. Continuous Improvement Loop

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| **1** | Tái cấu trúc System Prompt để thắt chặt grounding, yêu cầu bám sát context thô. | **Faithfulness** | Giảm thiểu ảo giác, đưa điểm Faithfulness trung bình lên trên 0.85. |
| **2** | Chuyển đổi bộ định tuyến Intent từ so khớp từ khóa cứng sang Semantic Classifier dùng embeddings. | **Answer Relevancy** | Định tuyến chính xác 100% các câu hỏi đặc thù như `RAGASEvaluator`, tăng Relevancy lên > 0.80. |
| **3** | Triển khai mô hình Cross-Encoder Reranker sau pha Vector Retrieval. | **Context Precision** | Sắp xếp các tài liệu chứa câu trả lời đúng lên vị trí top đầu, tối ưu hóa tốc độ đọc của LLM. |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> - Thêm các câu hỏi so sánh trực tiếp nhiều giải pháp kỹ thuật phức tạp đòi hỏi tổng hợp thông tin sâu sắc.
> - Thêm các câu hỏi chứa thuật ngữ tương tự dễ nhầm lẫn (ví dụ: phân biệt "threshold" trong RAGAS vs "threshold" trong học máy thông thường).

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** **RAGAS-inspired heuristic** (đo đạc dựa trên so khớp từ thô).

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> Chọn **DeepEval**.

| Tiêu chí | Lý do chọn |
|----------|------------|
| **Focus phù hợp vì...** | Hỗ trợ G-Eval cho phép chấm điểm theo rubric tùy chỉnh rất linh hoạt, phản ánh đúng ngữ nghĩa câu trả lời thay vì chỉ đếm từ trùng khớp thô. |
| **CI/CD integration vì...** | Hỗ trợ chạy native qua pytest, tích hợp hoàn hảo với GitHub Actions và tự động đồng bộ kết quả lên Cloud Dashboard để giám sát. |
| **Team workflow vì...** | Dashboard Confident AI của DeepEval cung cấp biểu đồ trực quan, giúp cả kỹ sư phần mềm lẫn Product Manager dễ dàng phân tích xu hướng chất lượng qua từng bản build. |
