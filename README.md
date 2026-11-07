# Kho bài thực hành `imrs-architecture`

Kho bài làm dùng chung cho Lab 4-7 của hệ thống IMRS. Nội dung bài Lab 4 hiện là bản nháp do AI hỗ trợ và cần sinh viên rà soát, xác nhận trước khi nộp.

## Cấu trúc

```text
README.md
AI_LOG.md
context-pack.md
adr/
  0001-chon-vector-store.md
  0002-co-che-phan-quyen-truy-xuat.md
lab4/
  LAB4_RAG_DESIGN.md
  images/
lab5/
  LAB5_C4_SERVING.md
lab6/
  LAB6_EVALUATION_REPORT.md
docs/
  SAD.md
.github/
  workflows/
    docs-check.yml
```

## Nội dung hiện có

- [Lab 4 - Thiết kế RAG](lab4/LAB4_RAG_DESIGN.md): drivers, so sánh D1-D3, kiểm chứng kỹ thuật, sơ đồ, nhật ký lỗi sơ đồ.
- [ADR-0001 - Kho vector](adr/0001-chon-vector-store.md): quyết định chọn Qdrant nội bộ.
- [ADR-0002 - Phân quyền truy xuất](adr/0002-co-che-phan-quyen-truy-xuat.md): lọc role trong yêu cầu tìm kiếm và kiểm tra quyền lần hai.
- [Gói ngữ cảnh](context-pack.md): các ràng buộc Lab 1 và baseline Lab 2-3.
- [Nhật ký sử dụng AI](AI_LOG.md): tóm tắt hỗ trợ AI và việc sinh viên cần tự rà soát.
- [Hồ sơ kiến trúc](docs/SAD.md): bản nền sau Lab 4, sẽ được tổng hợp/cập nhật ở Lab 7.

Tài liệu nguồn PDF/Word của Lab 1-4 được giữ trong thư mục làm việc bên cạnh các deliverable để tra cứu. Bản nộp cuối cần bảo đảm các file tham chiếu và chính sách lưu trữ phù hợp với yêu cầu của lớp.
