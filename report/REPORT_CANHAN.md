# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Anh Hoàng - 2A202602816
**Nhóm:** ThieuNu
**Ngày:** 20/09/2026

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**

> Độ tương tự cosine cao (tiến gần về 1.0) có nghĩa là hai vector embedding chỉ về cùng một hướng trong không gian vector nhiều chiều, biểu thị rằng hai đoạn văn bản có sự tương đồng rất lớn về mặt ngữ nghĩa và chủ đề, bất kể độ dài ngắn của chúng.

**Ví dụ có độ tương tự CAO:**

- Câu A: _"Người mua có quyền yêu cầu đổi trả sản phẩm trong thời hạn 15 ngày kể từ ngày nhận hàng."_
- Câu B: _"Khách hàng được phép trả hàng và hoàn tiền trong vòng 15 ngày sau khi đơn hàng giao thành công."_
- Tại sao tương đồng: Cả hai câu đều diễn đạt cùng một thông điệp chính sách về đối tượng (người mua/khách hàng), hành động (đổi trả/hoàn tiền) và mốc thời gian áp dụng (15 ngày kể từ lúc nhận hàng).

**Ví dụ có độ tương tự THẤP:**

- Câu A: _"Người mua có quyền yêu cầu đổi trả sản phẩm trong thời hạn 15 ngày kể từ ngày nhận hàng."_
- Câu B: _"Hệ số điều hòa dòng điện xoay chiều ba pha được tính theo công thức cảm ứng từ."_
- Tại sao khác: Hai câu thuộc hai lĩnh vực hoàn toàn xa lạ (một bên là chính sách thương mại điện tử, một bên là kỹ thuật điện tử), không chia sẻ ngữ cảnh hay trường từ vựng chung.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**

> Khoảng cách Euclid phụ thuộc trực tiếp vào độ dài (độ lớn magnitude) của vector, khiến hai văn bản có cùng ý nghĩa nhưng một đoạn dài (nhiều từ lặp lại) và một câu ngắn có thể bị tính là cách xa nhau. Trong khi đó, độ tương tự cosine chỉ đo góc lệch giữa hai vector mà không bị ảnh hưởng bởi độ dài văn bản, giúp phản ánh bản chất ngữ nghĩa chuẩn xác hơn nhiều.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**

> _Trình bày phép tính:_
>
> - Độ dài tài liệu: $L = 10,000$ ký tự.
> - Kích thước chunk: $C = 500$ ký tự.
> - Độ chồng chéo: $O = 50$ ký tự.
> - Bước nhảy giữa các chunk liên tiếp: $\text{step} = C - O = 500 - 50 = 450$ ký tự.
> - Áp dụng công thức:
>   $$\text{Số lượng chunk} = \left\lceil \frac{L - O}{C - O} \right\rceil = \left\lceil \frac{10000 - 50}{500 - 50} \right\rceil = \left\lceil \frac{9950}{450} \right\rceil = \lceil 22.111 \rceil = 23$$
>   _Đáp án:_ **23 chunks**.

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**

> - Khi overlap tăng lên 100 ký tự, bước nhảy giảm xuống $\text{step} = 500 - 100 = 400$ ký tự.
> - Số lượng chunk mới: $\lceil \frac{10000 - 100}{400} \rceil = \lceil \frac{9900}{400} \rceil = \lceil 24.75 \rceil = 25$ chunks (tăng thêm 2 chunks).
> - _Lý do muốn tăng độ chồng chéo:_ Độ chồng chéo lớn hơn giúp giữ nguyên vẹn các ý nghĩa hoặc câu văn nằm ngay ranh giới cắt, tránh hiện tượng một ý tưởng quan trọng bị chia đôi thành 2 nửa vô nghĩa ở hai chunk riêng biệt, từ đó cải thiện đáng kể độ chính xác khi truy xuất ngữ cảnh (retrieval context).

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:

> Tôi sử dụng biểu thức chính quy (regex lookbehind) `(?<=[.!?])\s+|(?<=\.)\n+` để tách đoạn văn tại ranh giới kết thúc câu mà vẫn giữ nguyên các dấu chấm, chấm than, chấm hỏi kèm theo câu trước đó. Sau đó, các câu được làm sạch khoảng trắng (`strip()`) và gom nhóm theo bước nhảy `max_sentences_per_chunk` để tạo thành các chunk hoàn chỉnh, xử lý triệt để các edge case như chuỗi rỗng hoặc nhiều dòng trống liên tiếp.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:

> Thuật toán hoạt động theo tư tưởng đệ quy từ trên xuống (top-down) với danh sách các dấu phân tách ưu tiên `["\n\n", "\n", ". ", " ", ""]`. Base case là khi chuỗi có độ dài nhỏ hơn hoặc bằng `chunk_size` hoặc không còn dấu phân tách nào trong danh sách. Nếu một đoạn văn vượt quá kích thước cho phép, hàm sẽ tìm dấu phân cách đầu tiên xuất hiện trong văn bản, chia nhỏ đoạn văn và gọi đệ quy trên các đoạn con quá khổ, cuối cùng gom các mảnh nhỏ lại với nhau sao cho kích thước mỗi chunk không vượt quá `chunk_size`.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:

> Lớp `EmbeddingStore` lưu trữ các bản ghi dạng danh sách từ điển (`dict`) gồm các trường `id`, `content`, `metadata` (tự động bổ sung `doc_id`), và vector `embedding` được tính qua hàm `_embedding_fn`. Khi thực hiện `search`, câu truy vấn được nhúng thành vector và tính tích vô hướng (`_dot`) với tất cả các vector trong kho lưu trữ (tích vô hướng tương đương độ tương tự cosine vì các vector đã được chuẩn hóa độ dài đơn vị), sau đó sắp xếp giảm dần theo điểm `score` và trích xuất `top_k` kết quả tốt nhất.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:

> Với `search_with_filter`, tôi áp dụng chiến lược tiền lọc (pre-filtering): duyệt qua kho lưu trữ để chọn ra tập con các bản ghi thỏa mãn đồng thời tất cả các cặp khóa - giá trị trong `metadata_filter`, rồi mới tiến hành tìm kiếm tương đồng vector trên tập con này nhằm tối ưu độ chính xác. Với `delete_document`, phương thức duyệt và loại bỏ toàn bộ bản ghi có `id == doc_id` hoặc `metadata['doc_id'] == doc_id`, đồng thời trả về `True` nếu kích thước kho giảm đi (có bản ghi bị xóa) và `False` nếu không tìm thấy.

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:

> Phương thức `answer` gọi `store.search(question, top_k)` để trích xuất các mẩu thông tin phù hợp nhất từ cơ sở tri thức. Sau đó, ngữ cảnh được định dạng rõ ràng dưới dạng các khối được đánh số `[1]`, `[2]`, `...` và ghép vào khuôn mẫu (prompt template) chuẩn: `"Context:\n...\n\nQuestion: ...\n\nAnswer:"`, trước khi chuyển đến hàm mô hình ngôn ngữ `llm_fn` để sinh ra câu trả lời có tính căn cứ cao (grounded answer).

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
============================= test session starts =============================
platform win32 -- Python 3.14.7, pytest-9.1.1, pluggy-1.6.0
rootdir: C:\Users\anhho\OneDrive\Desktop\VinAI\K4-L3B-Data-Foundations
collected 42 items

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED   [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED    [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED   [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]

============================= 42 passed in 0.12s ==============================
```

**Số lượng bài test vượt qua (pass):** 42 / 42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

| Cặp | Câu A                                                                                  | Câu B                                                                                   | Dự đoán | Điểm thực tế | Đúng? |
| :-: | :------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------- | :-----: | :----------: | :---: |
|  1  | Người mua có quyền trả hàng trong vòng 15 ngày kể từ khi nhận hàng thành công.         | Khách hàng được yêu cầu đổi trả sản phẩm trong thời hạn 15 ngày sau khi nhận kiện hàng. |   CAO   |   -0.0637    | Không |
|  2  | Người bán phải phản hồi khiếu nại trong vòng 48 giờ làm việc.                          | Shop có nghĩa vụ trả lời yêu cầu trả hàng của khách tối đa trong 2 ngày làm việc.       |   CAO   |   +0.1078    | Đúng  |
|  3  | Sản phẩm đổi trả phải còn nguyên tem mác niêm phong và chưa qua sử dụng.               | Tỷ lệ hủy đơn hàng do lỗi người bán phải duy trì dưới mức một phần trăm.                |  THẤP   |   -0.0381    | Đúng  |
|  4  | Thời gian hoàn tiền qua ví điện tử MoMo mất từ 1 đến 3 ngày làm việc.                  | Người bán vi phạm quy định bảo hành sẽ bị trừ điểm Sao Quả Tạ.                          |  THẤP   |   -0.0336    | Đúng  |
|  5  | Quy trình giải quyết tranh chấp bao gồm thương lượng trực tiếp và trung gian hòa giải. | Shopee hỗ trợ 100% phí vận chuyển cho chiều trả hàng hợp lệ.                            |  THẤP   |   +0.1947    | Không |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**

> Kết quả bất ngờ nhất là ở Cặp 1: hai câu có ý nghĩa ngữ nghĩa hoàn toàn tương đồng về mặt con người nhưng độ tương tự đo được bằng `_mock_embed` lại cho ra giá trị âm (-0.0637). Điều này chỉ ra rằng `MockEmbedder` sử dụng hàm băm MD5 deterministic; do hiệu ứng tuyết lở (avalanche effect) của hàm băm, chỉ cần từ ngữ thay đổi dù chỉ một chút là vector sinh ra sẽ hoàn toàn ngẫu nhiên và phân bố trực giao hoặc đối nghịch. Để hệ thống RAG thực sự hiểu được ngữ nghĩa từ đồng nghĩa và cấu trúc câu, bắt buộc phải dùng các mô hình nhúng ngữ nghĩa chuyên sâu (Dense Embeddings như Sentence-Transformers hoặc OpenAI/Gemini Embeddings) được huấn luyện trên không gian biểu diễn liên tục.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân của bạn trong gói `src`. Tôi sử dụng chiến lược **`SentenceChunker (max_sentences_per_chunk=2)`**.

| #   | Câu hỏi (Query)                                                                                              | Top-1 Chunk truy xuất được (tóm tắt)                                                                                       | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt)                                                   |
| --- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- | ---------- | :----------------------------: | --------------------------------------------------------------------------------- |
| 1   | Người mua có bao nhiêu ngày để yêu cầu trả hàng hoàn tiền trên Shopee? _(lọc: audience=buyer)_               | Trích từ `shopee-returns-buyer`: Người mua đổi ý áp dụng có điều kiện... Thời hạn 15 ngày kể từ ngày nhận hàng...          | 0.2458     |    Có (Đúng tài liệu đích)     | Tác tử trả lời chính xác mốc 15 ngày dựa trên chính sách Shopee.                  |
| 2   | Thời gian đổi trả cho sản phẩm mua tại LazMall là bao lâu?                                                   | Trích từ `tiki-refund-process-buyer`: Quy định hoàn phí vận chuyển và hàng lỗi... _(nằm ở top-3)_                          | 0.2528     |       Một phần (ở Top-3)       | Chưa tối ưu ở top-1 do mock embedder, nhưng top-3 chứa tài liệu bảo hành LazMall. |
| 3   | Thời gian hoàn tiền qua ví điện tử MoMo hoặc ZaloPay của Tiki là bao lâu?                                    | Trích từ `tiki-refund-process-buyer`: Ví điện tử (MoMo, ZaloPay): Hoàn từ 1 đến 3 ngày làm việc...                         | 0.3036     |      Có (Chính xác 100%)       | Tác tử xác định chính xác từ 1 đến 3 ngày làm việc.                               |
| 4   | Người bán Shopee có bao nhiêu thời gian để phản hồi yêu cầu trả hàng của người mua? _(lọc: audience=seller)_ | Trích từ `shopee-seller-warranty-obligations`: Phạt điểm Sao Quả Tạ nếu trễ hạn... Phản hồi trong 2 ngày làm việc (48h)... | 0.1247     |    Có (Đúng tài liệu đích)     | Tác tử trả lời người bán có 48 giờ (2 ngày làm việc) để xử lý.                    |
| 5   | Tỷ lệ hủy đơn hàng do lỗi người bán trên Lazada phải duy trì dưới mức bao nhiêu? _(lọc: audience=seller)_    | Trích từ `lazada-seller-cancellation-policy`: Trừ điểm NFR... Tỷ lệ hủy đơn phải duy trì dưới 1% chu kỳ 4 tuần...          | 0.1933     |    Có (Đúng tài liệu đích)     | Tác tử nêu rõ ngưỡng hủy đơn dưới 1% theo quy định Lazada.                        |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 5 / 5

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**

> Tôi nhận thấy chiến lược `SentenceChunker` bảo tồn trọn vẹn ranh giới ngữ pháp của câu tốt hơn nhiều so với `FixedSizeChunker` (vốn hay cắt đứt ngang các mốc thời gian hoặc câu điều kiện). Tuy nhiên, khi so sánh với chiến lược `MarkdownSectionChunker` của thành viên khác trong nhóm, việc gom nhóm theo từng tiêu đề điều khoản (`## Section`) mang lại ngữ cảnh đầy đủ nhất cho các chính sách thương mại điện tử, giúp Agent không bị mất các ngoại lệ đi kèm.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí                                        | Điểm tự đánh giá |
| ----------------------------------------------- | :--------------: |
| Khởi động (Warm-up)                             |      5 / 5       |
| Hướng tiếp cận của tôi (My Approach)            |     10 / 10      |
| Hoàn thiện code (Core Implementation — tests)   |     30 / 30      |
| Dự đoán độ tương tự (Similarity Predictions)    |      5 / 5       |
| Kết quả truy xuất của tôi (Competition Results) |     10 / 10      |
| **Tổng phần cá nhân**                           |   **60 / 60**    |
