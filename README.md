# Kho bài thực hành `imrs-architecture`

Kho bài làm dùng chung cho Lab 4-7 của hệ thống IMRS. Lab 4 và Lab 5 được AI hỗ trợ tổng hợp; các giả định, thuật ngữ theo giáo trình và yêu cầu cá nhân cần sinh viên rà soát trước khi nộp.

## Cấu trúc

```text
README.md
AI_LOG.md
context-pack.md
adr/
  0001-chon-vector-store.md
  0002-co-che-phan-quyen-truy-xuat.md
  0003-cau-hinh-dynamic-batching.md
lab4/
  LAB4_RAG_DESIGN.md
  images/
lab5/
  LAB5_C4_SERVING.md
  LAB5_REVIEW_REPORT.md
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
- [ADR-0003 - Dynamic Batching](adr/0003-cau-hinh-dynamic-batching.md): cấu hình batch khởi điểm có điều kiện và các rủi ro/đánh đổi.
- [Lab 5 - C4 và phục vụ mô hình](lab5/LAB5_C4_SERVING.md): tactic mapping, C4 Level 1-3, luồng lỗi, deployment, capacity và quyết định serving.
- [Báo cáo rà soát bản tham khảo Lab 5](lab5/LAB5_REVIEW_REPORT.md): phát hiện đối chiếu Word/Excel và căn cứ theo section/sheet/cell.
- [Gói ngữ cảnh](context-pack.md): các ràng buộc Lab 1 và baseline Lab 2-5.
- [Nhật ký sử dụng AI](AI_LOG.md): tóm tắt hỗ trợ AI và việc sinh viên cần tự rà soát.
- [Hồ sơ kiến trúc](docs/SAD.md): bản nền sau Lab 4, sẽ được tổng hợp/cập nhật ở Lab 7.

Tài liệu nguồn PDF/Word của Lab 1-4 được giữ trong thư mục làm việc bên cạnh các deliverable để tra cứu. Bản nộp cuối cần bảo đảm các file tham chiếu và chính sách lưu trữ phù hợp với yêu cầu của lớp.
