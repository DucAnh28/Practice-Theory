# Spring AOP Self-Invocation Bypass

- **Date:** 2026-10-06
- **Topic:** java-spring
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

- Tại sao gọi method có `@Transactional`, `@Async` hoặc `@Cacheable` từ một method khác trong cùng một class thì annotation lại không hoạt động?
- Cơ chế proxy của Spring AOP hoạt động thế nào và làm sao xử lý triệt để lỗi self-invocation (tự gọi nội bộ)?

## 2) Giải thích như trẻ lên 3

Tưởng tượng cửa ra vào có bác bảo vệ (Proxy) kiểm tra vé và đóng dấu mỗi khi có khách vào.
Nếu người ngoài bước vào qua cửa chính, bác bảo vệ chặn lại kiểm tra vé (`@Transactional`).
Nhưng nếu bạn đã ở sẵn trong nhà rồi đi từ phòng khách sang phòng ngủ (gọi nội bộ trong cùng class), bạn không đi qua cửa chính nên bác bảo vệ hoàn toàn không biết để đóng dấu.

## 3) Giải thích đơn giản cho dev

Spring AOP mặc định dùng **Dynamic Proxy** (CGLIB hoặc JDK Dynamic Proxy).
Khi một bean được inject vào bean khác, Spring không inject instance gốc (`target object`) mà inject một instance bọc ngoài (`proxy object`).
- Khi client bên ngoài gọi method: luồng chạy qua `Proxy` -> thực thi Advice / Interceptor (mở transaction, bắt async, check cache) -> chuyển tiếp tới `target object`.
- Khi `this.methodB()` được gọi từ `methodA()` bên trong cùng instance: lệnh gọi dùng con trỏ `this` trỏ thẳng vào target object gốc, hoàn toàn bỏ qua Proxy. Hệ quả là toàn bộ interceptor bị bypass.

## 4) Ví dụ code đơn giản

```java
@Service
public class OrderService {

    // Method A không có @Transactional
    public void createOrder(OrderRequest request) {
        // ... validate
        // Gọi nội bộ qua `this`: BYPASS PROXY!
        this.saveOrderWithTransaction(request);
    }

    @Transactional
    public void saveOrderWithTransaction(OrderRequest request) {
        // TransactionInterceptor KHÔNG được kích hoạt nếu gọi từ createOrder()
        orderRepository.save(request.toEntity());
    }
}
```

## 5) Trả lời Middle+

- **Nguyên nhân:** Spring AOP dựa trên cơ chế proxy. Các interceptor của `@Transactional`, `@Async`, `@Cacheable` chỉ được kích hoạt khi lời gọi đi qua Proxy. Lời gọi nội bộ (`this.method()`) gọi trực tiếp trên target instance nên bỏ qua proxy.
- **Cách khắc phục phổ biến:**
  1. **Tách class (Khuyên dùng nhất):** Chuyển method có annotation sang một Service/Component riêng rồi inject vào service hiện tại.
  2. **Self-injection:** Inject chính bean đó vào bản thân (dùng `@Lazy` hoặc `ObjectProvider` để tránh circular dependency) rồi gọi qua proxy: `self.saveOrderWithTransaction(...)`.
  3. **AopContext:** Dùng `((OrderService) AopContext.currentProxy()).saveOrderWithTransaction(...)` (yêu cầu bật `@EnableAspectJAutoProxy(exposeProxy = true)`).

## 6) Trả lời Senior

- **Bản chất kiến trúc:** Spring AOP là proxy-based framework, không phải full-blown bytecode manipulation ở runtime như AspectJ.
- **Production impact & rủi ro:**
  - `@Transactional` bị bypass dẫn đến silent data corruption (lỗi không rollback khi throw exception).
  - `@Async` bị bypass dẫn đến blocking main thread ngoài ý muốn, làm nghẽn thread pool của web server / worker.
  - `@Cacheable` bị bypass gây cache miss liên tục, đè tải trực tiếp xuống database.
- **Lựa chọn giải pháp & Trade-offs:**
  - **Refactor tách class:** Luôn là best practice vì tuân thủ Single Responsibility Principle (SRP). Nếu một method cần transaction/async riêng, nó thường thuộc về một transaction boundary hoặc use-case riêng.
  - **AspectJ Compile-time / Load-time Weaving (CTW/LTW):** Nếu dự án có nhu cầu can thiệp sâu (bắt self-invocation, private methods, object khởi tạo bằng `new`), chuyển sang dùng AspectJ thực thụ thay vì Spring Proxy. Trade-off: phức tạp cấu hình build (maven/gradle plugin) hoặc JVM agent (`-javaagent`).

## 7) Follow-up / pitfall

- **Pitfall 1:** Nghĩ rằng đặt `@Transactional` lên `private` method sẽ chạy được. Với Spring AOP proxy, private method không được proxy override/intercept.
- **Pitfall 2:** Dùng `AopContext.currentProxy()` nhưng quên cấu hình `exposeProxy = true` dẫn đến `IllegalStateException`.
- **Pitfall 3:** Dùng `self-injection` mà không có `@Lazy` ở phiên bản Spring cũ có thể gây `BeanCurrentlyInCreationException`.

## 8) 30 giây tóm tắt miệng

Spring AOP dùng proxy bọc ngoài target object. Mọi tính năng như `@Transactional` hay `@Async` chỉ chạy khi request đi xuyên qua proxy từ bên ngoài. Gọi nội bộ qua `this` đi thẳng vào target object nên bị bypass hoàn toàn. Cách fix chuẩn nhất là tách method đó sang một bean riêng để đảm bảo tuân thủ SRP và đi qua proxy bình thường.
