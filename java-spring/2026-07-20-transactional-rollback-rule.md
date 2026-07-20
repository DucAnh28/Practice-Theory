# Transactional — Khi nào KHÔNG rollback?

- **Date:** 2026-07-20
- **Topic:** java-spring
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Spring `@Transactional` khi nào **KHÔNG** rollback? Sai gì thường gặp với `rollbackFor` và checked exception?

## 2) Giải thích đơn giản

Tưởng tượng bạn mở giao dịch chuyển tiền. Một lỗi nhỏ (quên ghi sổ, sơ suất giấy tờ) thì "tự động hoàn tiền" — đó là rollback tự động của Spring. Nhưng Spring chỉ tự rollback với những lỗi nó coi là "nghiêm trọng" (`RuntimeException`, `Error`). Còn các lỗi "checked" (`IOException`, `SQLException`…) — Spring cảnh báo nhưng không tự rollback, trừ khi bạn nói rõ `rollbackFor`.

Nếu trong method bạn tự `catch` ngoại lệ rồi xử lý, Spring coi "tốt, xong việc" và **commit** luôn. Bạn muốn rollback thì phải rethrow hoặc gọi `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

Ngoài ra `@Transactional` hoạt động qua proxy (AOP). Gọi method B từ A trong cùng class → proxy không được kích hoạt lần 2.

### Bảng chọn nhanh

| Trường hợp | Rollback? |
|-----------|-----------|
| `RuntimeException` (vd `NullPointerException`) | ✅ Có |
| `Error` (vd `OutOfMemoryError`) | ✅ Có |
| Checked exception (`IOException`, `SQLException`…) | ❌ Không — trừ khi `rollbackFor` |
| Tự `catch` + không rethrow | ❌ Không |
| Gọi `@Transactional` trong cùng bean | ❌ Không — proxy không vào |

## 3) Trả lời Middle+

- `@Transactional` rollback mặc định cho `RuntimeException` và `Error`.
- Checked exception **không** rollback trừ khi khai `rollbackFor`.
- Tự `catch` trong method mà không rethrow là block rollback.
- `@Transactional` dựa proxy **(chuyển hướng qua AOP)**; self-invocation (gọi nội bộ cùng bean) không qua proxy. Muốn tách transaction riêng: autowire self.

## 4) Trả lời Senior

- Điểm mấu chốt: transaction boundary do AOP proxy quyết định. Code bên trong không biết hoặc tự quyết được.
- Chiến lược `rollbackFor`: chỉ rollback cho checked exception có ý nghĩa business (vd `InsufficientFundsException`). Tránh `rollbackFor = Exception.class` trừ khi rất cần.
- Pitfall tổ chức code:
  - `@Transactional` trên `private` method = vô hiệu.
  - Import sai `@Transactional` (Spring vs JTA).
  - Mở transaction xong gọi `@Async` method — async không share transaction cũ.
- `@Transactional(propagation = REQUIRES_NEW)` + `noRollbackFor`: dùng cho audit log "fire-and-forget" không bị rollback theo chính.
- Luôn nhớ: transaction là unit of work nhất quán về mặt nghiệp vụ, không phải UI/physical guard.

## Ví dụ code

```java
@Service
public class OrderService {

    // ✅ Mặc định rollback
    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order);
        inventoryRepo.deduct(order.getSku(), order.getQty());
        // ném RuntimeException ở dòng nào → rollback dòng đó
    }

    // ❌ Checked exception — KHÔNG rollback
    @Transactional
    public void upload(File f) throws IOException {
        fileRepo.save(f); // đã commit riêng
        Files.copy(f.toPath(), target); // IOException không rollback dòng trên
    }

    // ✅ Fix: rollbackFor
    @Transactional(rollbackFor = IOException.class)
    public void uploadFixed(File f) throws IOException { ... }

    // ❌ Tự catch rồi không rethrow
    @Transactional
    public void bad() {
        try {
            doWork();
        } catch (RuntimeException e) {
            log.warn("lỗi nhỏ", e);
            // Không rethrow → commit
        }
    }
}
```

## Follow-up / pitfall

- **Self-invocation:** Gọi `this.methodB()` từ `this.methodA()` trong cùng bean → methodB không có transaction riêng.
- **Check mà catch sai layer:** Catch `Exception` ở controller/scheduled task trigger rồi ghi log + tiếp tục → DB half-written.
- **`@Transactional` trên `private` method:** Vô hiệu, proxy không vào được.
- **Async + Transactional:** `@Async` method chạy thread riêng → không share transaction của caller.

## 5) 30 giây tóm tắt miệng

"Spring rollback mặc định với RuntimeException và Error, còn checked exception phải khai báo rollbackFor. Catch bên trong mà không rethrow là commit luôn. `@Transactional` hoạt động qua proxy, self-invocation không tạo transaction mới. Dùng `REQUIRES_NEW` cho audit log nếu muốn giữ dù rollback phần chính."