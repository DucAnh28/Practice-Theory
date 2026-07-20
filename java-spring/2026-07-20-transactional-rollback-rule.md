# Transactional Rollback Forbidden Rules trong Spring - **Date:** 2026-07-20
- **Topic:** java-spring
- **Level target:** Middle+ / Senior ## 1) Câu hỏi phỏng vấn Spring `@Transactional` khi nào KHÔNG rollback? Sai gì thường gặp với rollbackFor và checked exception? ## 2) Giải thích như trẻ lên 3 Mẹ bảo "nếu làm vỡ bao bánh thì đổ hết giỏ". Spring cũng vậy khi có lỗi thì rollback. Nhưng nếu mẹ nói "lỗi nhỏ không sao" (`catch` hoặc `@Transactional(noRollbackFor=...)`) thì mẹ giữ lại giỏ, không đổ. Checked exception (như `IOException`) thì mẹ không nghe lời rollback cũ, trừ khi bảo rõ `rollbackFor`. ## 3) Giải thích đơn giản cho dev - Mặc định: `RuntimeException` + `Error` → rollback.
- Checked exception → **KHÔNG** rollback trừ khi khai báo `rollbackFor`.
- Có thể mark `/noRollbackFor` để giữ dữ liệu dù có lỗi.
- `@Transactional` chỉ work khi gọi từ bên ngoài bean. Gọi tự method trong cùng class sẽ không có proxy → không rollback. ## 4) Ví dụ code đơn giản ```java
@Service
public class OrderService {

    // Case 1: mặc định rollback
    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order); // nếu dòng sau ném RuntimeException → rollback
        inventoryRepo.deduct(order.getSku(), order.getQty());
    }

    // Case 2: bắt checked exception, tự bắt nên transaction đã commit trước khi throw
    @Transactional
    public void upload(File f) throws IOException {
        fileRepo.save(f);
        // IOException là checked; không rollback trừ khi khai báo rollbackFor
        Files.copy(f.toPath(), target); // nếu fail → lưu DB vẫn còn
    }

    // Fix checked
    @Transactional(rollbackFor = IOException.class)
    public void uploadFixed(File f) throws IOException { ... }

    // Case 3: tự catch RuntimeException sẽ ngăn rollback
    @Transactional
    public void bad() {
        try {
            doWork();
        } catch (RuntimeException e) {
            // log; không rethrow → framework thấy "ok", sẽ commit
        }
    }

    // Case 4: proxy fail - gọi method A từ B trong cùng bean
    @Transactional
    public void A() { B(); } // B() chạy trong tx của A

    @Transactional
    public void B() { throw new RuntimeException("boom"); } // vẫn rollback vì cùng tx

    // Để B có tx riêng (có thểm noRollbackFor):
    @Transactional
    public void A2() { /* tx1 */ }
    @Transactional(noRollbackFor = BizException.class)
    public void B2() { /* tx2 riêng */ } // cần gọi B2 qua proxy khác (autowire self) ``` ## 5) Trả lời Middle+ - `@Transactional` rollback mặc định cho `RuntimeException` và `Error`.
- Checked exception (`Exception` không phải `RuntimeException`) mặc định **không** rollback. Dùng `rollbackFor` để bật.
- Không rollback khi đã `catch` hết exception trong method mà không rethrow.
- `@Transactional` dựa trên proxy; gọi nội bộ cùng bean không tạo proxy mới. ## 6) Trả lời Senior - Điểm mount TX: AOP proxy. Gọi cross-bean qua DI mới qua proxy; self-invocation là phổ biến gây bug.
- Chiến lược: chỉ `rollbackFor` checked exception có business ý nghĩa (ví dụ `InsufficientStockException`). Tránh `rollbackFor = Exception.class` trừ khi cần, vì `@Transactional` thường dùng cho fail-fast app.
- `@Transactional(propagation = REQUIRES_NEW)` + `noRollbackFor` dùng cho "fire-and-forget" audit log, nhưng cần xử lý nếu outer rollback mà outer commit.
- Đọc isolation + timeout: `readOnly = true` (optimize) + `isolation`/`propagation` chỉ khi cần đảm bảo consistency/PESSIMISTIC_WRITE để tránh serialize rác. ## 7) Follow-up / pitfall - Pitfall: bắt `Exception` rồi ghi log + tiếp tục → lỗi im lặng, DB half-written.
- Pitfall: `@Transactional` trên `private` method không có hiệu lực.
- Pitfall: `@Async` method gọi `@Transactional` nhưng async không share tx → rollback chỉ trong thread chạy method async, không rollback DB nếu caller đã commit. ## 8) 30 giây tóm tắt miệng "Spring rollback theo mặc định RuntimeException và Error. Checked exception chỉ rollback khi khai báo `rollbackFor`. Cẩn thận catch exception trong tx — catch mà không ném lại sẽ block rollback. Ngoài ra `@Transactional` chỉ hoạt động qua proxy; self-invocation không tạo tx mới. Dùng `REQUIRES_NEW` cho audit log nếu muốn giữ log dù rollback phần chính."
