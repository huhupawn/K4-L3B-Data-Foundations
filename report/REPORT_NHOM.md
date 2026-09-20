# Báo Cáo Nhóm — Lab 7: Embedding & Vector Store

> **Nộp 1 bản / nhóm.** Phần cá nhân (hướng tiếp cận, kết quả riêng, dự đoán…) mỗi thành viên nộp riêng trong `REPORT_CANHAN.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần nhóm: 40** = Lựa chọn tài liệu (10) + Thiết kế chiến lược (15) + Chất lượng truy xuất (10) + Thuyết trình (5).

---

## 1. Lựa chọn tài liệu (Document Set Quality) — Nhóm (10 điểm)

### Chủ đề (Domain) & Lý Do Chọn

**Chủ đề:** Chính sách Đổi trả, Hoàn tiền, Bảo hành và Quy chế Vận hành Người bán/Người mua trên các sàn Thương mại Điện tử tại Việt Nam (Shopee, Lazada, Tiki).

**Tại sao nhóm chọn chủ đề này?**

> Đây là miền tri thức có tính ứng dụng thực tế cực kỳ cao nhưng cấu trúc chính sách rất phức tạp, phân chia ranh giới nghiêm ngặt giữa quyền lợi Người mua (`buyer`) và nghĩa vụ Người bán (`seller`). Nếu hệ thống RAG không có chiến lược chia nhỏ (chunking) hợp lý và thiếu cơ chế lọc siêu dữ liệu (metadata filtering), tác tử rất dễ trả lời nhầm quy định xử phạt của người bán cho người mua hoặc ngược lại, gây ra sai lệch nghiêm trọng trong hỗ trợ khách hàng.

### Danh sách tài liệu (Data Inventory)

|  #  | Tên tài liệu                                                   | Nguồn (Source URL)                                                             | Ngày lấy / Phiên bản  | Số ký tự | Metadata đã gán                                           |
| :-: | -------------------------------------------------------------- | ------------------------------------------------------------------------------ | :-------------------: | :------: | --------------------------------------------------------- |
|  1  | Chính Sách Trả Hàng Và Hoàn Tiền Shopee Dành Cho Người Mua     | https://help.shopee.vn/portal/article/77242-Chinh-sach-tra-hang-va-hoan-tien   | 2026-09-18 (v2026.02) |  1,745   | audience=buyer, category=returns-policy, language=vi      |
|  2  | Chính Sách Bảo Hành Và Đổi Trả Dành Cho Người Mua LazMall      | https://www.lazada.vn/helpcenter/chinh-sach-bao-hanh-lazmall.html              | 2026-09-18 (v2026.01) |  1,480   | audience=buyer, category=warranty-policy, language=vi     |
|  3  | Quy Trình Hoàn Tiền Và Phương Thức Thanh Toán Tiki             | https://hotro.tiki.vn/s/article/phuong-thuc-va-thoi-gian-hoan-tien             | 2026-09-19 (v2026.03) |  1,633   | audience=buyer, category=refund-process, language=vi      |
|  4  | Nghĩa Vụ Bảo Hành Và Xử Lý Khiếu Nại Dành Cho Người Bán Shopee | https://banhang.shopee.vn/edu/article/1825-Quy-dinh-bao-hanh-danh-cho-Shop     | 2026-09-19 (v2026.02) |  1,512   | audience=seller, category=seller-obligations, language=vi |
|  5  | Quy Định Tỷ Lệ Hủy Đơn Hàng Và Chế Tài Người Bán Lazada        | https://sellercenter.lazada.vn/seller/helpcenter/ty-le-huy-don-va-che-tai.html | 2026-09-19 (v2026.01) |  1,520   | audience=seller, category=seller-regulations, language=vi |
|  6  | Quy Trình Hòa Giải Và Giải Quyết Tranh Chấp Sàn TMĐT           | https://chinhsach.thuongmaidientu.vn/quy-trinh-hoa-giai-tranh-chap             | 2026-09-19 (v2026.01) |  1,568   | audience=both, category=dispute-resolution, language=vi   |

**Danh sách kiểm tra quản trị dữ liệu (Data governance checklist):**

- [x] Tập tài liệu (Corpus) chỉ chứa nguồn công khai/được phép dùng và không chứa dữ liệu cá nhân, thông tin đăng nhập hoặc tài liệu nội bộ.
- [x] Mỗi tài liệu có `source_url`, `retrieved_at`, `document_version` (hoặc ngày hiệu lực) trong metadata.

### Cấu trúc Metadata (Metadata Schema)

| Trường metadata    | Kiểu  | Ví dụ giá trị                                             | Tại sao hữu ích cho truy xuất (retrieval)?                                                                               |
| ------------------ | :---: | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `audience`         | `str` | `buyer`, `seller`, `both`                                 | **Bắt buộc trong L3B**: Giúp lọc chính xác đối tượng áp dụng của điều khoản, tránh nhầm lẫn giữa người mua và người bán. |
| `category`         | `str` | `returns-policy`, `warranty-policy`, `seller-regulations` | Cho phép truy xuất chuyên sâu theo phân loại chính sách, tránh xung đột giữa quy trình hoàn tiền và bảo hành.            |
| `source_url`       | `str` | `https://help.shopee.vn/...`                              | Cung cấp nguồn trích dẫn kiểm chứng cho tác tử LLM trích xuất link tham khảo.                                            |
| `retrieved_at`     | `str` | `2026-09-18`                                              | Theo dõi tính cập nhật của thông tin chính sách TMĐT vốn biến động định kỳ.                                              |
| `document_version` | `str` | `2026.02`                                                 | Hỗ trợ phân biệt các bản cập nhật điều khoản qua từng năm/quý.                                                           |
| `language`         | `str` | `vi`                                                      | Phân loại ngôn ngữ khi mở rộng đa ngôn ngữ cho khách hàng quốc tế.                                                       |

---

## 2. Thiết kế chiến lược (Strategy Design) — Nhóm (15 điểm)

### Phân tích đường cơ sở (Baseline Analysis)

Chạy `ChunkingStrategyComparator().compare()` trên các tài liệu mẫu của nhóm:

| Tài liệu                                    | Chiến lược (Strategy)            | Số lượng Chunk | Độ dài trung bình |          Giữ được ngữ cảnh không?           |
| ------------------------------------------- | -------------------------------- | :------------: | :---------------: | :-----------------------------------------: |
| Quy Trình Hòa Giải Và Giải Quyết Tranh Chấp | FixedSizeChunker (`fixed_size`)  |       5        |       284.4       |      Một phần (bị đứt câu ở ranh giới)      |
| Quy Trình Hòa Giải Và Giải Quyết Tranh Chấp | SentenceChunker (`by_sentences`) |       3        |       433.0       |    Tốt (giữ nguyên từng câu hoàn chỉnh)     |
| Quy Trình Hòa Giải Và Giải Quyết Tranh Chấp | RecursiveChunker (`recursive`)   |       6        |       215.7       |    Khá tốt (tôn trọng ngắt đoạn và câu)     |
| Quy Định Tỷ Lệ Hủy Đơn Hàng Lazada          | FixedSizeChunker (`fixed_size`)  |       5        |       272.4       | Kém (mốc số phạt bị cắt rời khỏi điều kiện) |
| Quy Định Tỷ Lệ Hủy Đơn Hàng Lazada          | SentenceChunker (`by_sentences`) |       4        |       309.2       |  Tốt (các ý chế tài nằm trọn trong 2 câu)   |
| Quy Định Tỷ Lệ Hủy Đơn Hàng Lazada          | RecursiveChunker (`recursive`)   |       5        |       247.2       |     Tốt (chia theo từng gạch đầu dòng)      |

- **Code snippet:**

```python
import re

class MarkdownSectionChunker:
    """Cắt nhỏ văn bản chính sách dựa trên tiêu đề cấp 2 (##) của Markdown."""
    def chunk(self, text: str) -> list[str]:
        if not text:
            return []
        sections = re.split(r"(?=(?:^|\n)##\s+)", text.strip())
        return [sec.strip() for sec in sections if sec.strip()]
```

## 3. Câu hỏi đánh giá & Chất lượng truy xuất (Retrieval Quality) — Nhóm (10 điểm)

### Câu hỏi đánh giá & Câu trả lời chuẩn (nhóm thống nhất)

|  #  | Câu hỏi (Query)                                                                                              | Câu trả lời chuẩn (Gold Answer)                                                                          | Chunk nào chứa thông tin?                                    |
| :-: | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
|  1  | Người mua có bao nhiêu ngày để yêu cầu trả hàng hoàn tiền trên Shopee? _(lọc: audience=buyer)_               | Trong vòng 15 ngày kể từ thời điểm đơn hàng giao thành công (áp dụng cho cả Shopee Mall và hàng thường). | Mục 1 trong tài liệu `shopee-returns-buyer.md`               |
|  2  | Thời gian đổi trả cho sản phẩm mua tại LazMall là bao lâu?                                                   | 30 ngày kể từ ngày nhận hàng (so với 7 ngày của gian hàng thường).                                       | Mục 1 trong tài liệu `lazada-warranty-buyer.md`              |
|  3  | Thời gian hoàn tiền qua ví điện tử MoMo hoặc ZaloPay của Tiki là bao lâu?                                    | Từ 1 đến 3 ngày làm việc sau khi kiểm định hàng hoàn đạt chuẩn.                                          | Mục 1 trong tài liệu `tiki-refund-process-buyer.md`          |
|  4  | Người bán Shopee có bao nhiêu thời gian để phản hồi yêu cầu trả hàng của người mua? _(lọc: audience=seller)_ | Tối đa 2 ngày làm việc (48 giờ), quá hạn Shopee tự động hoàn tiền cho người mua.                         | Mục 1 trong tài liệu `shopee-seller-warranty-obligations.md` |
|  5  | Tỷ lệ hủy đơn hàng do lỗi người bán trên Lazada phải duy trì dưới mức bao nhiêu? _(lọc: audience=seller)_    | Dưới 1% tính theo chu kỳ 4 tuần liên tiếp; nếu vi phạm sẽ bị phạt OVL và NFR.                            | Mục 1 trong tài liệu `lazada-seller-cancellation-policy.md`  |

### Tổng hợp chất lượng truy xuất của nhóm

|  #  | Câu hỏi                                     | Chiến lược tốt nhất cho câu này       | Có chunk liên quan trong top-3? | Ghi chú                                                                  |
| :-: | ------------------------------------------- | ------------------------------------- | :-----------------------------: | ------------------------------------------------------------------------ |
|  1  | Người mua có bao nhiêu ngày đổi trả Shopee? | `SentenceChunker` & `MarkdownSection` |           Có (Top-1)            | Lọc `audience=buyer` loại trừ hoàn toàn chính sách của người bán.        |
|  2  | Thời gian đổi trả LazMall?                  | `MarkdownSectionChunker`              |           Có (Top-2)            | Truy xuất đúng điều khoản 30 ngày của LazMall.                           |
|  3  | Thời gian hoàn tiền MoMo/ZaloPay Tiki?      | `SentenceChunker`                     |           Có (Top-1)            | Trích xuất chính xác dòng quy định ví điện tử 1-3 ngày.                  |
|  4  | Thời hạn người bán Shopee phản hồi?         | `SentenceChunker` & `MarkdownSection` |           Có (Top-1)            | Lọc `audience=seller` giúp định vị thẳng tài liệu nghĩa vụ Shop.         |
|  5  | Tỷ lệ hủy đơn người bán Lazada?             | `SentenceChunker` & `MarkdownSection` |           Có (Top-1)            | Lọc `audience=seller` giúp loại bỏ nhầm lẫn với tỷ lệ hủy đơn của khách. |

**Lọc bằng metadata có giúp ích không? Ở câu hỏi nào?**

> Lọc bằng metadata (`audience: buyer` hoặc `seller`) mang lại sự khác biệt quyết định ở **Câu 1, Câu 4 và Câu 5**. Nếu không có bộ lọc này, khi hỏi về _"trả hàng"_ hay _"hủy đơn"_, hệ thống rất dễ lấy nhầm chunk quy định quyền hạn của người mua để trả lời cho câu hỏi về trách nhiệm của người bán (hoặc ngược lại). Việc tiền lọc (pre-filter) giúp thu hẹp không gian tìm kiếm, nâng tỷ lệ chính xác của top-1 lên tuyệt đối.

---

## 4. Thuyết trình (Demo) & Bài học nhóm — Nhóm (5 điểm)

**Những phân tích (insights) hay nhất nhóm sẽ trình bày:**

1. **Sự vượt trội của Chunking theo ngữ cảnh tài liệu (Document-aware Chunking)**: Cắt theo ranh giới câu hoặc đề mục Markdown vượt trội hoàn toàn so với cắt kích thước cố định ngẫu nhiên trong việc bảo toàn các mốc thời gian và chế tài pháp lý.
2. **Vai trò cốt lõi của Metadata Filtering trong RAG đa đối tượng**: Với các bài toán có nhiều nhóm người dùng (Người mua vs Người bán), metadata filtering là chốt chặn bắt buộc để ngăn chặn hiện tượng trích xuất thông tin chéo (cross-audience hallucination).
3. **Giới hạn của hàm băm Mock Embedding**: Việc mô phỏng vector bằng hash MD5 cho thấy rõ nhược điểm: không có khả năng nhận biết từ đồng nghĩa nếu không có mô hình Dense Embedding thực thụ.

**Bài học rút ra khi so sánh trong nhóm:**

> Cùng một bộ tài liệu và cùng một câu hỏi, nhưng việc chọn kích thước chunk và chiến lược cắt khác nhau tạo ra sự chênh lệch lớn về chất lượng câu trả lời. Chunk quá nhỏ làm mất điều kiện ràng buộc, chunk quá lớn đưa thừa thông tin nhiễu khiến LLM dễ bị xao lãng.

**Nếu làm lại, nhóm sẽ thay đổi gì trong chiến lược dữ liệu (data strategy)?**

> Nhóm sẽ:
>
> 1. Sử dụng mô hình nhúng thực thụ chuyên dụng cho tiếng Việt (như `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` hoặc `text-embedding-3-small`).
> 2. Bổ sung thêm trường metadata `subcategory` (ví dụ: `hang-dien-tu`, `thoi-trang`, `thuc-pham`) vì chính sách đổi trả của các ngành hàng này có các điều kiện ngoại lệ rất khác nhau.

---

## Tự Đánh Giá (Phần Nhóm)

| Tiêu chí                                 | Điểm tự đánh giá |
| ---------------------------------------- | :--------------: |
| Lựa chọn tài liệu (Document Set Quality) |     10 / 10      |
| Thiết kế chiến lược (Strategy Design)    |     12 / 15      |
| Chất lượng truy xuất (Retrieval Quality) |      8 / 10      |
| Thuyết trình (Demo)                      |      2 / 5       |
| **Tổng phần nhóm**                       |   **32 / 40**    |
