# ADR-0004: Guardrails cho hệ đa tác nhân CAAN

- **Trạng thái:** Đề xuất cho thiết kế Lab 6; chưa triển khai và chưa được stakeholder phê duyệt.
- **Ngày:** 2026-11-25

## Bối cảnh

CAAN theo đề có vòng lặp Orchestrator dựa vào phản hồi LLM để dừng; đưa nguyên văn đoạn RAG vào prompt; cho SchedulingAgent ghi lịch trực tiếp; chạy mã Python do LLM sinh trên máy chủ ứng dụng bằng tài khoản dịch vụ backend; và dùng chung Redis với khóa `task:<id>`. Đây là thiết kế cần đánh giá trước khi triển khai. Ràng buộc kế thừa gồm dữ liệu y tế ở mạng bệnh viện, một RTX 4090 24 GB dùng chung cho RAG/LLM/embedding và mục tiêu TTFT cần đo. Chưa có bằng chứng trong đầu vào để kết luận ACL, TTL, timeout, audit hay HA hiện không tồn tại ở phần khác của hệ thống.

## Quyết định

Đề xuất thực thi guardrails ở mã điều phối, gateway và hạ tầng, không dựa riêng vào prompt:

1. Bổ sung **Execution Governor** độc lập với LLM, giới hạn `MAX_STEPS`, `TOKEN_BUDGET`, `TOTAL_TIMEOUT`, timeout từng tool và retry tối đa có backoff. Khi hết ngân sách, Governor hủy tác vụ con, chuyển FAILED và thông báo người dùng. `MAX_STEPS=8`, `MAX_RETRIES=3` là giá trị tham khảo chưa có benchmark; token budget và timeout chưa được đặt số.
2. Tách **Sandbox Runner** khỏi IMRS Backend. Chạy mã bằng UID không phải root, `--network none`, giới hạn bộ nhớ/CPU/PID, filesystem chỉ đọc, chỉ mount dữ liệu được duyệt và timeout cưỡng chế. `--memory=256m` và `--cpus=0.5` lấy từ đề. Backend không có quyền Docker daemon/socket; runner có đặc quyền phải có API hẹp, danh tính riêng và được cô lập.
3. Mọi thao tác ghi tạo **đề xuất**; dịch vụ nghiệp vụ chỉ commit sau xác nhận từ người có quyền, kiểm tra quyền lần cuối và idempotency key.
4. Cấp quyền tối thiểu theo agent/tool; kiểm tra role và phạm vi bệnh nhân tại gateway ngay trước truy cập dữ liệu/công cụ.
5. Xem nội dung RAG là dữ liệu không tin cậy. Dấu phân định trong prompt chỉ hỗ trợ mô hình; không cho nội dung truy xuất tự kích hoạt tool. Gateway áp dụng allowlist và kiểm tra tham số.
6. Tách trạng thái theo phiên/bệnh nhân, ACL theo agent, TTL và chính sách xóa. Ghi audit sự kiện công cụ cần thiết, giảm thiểu PII, kiểm soát quyền đọc/sửa và thời gian lưu.

## Phương án đã cân nhắc

| Phương án | Đánh giá |
|---|---|
| Chỉ thêm chỉ thị an toàn vào system prompt | Không chọn làm lớp cưỡng chế: LLM có thể diễn giải sai hoặc bị nội dung trong ngữ cảnh tác động; prompt không giới hạn quyền OS/API. |
| Dùng một LLM khác làm người kiểm duyệt | Có thể bổ sung, không dùng làm kiểm soát duy nhất vì vẫn là đánh giá xác suất và tăng tải GPU dùng chung. |
| Bỏ thực thi Python tự do, thay bằng hàm phân tích allowlist | Giảm rủi ro và độ phức tạp; cần cân nhắc nếu sandbox an toàn không kịp trong thời hạn bốn tháng. Đây là phương án dự phòng khả thi. |
| Chạy mã trong backend và lọc chuỗi mã | Không chọn vì lọc cú pháp không tạo ranh giới quyền; tiến trình vẫn có quyền và tệp của backend. |

## Hệ quả

**Tích cực:** giới hạn vòng lặp và thời gian có thể cưỡng chế độc lập với đầu ra LLM; mã sinh không mặc nhiên thừa hưởng quyền backend; thao tác ghi có bước xác nhận; quyền theo agent và trạng thái theo phiên làm giảm phạm vi thiệt hại; audit giúp truy vết sự cố.

**Đánh đổi/rủi ro:** Governor có thể từ chối tác vụ hợp lệ nếu ngân sách quá thấp; retry/backoff làm tăng độ trễ; bước xác nhận thêm thao tác cho bác sĩ; sandbox giới hạn thư viện và dữ liệu, tốn thời gian khởi tạo/vận hành; runner có quyền điều khiển container là thành phần nhạy cảm; container dùng chung kernel không chống tuyệt đối thoát container; audit log có thể tự trở thành nguồn lộ PII nếu ghi quá nhiều hoặc giữ quá lâu. Các phần bổ sung phải phù hợp thời hạn bốn tháng và ngân sách tổng 15.000 USD; hiện chưa có ước lượng chi phí triển khai.

## Việc cần làm trước khi chấp nhận vận hành

- Xác minh quyền hiện hữu, nội dung Redis, cách cấp ID, TTL, audit và chính sách HA; không coi thông tin thiếu trong sơ đồ là chứng cứ hệ thống không có.
- Chốt `TOKEN_BUDGET`, `TOTAL_TIMEOUT`, timeout tool, retry và queue/hủy theo workload giả lập; đo TTFT khi RAG, tóm tắt và embedding dùng chung GPU.
- Thử bộ ca prompt injection, truy cập sai phạm vi bệnh nhân, thao tác ghi lặp, đọc tệp/mạng trong sandbox, vượt tài nguyên, Redis lỗi và runner timeout.
- Review quyền Docker daemon, mount path, secret, audit retention và quy trình xử lý FAILED với nhóm vận hành/bác sĩ.
- Cập nhật ADR bằng kết quả thử nghiệm trước khi triển khai; trạng thái hiện tại chỉ là đề xuất thiết kế cho Lab 6.

## Căn cứ

- `SAD-Lab-Manual-v3.pdf`, yêu cầu Lab 6 và rubric.
- `LAB6_EVALUATION_REPORT.docx`: tài liệu đối chiếu; các tham số ngoài ràng buộc đề đều cần kiểm chứng.
- `context-pack.md`, `lab4/LAB4_RAG_DESIGN.md`, `lab5/LAB5_C4_SERVING.md`, ADR-0001–0003.
