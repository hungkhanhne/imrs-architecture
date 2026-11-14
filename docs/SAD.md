# Hồ sơ kiến trúc phần mềm IMRS

**Trạng thái:** Bản nền sau Lab 4; sẽ được tổng hợp và hoàn thiện ở Lab 7.

## Phạm vi hệ thống

IMRS hỗ trợ nhân viên y tế tra cứu hồ sơ và phác đồ. Tính năng RAG trong Lab 4 truy vấn khoảng 5.000 PDF nội bộ và trả lời có trích dẫn. Dữ liệu y tế và xử lý AI phải nằm trong mạng bệnh viện.

## Kiến trúc thành phần hiện tại

- **Giao diện:** Web App/View; gửi yêu cầu qua API Gateway.
- **Backend:** Python/FastAPI với `WebUIController`, `QueryOrchestrator`, `PIIRedactor`, `VectorDBConnector`, `RelationalDBConnector`.
- **Kho quan hệ:** PostgreSQL cho dữ liệu quan hệ và `AILog`.
- **Kho vector:** Qdrant nội bộ cho vector RAG và payload nguồn/quyền.
- **Mô hình embedding:** một `EmbeddingService` logic dùng chung cho ingestion và query; cả hai dùng cùng model/phiên bản. Khi đổi model cần nhúng lại toàn kho.
- **LLM:** LLM Serving Gateway định tuyến mô hình nội bộ chính hoặc dự phòng; trả câu trả lời dạng streaming.
- **Tài nguyên AI:** LLM và embedding dùng chung GPU RTX 4090 24 GB; Lab 5 dự kiến bổ sung tác vụ tóm tắt, nên cần đo tranh chấp tài nguyên.
- **Xử lý âm thanh:** RabbitMQ và `TranscriptionWorker` theo luồng bất đồng bộ Lab 3; tách biệt với truy vấn văn bản RAG.

## Ranh giới bảo mật RAG

1. Người dùng xác thực qua Web App và API Gateway.
2. `AccessPolicy` lấy role từ phiên đã xác thực ở máy chủ.
3. `VectorDBConnector` gửi yêu cầu tìm kiếm Qdrant kèm bộ lọc `allowed_roles`.
4. Backend kiểm tra quyền lần hai trước khi dùng đoạn trong prompt/cache.
5. Cache được scope theo role, phiên bản chính sách và corpus.
6. `AuditLogger` ghi quyết định và ID nguồn, không ghi nội dung y tế thô.

## Quyết định đã ghi nhận

- [ADR-0001](../adr/0001-chon-vector-store.md): Qdrant nội bộ làm kho vector.
- [ADR-0002](../adr/0002-co-che-phan-quyen-truy-xuat.md): lọc role trong yêu cầu tìm kiếm và kiểm tra lại ở ứng dụng.

## Nội dung cần hoàn thiện ở Lab 5-7

C4 container/component đầy đủ, triển khai và phục vụ mô hình, kết quả đánh giá chất lượng/tải, sơ đồ tổng hợp, thuộc tính chất lượng đã đo, giả định, rủi ro và các quyết định kiến trúc còn lại. Không xem bản nền này là hồ sơ kiến trúc hoàn chỉnh.
