# SAGA Pattern - Phân tán giao dịch không dùng 2PC
- **Date:** 2026-08-14
- **Topic:** microservices
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn
>SAGA pattern là gì? Khi nào dùng thay vì 2PC trong microservices?

## 2) Giải thích đơn giản
2PC (Two-Phase Commit) giống như "hỏi cả nhóm rồi mới làm": tất cả participant phải sẵn sàng trước, coordinator mới ra lệnh commit. Nhưng nếu 1 participant lỗi hoặc mạng lag → tất cả rollback, và trong thời gian chờ → resource lock.

SAGA pattern khác: thay vì commit 1 lần, ta chia giao dịch lớn thành **chuỗi transaction nhỏ**, mỗi bước có **compensating transaction** (thao tác "undo") riêng. Nếu bước thứ N fail → chạy compensating của N-1, N-2... ngược về đầu.

Kiểu "làm dần, sai thì dọn dần", không lock dài.

| So sánh | 2PC | SAGA |
|---------|-----|------|
| Consistency | Atomic (tấtước cùng) | Eventually consistent |
| Lock time | Dài | Ngắn / không lock |
| Rollback | Coordinator quyết định | Mỗi service tự undo |
| Coordination | Cần coordinator | Có thể orchestration hoặc choreography |
| Failure | Block toàn cục | Từng bước rollback |

## 3) Trả lời Middle+
SAGA là pattern để quản lý giao dịch phân tán trong microservices mà không dùng 2PC blocking.

Có 2 cách tổ chức:
- **Orchestration:** 1 service trung tâm (saga orchestrator) gọi lần lượt từng step và ra lệnh compensating khi fail.
- **Choreography:** Mỗi service sau khi xong phát event, service sau lắng nghe và chạy. Không trung tâm, nhưng khó trace.

Ví dụ đặt hàng:
1. Order service tạo order → success
2. Payment service trừ tiền → success
3. Inventory service trừ kho → **fail (hết hàng)**

→ Chạy compensating:
- Payment refund tiền
- Order đổi sang trạng thái cancelled

Ưu tiên SAGA khi:
- Các service thuộc不同 bounded context, không share DB
- Không chấp nhận lock dài (2PC)
- Chấp nhận eventual consistency

## 4) Trả lời Senior
SAGA giải quyết bài toán atomicity trong phân tán bằng eventual consistency, nhưng trade-off lớn là **không có isolation** (đọc có thể thấy dữ liệu giữa chừng).

Vấn đề thực tế:

**Race condition / dirty read:**
Bước 1 tạo order (status=PENDING). Bước 2 payment đang xử lý, user vừa hủy order.HOặc inventory hết hàng mà payment đã trừ tiền. Cần idempotency key trên mỗi compensating action, và state machine rõ ràng.

**Orchestration vs Choreography trade-off:**
- Orchestration: dễ debug, single source of truth cho flow, nhưng orchestrator thành single point of failure và bottleneck.
- Choreography: loosely coupled, scale tốt, nhưng khi fail khó biết ai đang ở đâu → cần correlation id + distributed tracing bắt buộc.

**Production concern:**
- Timeout giữa các step: nếu orchestrator không nhận response sau 30s → trigger compensating?
- Partial failure + retry: compensating idempotent, có thể dùng saga state machine lưu trạng thái (PENDING / COMPENSATING / FAILED / COMPLETED) trong DB riêng hoặc event log.
- Monitoring: cần trace toàn bộ saga instance (sagaId), alert khi compensation rate cao.
- Data inconsistency window: trong thời gian compensating chạy, hệ thống ở trạng thái inconsistent → frontend hoặc downstream service phải hiểu và handle gracefully.

**Khi KHÔNG dùng SAGA:**
- Giao dịch cần strong consistency (banking transfer nội bộ)
- Số bước < 3, có thể dùng transactional outbox + local transaction đơn giản hơn
-Database đã hỗ trợ distributed transaction tốt và team chấp nhận 2PC latency

## 5) 30 giây tóm tắt miệng
"SAGA pattern chia giao dịch lớn thành chuỗi bước nhỏ, mỗi bước có hành động rollback riêng gọi là compensating transaction. Nếu bước nào fail, hệ thống chạy compensating ngược về để đưa về trạng thái ban đầu thay vì rollback 1 lần như 2PC. Có 2 cách: orchestration có trung tâm điều phối, hoặc choreography mỗi service tự nghe event. Ưu điểm là không lock lâu, nhược điểm là eventually consistent và cần xử lý race condition, idempotency kỹ."

## Follow-up / pitfall
- **Interview twist:** "Làm sao đảm bảo compensating thực sự chạy?" → cần retry mechanism + dead-letter queue + alert; không thể "fire and forget".
- **Pitfall:** Chia saga quá nhỏ → quá nhiều message, overkill. Hoặc compensator không đúng nghĩa (ví dụ refund không cân bằng với charge).
- **Pitfall:** Choreography mà thiếu correlationId → khi debug không biết request nào thuộc saga nào.
