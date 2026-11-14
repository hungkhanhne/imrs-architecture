# ADR-0003: Cấu hình Dynamic Batching cho LLM Serving Gateway

- **Trạng thái:** Đề xuất có điều kiện cho Lab 5; cần đánh giá bằng thử tải trước khi coi là cấu hình vận hành.
- **Ngày:** 2026-11-14
- **Phạm vi:** Phục vụ truy vấn RAG và tác vụ AI nội bộ trên một GPU RTX 4090 24 GB.

## Bối cảnh

Giả định đề bài là 100 bác sĩ, mỗi người gửi một câu hỏi mỗi 30 giây, nên tốc độ đến trung bình `λ = 100/30 ≈ 3,333 req/s`. Thời gian xử lý GPU cho batch 1/4/8 lần lượt là `0,4/0,6/0,9 giây`; `MAX_WAIT_TIME` tham khảo là `50 ms`. Các số này là đầu vào giả định, chưa phải benchmark.

GPU dùng chung cho LLM và `EmbeddingService`; Lab 5 dự kiến bổ sung tóm tắt bệnh án. Bảng năng lực hiện chỉ tính tải truy vấn trợ lý phác đồ. ASR Performance nói đến TTFT ở 100 kết nối đồng thời; thời gian hoàn tất batch không thay thế phép đo TTFT.

## Quyết định

Đề xuất `BATCH_SIZE = 4` và `MAX_WAIT_TIME = 50 ms` cho `BatchScheduler` làm cấu hình khởi điểm để thử nghiệm. Các giá trị là tham số cấu hình, không hard-code.

Queue có giới hạn cấu hình. Khi queue đầy, gateway phải thực hiện back-pressure hoặc từ chối request với kết quả quá tải rõ ràng cho Backend; giá trị giới hạn và timeout chờ queue chưa được chọn vì chưa có số liệu đo. Không được để request chờ vô hạn.

`CircuitBreaker` quản lý Primary/Fallback độc lập với thuật toán gom lô: retry có giới hạn/backoff chỉ khi còn Closed; Open thì không gọi Primary; Half-Open chỉ cho một probe đồng thời. Câu trả lời từ Fallback phải kèm `degraded=true` để UI thông báo. `FAILURE_THRESHOLD`, timeout gọi mô hình, retry budget/backoff và `RESET_TIMEOUT` là tham số cần xác định qua thử nghiệm.

## Các phương án đã cân nhắc

| Phương án | Lợi ích theo giả định đề bài | Chi phí/đánh đổi | Quyết định |
|---|---|---|---|
| Batch tối đa 1 | Thời gian xử lý từng request giả định là 0,4 giây. | `μ=1/0,4=2,5 req/s`, thấp hơn `λ≈3,333 req/s`; mô hình queue vô hạn sẽ tăng backlog. Nếu queue bị giới hạn, hệ thống sẽ phải từ chối/back-pressure. | Không chọn làm cấu hình phục vụ tải giả định. |
| Batch tối đa 4 | `μ=4/0,6≈6,667 req/s`, `ρ≈0,5` khi batch đầy; thời gian xử lý batch giả định ngắn hơn batch 8. | 50 ms không bảo đảm batch đầy; `ρ` là năng lực lô đầy, không phải TTFT thực. | Chọn làm điểm khởi đầu có điều kiện. |
| Batch tối đa 8 | `μ=8/0,9≈8,889 req/s`, `ρ≈0,375` khi batch đầy. | Thời gian xử lý batch giả định là 0,9 giây; gom đủ batch khó khi queue rỗng. | Chưa chọn vì lợi ích thêm chưa được chứng minh cần thiết. |
| `MAX_WAIT=1 ms` | Giảm thời gian chờ gom khi GPU rảnh. | Theo phép xấp xỉ worksheet, lô chờ gần 1 request; ít cơ hội tăng batch khi hệ thống rảnh. | Dùng làm điểm so sánh thử nghiệm, không chọn mặc định. |
| `MAX_WAIT=500 ms` | Tăng cơ hội đợi thêm request khi GPU rảnh. | Trong mô hình worksheet, `W+t(E[B])` xấp xỉ 1,011 giây so với 0,461 giây tại 50 ms; đây không phải kỳ vọng thống kê chính xác hay TTFT. | Không chọn làm mặc định; giữ làm điểm nhạy cảm. |

## Căn cứ định lượng và giới hạn

| B | t(B), s | μ=B/t(B), req/s | λ, req/s | ρ=λ/μ | W+t(B), s theo công thức worksheet |
|---:|---:|---:|---:|---:|---:|
| 1 | 0,4 | 2,500 | 3,333 | 1,333 | 0,450* |
| 4 | 0,6 | 6,667 | 3,333 | 0,500 | 0,650 |
| 8 | 0,9 | 8,889 | 3,333 | 0,375 | 0,950 |

\* Workbook đánh dấu độ trễ không áp dụng cho batch 1 vì `ρ>1`. Phép cộng `W+t(B)` không phải “độ trễ tối thiểu”; đó là timer tối đa cộng thời gian xử lý trong một nhánh giả định. `t(B)` là thời gian hoàn tất batch, không phải TTFT.

Worksheet nội suy tuyến tính giữa B=1 và B=4 cho `a≈0,333 s`, `s≈0,0667 s/request`. Phương trình cân bằng `B=λ(a+sB)` cho `B*≈1,429`, `t(B*)≈0,429 s`, tại đó `λt(B*)/B*=1`. Nghiệm này là điểm cân bằng của mô hình nội suy ở mức sử dụng 100%, không chứng minh queue ổn định hoặc còn dư địa. Ước lượng `1,5×t(B*)` không có mô hình arrival/queue đủ để xem là latency dự báo.

Phép độ nhạy worksheet dùng `B≈min(Bmax,1+λW)` rồi nội suy `t(B)`. Đây là xấp xỉ; `min` của kỳ vọng không bằng kỳ vọng của batch bị chặn đối với arrival ngẫu nhiên. Không dùng thông lượng `B/t(E[B])` từ phép này làm capacity bền vững.

## Rủi ro

- **R1 — Tải GPU chưa đầy đủ:** bảng không cộng tác vụ tóm tắt và embedding trên cùng GPU. Kết quả capacity có thể không đại diện workload tổng.
- **R2 — TTFT chưa chứng minh:** `t(B)` đo thời gian hoàn tất cả batch theo giả định, không phải TTFT; SLO cần đo trong kịch bản 100 kết nối đồng thời.
- **R3 — Queue chưa được cấu hình:** thiếu queue cap và chính sách đầy; có nguy cơ chờ quá lâu hoặc từ chối không nhất quán.
- **R4 — Fallback chưa được đánh giá:** năng lực CPU và chất lượng mô hình dự phòng chưa có benchmark; degraded flag và thông báo phải xuyên suốt Backend/UI.
- **R5 — Retry có thể khuếch đại sự cố:** cần deadline và giới hạn retry; không retry khi breaker Open.
- **R6 — Cache có thể cũ hoặc lộ chéo quyền:** khóa cache phải gồm role, policy version và corpus version; kiểm tra quyền trước khi trả cache hit.
- **R7 — Threshold chưa có số liệu:** `FAILURE_THRESHOLD`, timeout và `RESET_TIMEOUT` chưa được quyết định; giá trị sai có thể gây chuyển dự phòng giả hoặc phục hồi chậm.

## Non-risks và điều chưa được chứng minh

- Phép tính `μ` và `ρ` cho batch 1/4/8 khớp các giả định đầu vào; đây là xác nhận số học, không phải xác nhận hiệu năng thật.
- `ρ<1` cho batch 4/8 chỉ áp dụng trong mô hình batch đầy; không chứng minh mọi request thực tế được xử lý trong SLO.
- `MAX_WAIT=50 ms` chỉ giới hạn thời gian gom lô theo cấu hình; không giới hạn tổng latency do queue, suy luận, truy xuất và mạng.
- Tài liệu dùng dữ liệu giả lập, không có dữ liệu bệnh nhân thật; điều đó không tự chứng minh thiết kế bảo vệ PII khi triển khai.
- Chưa có căn cứ trong đầu vào để kết luận chọn batch 4 làm giảm chi phí hoặc số GPU.

## Sensitivity Points

1. Tốc độ và phân bố arrival `λ`, bao gồm request dồn đợt.
2. Thời gian xử lý từng batch `t(B)` và quy luật nội suy giữa các batch chưa đo.
3. `BATCH_SIZE`, `MAX_WAIT_TIME` và queue capacity/hành vi khi đầy.
4. Tải đồng thời của RAG, tóm tắt và embedding trên RTX 4090.
5. Cache hit/miss và chi phí khi corpus/policy thay đổi.
6. Timeout, retry budget, `FAILURE_THRESHOLD`, `RESET_TIMEOUT` và năng lực Fallback.

## Trade-off Points

- **Batch size:** batch lớn tăng thông lượng tối đa theo mô hình lô đầy nhưng tăng thời gian xử lý batch và khó đầy khi ít request; batch nhỏ giảm thời gian xử lý từng lô nhưng có capacity thấp hơn.
- **MAX_WAIT:** tăng wait tạo cơ hội gom request nhưng tăng latency lúc GPU rảnh; ở tải cao, batch có thể hình thành từ queue, nên wait không phải đòn bẩy throughput duy nhất.
- **Queue cap:** queue lớn hấp thụ burst nhưng có thể tăng latency; queue nhỏ giữ giới hạn chờ tốt hơn nhưng từ chối nhiều request hơn khi burst.
- **Fallback CPU:** giúp tiếp tục phục vụ khi Primary lỗi nhưng chất lượng và năng lực có thể thấp hơn; cần hiển thị degraded, không che giấu chuyển chế độ.
- **Response cache:** giảm công việc lặp nhưng đổi lại nguy cơ dữ liệu cũ/phân quyền sai nếu key và invalidation không đủ.

## Câu hỏi cần trả lời trước khi chấp nhận cấu hình vận hành

1. SLO TTFT cụ thể là gì, đo từ mốc nào đến mốc nào, và có đạt khi RAG, tóm tắt, embedding cùng tải trên một RTX 4090 không?
2. Khi queue đầy hoặc retry đang diễn ra, hệ thống giới hạn tổng thời gian chờ và tránh khuếch đại tải bằng chính sách nào?
3. Fallback CPU có đủ năng lực và chất lượng/an toàn cho loại request nào; tiêu chí nào cho phép breaker trở lại Closed sau probe?

## Kế hoạch đánh giá lại

1. Dùng dữ liệu giả lập, không có PII thật; chạy workload 100 kết nối đồng thời gồm RAG, tóm tắt và embedding.
2. Ghi TTFT, throughput, queue depth, tỷ lệ từ chối, tỷ lệ fallback, trạng thái breaker và tài nguyên GPU.
3. So sánh batch tối đa 4/8 và wait 1/50/500 ms theo cùng workload; không dùng latency worksheet làm kết quả đo.
4. Thử queue đầy, timeout Primary, retry đến ngưỡng, Fallback lỗi và probe Half-Open; xác nhận UI luôn hiện degraded khi cần.
5. Cập nhật ADR khi có kết quả đo; chưa đổi quyết định nếu chưa có bằng chứng so sánh.

## Nguồn

- `LAB5_C4_SERVING.docx`: Phần A (tactics/C4/Sequence/Deployment), Phần B (capacity và độ nhạy), Phần C (ADR-0003).
- `LAB5_CAPACITY_ESTIMATE.xlsx`: `GiaDinh`, `NangLuc`, `DoNhay`; các giá trị thời gian và tải là giả định đề bài.
- `context-pack.md`: ràng buộc GPU, ngân sách, tiến độ, TTFT và dữ liệu nội bộ.
- `adr/0001-chon-vector-store.md`, `adr/0002-co-che-phan-quyen-truy-xuat.md`: baseline Lab 4.
