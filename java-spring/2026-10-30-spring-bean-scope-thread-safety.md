# Spring Bean Scope và Thread Safety

- **Date:** 2026-10-30
- **Topic:** java-spring
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> Spring Bean mặc định có scope nào? Khi nào cần thiết kế bean prototype để đảm bảo thread safety? Làm sao phân biệt prototype vs singleton trong context đa luồng?

## 2) Giải thích như trẻ lên 3

Giống như cô giáo lớp học: singleton như một phòng thiệt vì mọi người cùng dùng một bộ đồ, prototype như mỗi người một bộ đồ riêng. Nếu đồ ấy "bị xệ" khi nhiều người dùng chung, thì cần mỗi người một bộ!

## 3) Giải thích đơn giản cho dev

- **Singleton Bean (mặc định):** Spring tạo một instance duy nhất, chia sẻ cho tất cả request. Đơn giản nhưng **KHÔNG thread-safe** nếu bean chứa state mutable.
- **Prototype Bean:** Mỗi lần get/create đều tạo mới instance. Thread-safe vì mỗi thread có instance riêng, nhưng tốn tài nguyên hơn.
- **Thread Safety:** Singleton bean phải **immutable** hoặc **stateless**. Nếu có state (như cache, counter), dùng prototype hoặc thread-safe structures (ConcurrentHashMap).

## 4) Ví dụ code đơn giản

```java
@Component
@Scope("singleton") // mặc định
public class CounterService {
    private int count = 0; // STATE MUTABLE - KHÔNG thread-safe!
    public int increment() { return ++count; }
}

@Component
@Scope("prototype")
public class UserSession {
    private String userId;
    private LocalDateTime loginTime;
    // Mỗi session là một instance riêng - thread-safe!
}
```

## 5) Trả lời Middle+

Spring singleton bean mặc định, nhưng chỉ **thread-safe khi stateless**. Nếu bean có state mutable, phải:
1. Dùng `@Scope("prototype")` và Spring tự quản lý lifecycle
2. Hoặc dùng `synchronized`/`AtomicInteger` cho state
3. Hoặc thiết kế stateless hoàn toàn

Với Web MVC: singleton service thường dùng được vì request xử lý stateless, nhưng cẩn thận với state trong service hoặc dependency.

## 6) Trả lời Senior

**Production trade-offs:**
- Singleton giúp connection pooling hiệu quả (JDBC, Redis), nhưng nếu bean chứa state mutable như `Map`, `List` → race condition.
- Prototype gây **bean instantiation overhead**, nhưng giúp isolate state per request/context. Cần `ScopedProxy` để inject prototype bean vào singleton.
- **Request scope** (`@RequestScope`) kết hợp với prototype-like cho web request, giống prototype nhưng giảm memory leak.

**Pattern:** Dùng `@Primary` + interface để bind multiple implementations, và `@Lazy` để break circular dependency. Khi dùng `ApplicationContext.getObject()` để get prototype bean.

## 7) Follow-up / pitfall

- **Pitfall 1:** Chia sẻ `Map` hoặc `List` trong singleton service → race condition. Dùng `ConcurrentHashMap` hoặc `@Scope("prototype")`.
- **Pitfall 2:** `@Autowired` prototype bean vào singleton → chỉ inject lúc khởi tạo, không theo từng request. Dùng `ObjectProvider<T>` hoặc `Lookup Method Injection`.
- **Pitfall 3:** Forget `@PostConstruct` gọi nhiều lần với prototype → mỗi instance đều gọi.

## 8) 30 giây tóm tắt miệng

> "Spring Bean singleton mặc định nhưng KHÔNG tự động thread-safe. Nếu bean có state mutable cần dùng @Scope('prototype'), thiết kế stateless hoặc synchronized. Trong production, thường dùng singleton cho service stateless, prototype cho request-scoped objects như UserSession. Lưu ý inject prototype vào singleton dùng ObjectProvider hoặc Lookup Method Injection."