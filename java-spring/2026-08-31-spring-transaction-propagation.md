# Spring Transaction Propagation

- **Date:** 2026-08-31
- **Topic:** java-spring
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Hai bean gọi nhau trong cùng transaction, `orderService.placeOrder()` gọi `inventoryService.reserve()`. Khi `inventoryService` dùng `@Transactional(propagation = REQUIRES_NEW)` thì chuyện gì xảy ra? `REQUIRED` và `REQUIRES_NEW` khác nhau như thế nào ở mức production?

## 2) Giải thích như trẻ lên 3

`@Transactional` là thẻ "đảm bảo hoàn thiện" dán trên method. Khi method A gọi method B:

- `REQUIRED`: nếu A đang có thẻ → B **ghép chung** thẻ của A. B lỗi → cả A và B cùng rollback.
- `REQUIRES_NEW`: B **tạo thẻ riêng**, rời thẻ A ra. B rollback riêng; A chỉ có thể tiếp tục nếu xử lý exception từ B.

## 3) Giải thích đơn giản cho dev

- `REQUIRED`: tham gia transaction hiện tại; tạo mới nếu chưa có.
- `REQUIRES_NEW`: luôn tạo transaction mới, tạm dừng transaction hiện tại.

Spring dùng `TransactionStatus` + `PlatformTransactionManager`. Propagation quyết định có dùng lại `Connection` của transaction cha không.

## 4) Ví dụ code đơn giản

```java
@Service
class OrderService {
    // Transaction cha: REQUIRED
    @Transactional
    public void placeOrder() {
        // ... write order
        inventoryService.reserve(); // gọi method có REQUIRES_NEW
        // nếu reserve() rollback và exception được xử lý, placeOrder() vẫn có thể commit
    }
}

@Service
class InventoryService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void reserve() {
        // transaction riêng
        throw new RuntimeException("hết hàng");
        // rollback riêng, không ảnh hưởng OrderService
    }
}
```

## 5) Trả lời Middle+

Propagation điều khiển việc có dùng lại transaction hiện tại hay tạo mới. Dùng `REQUIRED` cho các method cùng logic nên cùng commit/rollback. Dùng `REQUIRES_NEW` cho tác vụ độc lập như audit log. Phải xử lý exception nếu không muốn lỗi truyền về transaction cha.

## 6) Trả lời Senior

- `REQUIRES_NEW` mở connection mới → tốn tài nguyên, transaction con commit/rollback độc lập với transaction cha.
- Hợp với audit log cần được lưu dù transaction cha rollback. Notification thường nên dùng outbox sau commit thay vì gửi trực tiếp trong transaction.
- Cạm bằi: exception propagation, retry complexity, deadlock tiềm năng nếu lock cha/con overlap.
- Theo dõi connection pool, thời gian giữ lock và transaction latency; trace context thường vẫn đi cùng thread nhưng cần xác nhận với async boundaries.

## 7) Follow-up / pitfall

- `@Transactional` trên method `public` của bean khác — gọi nội bộ (self-invocation) bỏ qua AOP proxy → propagation không áp dụng.
- Transaction con có thể commit dù transaction cha rollback; phải chấp nhận tính độc lập dữ liệu này.
- Không dùng `REQUIRES_NEW` chỉ để né rollback; chọn propagation theo ranh giới nhất quán dữ liệu.

## 8) 30 giây tóm tắt miệng

*"Propagation quyết định method gặp nhau trong transaction thế nào. `REQUIRED` ghép chung transaction cha — lỗi runtime ở phần con thường khiến toàn bộ transaction rollback. `REQUIRES_NEW` tách thành transaction riêng — dùng cho audit log/notification để thất bại con không kéo theo cha. Nó cần connection riêng và có thể tăng áp lực connection pool, nên chỉ dùng khi thật sự cần ranh giới commit độc lập."*
