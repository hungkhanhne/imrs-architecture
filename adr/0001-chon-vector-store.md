# ADR-0001: Chọn kho vector cho RAG

- **Trạng thái:** Được chấp nhận cho thiết kế Lab 4; cần benchmark trước triển khai thật.
- **Ngày:** 2026-11-07
- **Ngữ cảnh:** IMRS cần tìm kiếm trên khoảng 5.000 PDF nội bộ, lưu vector cùng metadata nguồn và quyền truy cập. Tài liệu, truy vấn và vector phải ở trong mạng bệnh viện. Lab 2 đã xác định `VectorDBConnector` kết nối Qdrant bằng gRPC/REST; PostgreSQL được dùng cho dữ liệu quan hệ. Lab 4 mới bổ sung bản Word tham khảo cũng chọn Qdrant tự triển khai và giữ PostgreSQL cho audit log.

## Quyết định

Chọn Qdrant tự triển khai trong hạ tầng nội bộ làm kho vector RAG. `VectorDBConnector` là lớp truy cập duy nhất từ IMRS Backend; connector gửi vector, điều kiện payload `allowed_roles` và số lượng kết quả cần lấy. PostgreSQL tiếp tục lưu dữ liệu quan hệ và bảng `AILog`, không chuyển sang lưu vector trong quyết định này.

Mô hình embedding chạy cục bộ sau một `EmbeddingService` logic dùng chung cho ingestion và query; hai luồng phải dùng cùng `model_id`/phiên bản. Chưa khóa tên mô hình, chiều vector, cấu hình chỉ mục hoặc quy mô phần cứng Qdrant vì tài liệu hiện có không đưa ra benchmark tương ứng.

## Các phương án đã cân nhắc

- **pgvector trên PostgreSQL sẵn có:** hỗ trợ cosine và có thể đơn giản hóa số dịch vụ. Tuy nhiên, chưa xác nhận môi trường PostgreSQL thực tế, tài nguyên và tải chung; lựa chọn này cũng đổi khỏi baseline Qdrant trong Lab 2. Khi ANN dùng điều kiện `WHERE`, pgvector có thể lọc sau khi quét chỉ mục và trả ít hơn `k`. Có thể đánh giá lại nếu phép đo và điều kiện triển khai cho thấy đây là lựa chọn tốt hơn.
- **Qdrant tự triển khai nội bộ:** thống nhất với Lab 2 và bản Word tham khảo, hỗ trợ lọc payload trong yêu cầu tìm kiếm. Đổi lại, nhóm phải tự quản lý cấu hình bảo mật, lưu trữ bền vững, sao lưu, giám sát và nâng cấp.
- **Vector DB dịch vụ đám mây:** không chọn vì dữ liệu y tế và truy vấn nội bộ không được rời mạng bệnh viện; chưa có báo giá để đánh giá chi phí.

## Hệ quả

### Tích cực

- Giữ nhất quán tên và vai trò `VectorDBConnector`/Qdrant đã có từ Lab 2.
- Cho phép mang metadata nguồn, trang và quyền cùng vector để truy xuất có trích dẫn và áp dụng điều kiện role.
- Không gửi PDF, câu hỏi, vector hoặc ngữ cảnh sang dịch vụ AI/vector công cộng.
- Có thể thay đổi kho vector sau `VectorDBConnector` mà không để tầng giao diện phụ thuộc API Qdrant.

### Tiêu cực và rủi ro

- Thêm một dịch vụ cần theo dõi và vận hành riêng bên cạnh PostgreSQL, API Gateway, RabbitMQ và các thành phần Lab 2-3.
- Embedding và LLM cùng dùng GPU RTX 4090 mục tiêu; tác vụ dự kiến ở Lab 5 làm tăng khả năng tranh chấp tài nguyên, cần benchmark chung.
- Tự triển khai không tự đảm bảo an toàn; phải cấu hình xác thực/mạng, lưu trữ, backup/restore, giám sát và log.
- Tài nguyên CPU/RAM/đĩa và độ trễ chưa được đo cho corpus, payload và tải IMRS.
- Thay mô hình embedding cần nhúng lại tài liệu và tạo lại dữ liệu/vector có cùng không gian embedding.
- Dữ liệu quyền sai hoặc cũ trong payload có thể gây lộ tài liệu; phải có nguồn quản trị metadata và quy trình cập nhật.

## Việc cần làm tiếp

1. Tạo payload index cho `allowed_roles` trước khi nạp dữ liệu, theo hướng dẫn Qdrant.
2. Đo recall, độ trễ, tài nguyên GPU dùng chung và hành vi lọc trên corpus thử nghiệm không chứa PII thật.
3. Kiểm tra cấu hình mạng, xác thực, backup/restore, monitoring và quyền truy cập của connector.
4. Xem xét lại ADR nếu PostgreSQL sẵn có có năng lực đã được xác nhận và benchmark cho thấy pgvector đáp ứng tải/ACL tốt hơn.

## Nguồn

- `LAB2_MVC.docx`: `VectorDBConnector`, Qdrant và luồng gRPC/REST.
- `LAB4_RAG_DESIGN.docx`: bản tham khảo Lab 4 chọn Qdrant nội bộ, PostgreSQL cho quan hệ/audit.
- `L1_PhanHungKhanh.pdf`: ràng buộc nội bộ, bảo mật, chi phí và tiến độ.
- `SAD-Lab-Manual-v3.pdf`: yêu cầu Lab 4.
- [Lọc dữ liệu Qdrant](https://qdrant.tech/documentation/search/filtering/): lọc payload và lập payload index.
- [Cài đặt Qdrant](https://qdrant.tech/documentation/installation/): tùy chọn triển khai và lưu ý bảo mật/vận hành.
- [README pgvector](https://github.com/pgvector/pgvector/blob/master/README.md#filtering): cosine và hành vi lọc khi dùng ANN.
