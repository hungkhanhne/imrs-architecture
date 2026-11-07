# Nhật ký sử dụng AI - Lab 4

> Nhật ký này ghi tóm tắt các bước Codex đã hỗ trợ trong workspace, không phải bản ghi tương tác của sinh viên với Claude hoặc công cụ khác. Cột “Đánh giá của sinh viên” hiện là ghi chú gợi ý do AI soạn, cần sinh viên tự kiểm tra và xác nhận trước khi nộp.

| # | Công cụ | Mục đích | Yêu cầu đầu vào (tóm tắt) | Kết quả AI | Đánh giá của sinh viên (gợi ý, cần xác nhận) | Chỉnh sửa |
|---|---|---|---|---|---|---|
| 1 | OpenAI Codex | Đọc yêu cầu Lab 4 | Đọc manual, rút ra cấu trúc deliverable, drivers, so sánh D1-D3, sơ đồ, ADR và nhật ký AI. | Tạo danh sách yêu cầu và gói ngữ cảnh ban đầu. | Cần đối chiếu từng mục với manual và nguồn Lab 1. | Tạo `context-pack.md`; giữ ghi chú về mục chưa có nguồn. |
| 2 | OpenAI Codex | Đọc Lab 1 PDF | Đối chiếu UC, QA, ràng buộc kỹ thuật và kinh doanh với yêu cầu RAG. | Xác định QA01-QA03, UC04-UC07, BC03-BC04, TC01-TC03. | TTFT <1,5 giây và phục hồi <=5 giây là mục tiêu đề xuất trong báo cáo, không phải kết quả đo; yêu cầu tiếng Việt/y khoa cần thể hiện. | Cập nhật gói ngữ cảnh, drivers, traceability và ADR. |
| 3 | OpenAI Codex | Đọc Lab 2 MVC | Đọc văn bản và sơ đồ nhúng để giữ tên lớp, connector, DB và ranh giới hệ thống. | Ghi nhận Web App/View, API Gateway, FastAPI, `WebUIController`, `QueryOrchestrator`, Qdrant, PostgreSQL và LLM Serving Gateway. | Luồng mới nên kế thừa tên và ranh giới đã chọn trong Lab 2. | Thêm API Gateway và backend/controller vào sơ đồ RAG; chọn Qdrant để thống nhất baseline. |
| 4 | OpenAI Codex | Đọc Lab 3 Dynamic | Đối chiếu trình tự xử lý âm thanh bất đồng bộ và các giao thức. | Ghi nhận RabbitMQ/`TranscriptionWorker` là luồng riêng, không phải chuỗi truy vấn văn bản RAG. | Không trộn luồng âm thanh với RAG; chỉ tái sử dụng các thành phần/hạ tầng phù hợp. | Ghi rõ ranh giới này trong sơ đồ và nhật ký lỗi. |
| 5 | OpenAI Codex | Đọc Word Lab 4 tham khảo | Dùng file mới để so sánh chéo drivers, D1-D3, ADR và sơ đồ. | Thấy bản Word chọn Qdrant nội bộ, payload filter theo role, role phía máy chủ, kiểm tra quyền lần hai và cache theo role; sơ đồ tham khảo bỏ qua API Gateway. | Cần xem Word là tài liệu tham khảo, không chép xác nhận cá nhân của tác giả; kiểm tra các điểm chưa thống nhất với Lab 2. | Chuyển lựa chọn từ pgvector sang Qdrant; sửa flow để có Web App/API Gateway; ghi khác biệt và nguồn. |
| 6 | OpenAI Codex + tài liệu chính thức | Kiểm chứng khẳng định kỹ thuật | Kiểm tra Qdrant payload filter/index, API, triển khai nội bộ và hành vi lọc ANN của pgvector. | Xác nhận Qdrant hỗ trợ lọc payload và gRPC/HTTP; cần cấu hình bảo mật/vận hành khi tự triển khai. pgvector có thể trả ít hơn k khi ANN áp dụng filter sau index scan. | Tránh gọi mọi cơ chế lọc là pre-filter tuyệt đối; không khẳng định thứ tự ANN chưa được kiểm chứng. | Dùng thuật ngữ “lọc trong yêu cầu tìm kiếm”; thêm caveat và link tới tài liệu chính thức. |
| 7 | OpenAI Codex | Soạn và đồng bộ deliverable | Cập nhật drivers, so sánh, sơ đồ, ADR, context pack và nhật ký theo cấu trúc chung Lab 4-7. | Tạo bản nháp Markdown, Mermaid và hình xuất cho hai sơ đồ. | Sinh viên cần rà soát chính sách dữ liệu, rubric, ngưỡng chấp nhận và đánh giá cá nhân trước khi nộp. | Lưu vào cấu trúc `README.md`, `AI_LOG.md`, `context-pack.md`, `adr/`, `lab4/`, `lab5/`, `lab6/`, `docs/`, `.github/workflows/`. |

## Nguồn kỹ thuật đã kiểm tra

- [Tài liệu lọc Qdrant](https://qdrant.tech/documentation/search/filtering/)
- [Tài liệu cài đặt Qdrant](https://qdrant.tech/documentation/installation/)
- [README pgvector](https://github.com/pgvector/pgvector/blob/master/README.md#filtering)

## Phần sinh viên cần tự hoàn thiện

Thay hoặc xác nhận cột “Đánh giá của sinh viên” bằng nhận xét của chính mình; cập nhật nguồn và chỉnh sửa thực tế; xác nhận các mục tiêu TTFT/khôi phục; rà soát sơ đồ, quyết định D1-D3 và quy định của giảng viên trước khi nộp.
