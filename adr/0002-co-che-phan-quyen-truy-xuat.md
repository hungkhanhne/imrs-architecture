# ADR-0002: Áp dụng quyền trong yêu cầu tìm kiếm RAG

- **Trạng thái:** Được chấp nhận cho thiết kế Lab 4; cần xác nhận qua rà soát bảo mật.
- **Ngày:** 2026-11-07
- **Ngữ cảnh:** Corpus gồm phác đồ chuyên môn chỉ dành cho bác sĩ và tài liệu hướng dẫn bệnh nhân. Rủi ro là đoạn trái quyền bị trả vào ứng dụng rồi lọt vào prompt, cache hoặc log. Lab 1 yêu cầu phân quyền, chặn/ghi log các tình huống tấn công và bảo vệ PII. Lab 2 đã có `WebUIController` và `QueryOrchestrator`; bản Word Lab 4 đề xuất Qdrant payload filter với role lấy từ JWT/phiên đã xác thực và một lượt kiểm tra quyền bổ sung.

## Quyết định

`AccessPolicy` lấy principal và role từ phiên đã được xác thực ở phía máy chủ; không tin role do trình duyệt gửi. Trên cache miss, `VectorDBConnector` gửi tìm kiếm đến Qdrant cùng điều kiện payload `allowed_roles` được tạo từ role đã xác minh. Payload quyền được tạo từ nguồn quản trị tin cậy và có chỉ mục cho trường lọc.

Sau khi nhận kết quả, IMRS Backend kiểm tra lại metadata quyền của từng đoạn trước khi đoạn được đưa vào prompt hoặc cache. Nếu danh tính không hợp lệ hoặc không có role được phép, hệ thống từ chối theo hướng fail-closed và ghi sự kiện audit không có câu hỏi/nội dung bệnh án thô. Cache key bao gồm tối thiểu dấu vân tay câu hỏi, role, phiên bản chính sách và phiên bản corpus.

Tài liệu gọi đây là **lọc trong yêu cầu tìm kiếm**. Quyết định không khẳng định rằng mọi thuật toán tìm kiếm nội bộ của Qdrant loại mọi ứng viên trái quyền trước khi duyệt chỉ mục.

## Các phương án đã cân nhắc

- **Lọc trong yêu cầu Qdrant rồi kiểm tra lại ở ứng dụng:** chọn vì bộ lọc quyền được áp dụng tại nơi truy xuất, trước khi kết quả được dùng ở tầng ứng dụng. Lượt kiểm tra thứ hai giảm rủi ro nếu metadata lỗi hoặc phản hồi không như mong đợi.
- **Lấy top-k chung rồi lọc ở ứng dụng:** loại vì đoạn trái quyền đã sang ứng dụng trước khi bị loại, làm tăng bề mặt rò rỉ qua prompt/cache/log. Khi xóa các kết quả không được phép, số đoạn còn lại cũng có thể thấp hơn `k`; truy vấn bổ sung có thể tăng kết quả nhưng không khắc phục việc đoạn trái quyền đã đi vào ứng dụng.
- **Tách collection theo role:** chưa chọn làm mặc định vì tăng khối lượng đồng bộ khi thay đổi quyền/tài liệu. Có thể cân nhắc nếu chính sách bệnh viện yêu cầu cô lập tập dữ liệu mạnh hơn.
- **Role do client tự khai báo:** loại vì người gọi có thể sửa giá trị và giả mạo vai trò.

## Hệ quả

### Tích cực

- Role bắt nguồn từ phiên đã xác thực, thống nhất tại một `AccessPolicy` phía máy chủ.
- Truy vấn vector có điều kiện quyền; trước khi gọi LLM, ứng dụng kiểm tra lại từng đoạn.
- Cache được phân vùng theo quyền/chính sách/corpus, hạn chế trả nhầm câu trả lời giữa nhóm người dùng.
- Log cho phép điều tra quyết định truy cập và nguồn sử dụng mà không lưu nội dung y tế thô.

### Tiêu cực và rủi ro

- Metadata `allowed_roles` sai, cũ hoặc do nguồn không tin cậy tạo ra có thể mở quyền sai.
- Thiếu payload index hoặc thiết kế lọc chưa phù hợp có thể làm giảm hiệu năng; cần đo với phân bố role thực tế.
- Kiểm tra hai lần không thay thế quản trị khóa, phân quyền mạng, bảo vệ connector và kiểm thử bảo mật.
- Cache không bao gồm phiên bản chính sách/corpus có thể trả câu trả lời cũ sau khi quyền tài liệu thay đổi.
- Bộ lọc có thể làm số kết quả thực tế ít hơn top-k mong muốn; ứng dụng phải xử lý trường hợp ít kết quả hoặc không có nguồn được phép.

## Việc cần làm tiếp

1. Lập payload index cho `allowed_roles` trước ingestion; kiểm tra logic OR/AND của nhiều role bằng dữ liệu thử.
2. Kiểm thử truy cập chéo role, danh tính thiếu/hết hạn, cache hit sau thay đổi quyền, prompt injection và audit log.
3. Quy định quy trình phê duyệt, cập nhật, tái lập chỉ mục và thu hồi quyền tài liệu.
4. Đặt chính sách TTL/vô hiệu hóa cache khi thay đổi quyền, corpus hoặc phiên bản mô hình.
5. Đo recall/top-k sau lọc; không bỏ bộ lọc quyền để cải thiện số lượng kết quả.

## Nguồn

- [`L1_PhanHungKhanh.pdf`](../../L1_PhanHungKhanh.pdf): UC04-UC07, QA02, BC03.
- [`LAB2_MVC.docx`](../../LAB2_MVC.docx): `WebUIController`, `QueryOrchestrator`, `VectorDBConnector`, Qdrant.
- [`LAB4_RAG_DESIGN.docx`](../../LAB4_RAG_DESIGN.docx): payload filter theo role, kiểm tra bổ sung và cache phân quyền.
- [Tài liệu lọc Qdrant](https://qdrant.tech/documentation/search/filtering/): điều kiện payload và payload index.
- [Cài đặt Qdrant](https://qdrant.tech/documentation/installation/): cấu hình bảo mật và API gRPC/HTTP.
