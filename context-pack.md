# Gói ngữ cảnh IMRS - Lab 4 và Lab 5

> Tóm tắt Lab 1-3, Word Lab 4 tham khảo và manual SAD. Tài liệu nguồn nằm cạnh thư mục `imrs-architecture/`.

## Phạm vi và baseline

IMRS giúp nhân viên y tế tra cứu bằng tiếng Việt trên khoảng 5.000 PDF nội bộ, trả lời bằng LLM nội bộ kèm nguồn/trang. Phác đồ chuyên môn chỉ dành cho bác sĩ; hướng dẫn bệnh nhân có quyền riêng.

Lab 2: Web App → API Gateway → backend Python/FastAPI (`WebUIController`, `QueryOrchestrator`, `PIIRedactor`, DB connector); PostgreSQL cho dữ liệu quan hệ, Qdrant cho vector, LLM Serving Gateway cho streaming. Lab 3: RabbitMQ/`TranscriptionWorker` xử lý âm thanh bất đồng bộ, tách khỏi RAG văn bản.

## Ràng buộc Lab 1

- Dữ liệu, câu hỏi, vector, prompt và suy luận ở trong mạng bệnh viện; không gọi dịch vụ công cộng.
- Mục tiêu hạ tầng: máy chủ nội bộ, một RTX 4090 24 GB dùng chung cho LLM/embedding; Lab 5 dự kiến thêm tóm tắt bệnh án. Ngân sách <=15.000 USD; thời hạn bốn tháng.
- 100 kết nối đồng thời; TTFT <1,5 giây là mục tiêu đề xuất cần đo.
- Chặn/ghi log tấn công SQL hoặc prompt injection; mục tiêu rò rỉ PII 0% trong bộ kiểm thử.
- LLM chính lỗi: chuyển sang bản dự phòng ngoại tuyến, không làm sập ứng dụng; phục hồi <=5 giây là mục tiêu đề xuất cần kiểm chứng.
- Giao diện hỗ trợ tiếng Việt và thuật ngữ y khoa.

## RAG và quyết định hiện tại

Nạp PDF → trích xuất → chia đoạn → embedding cục bộ → Qdrant lưu vector cùng nguồn/trang/`allowed_roles`. Một `EmbeddingService` logic dùng cùng model/phiên bản ở ingestion và query. Tạo payload index cho `allowed_roles` trước ingestion. Khi hỏi, máy chủ xác minh role, lọc payload trong yêu cầu tìm kiếm, kiểm tra quyền lần hai, làm sạch ngữ cảnh, gọi LLM nội bộ và trả trích dẫn. `AuditLogger` ghi quyết định/ID nguồn vào PostgreSQL, không ghi nội dung thô. Cache phân vùng theo role, phiên bản chính sách và corpus.

- **D1:** Qdrant nội bộ theo Lab 2; PostgreSQL giữ dữ liệu quan hệ và audit.
- **D2:** embedding đa ngôn ngữ cục bộ qua `EmbeddingService`; chưa chọn model cụ thể.
- **D3:** lọc role trong yêu cầu vector, rồi kiểm tra quyền lần hai ở ứng dụng.

## Lab 5 — serving baseline có điều kiện

- `LLM Serving Gateway` định tuyến tới Primary/Fallback nội bộ; Fallback trả `degraded=true` để giao diện hiển thị chế độ suy giảm.
- Cấu hình khởi điểm trong ADR-0003: batch tối đa 4, `MAX_WAIT_TIME=50 ms`; queue có giới hạn nhưng giới hạn cụ thể và hành vi khi đầy cần xác định bằng thử tải.
- `CircuitBreaker` dùng Closed/Open/Half-Open; retry có giới hạn khi Closed, Open không gọi Primary, Half-Open chỉ cho một probe đồng thời. Timeout, failure threshold và reset timeout chưa có số đo.
- Một RTX 4090 24 GB dùng chung cho LLM và `EmbeddingService`; cùng model/version embedding được dùng ở ingestion/query. Tóm tắt bệnh án cũng dự kiến dùng tài nguyên AI này.
- Tốc độ đến giả định `100/30 ≈ 3,333 req/s`; thời gian GPU batch 1/4/8 là 0,4/0,6/0,9 giây. Đây là dữ liệu đề bài, không phải benchmark.
- `μ=B/t(B)` và `ρ=λ/μ` chỉ mô tả năng lực theo giả định batch đầy. `MAX_WAIT+t(B)` không phải TTFT hoặc độ trễ tối thiểu; mô hình nội suy `B*` không phải bằng chứng queue ổn định.
- Khi kiểm tra tải, đưa chung RAG, tóm tắt và embedding vào cùng kịch bản. Đo TTFT, queue depth, từ chối, fallback và tài nguyên GPU; chưa có kết quả đo trong baseline này.

## Còn cần xác định

Embedding model/phiên bản, kích thước chunk, top-k, TTL cache, cấu hình Qdrant, queue capacity, SLO và benchmark tải, timeout/threshold của breaker, năng lực Fallback, quy trình cập nhật/thu hồi quyền. Mục tiêu thời gian Lab 1 chưa phải kết quả đo.
