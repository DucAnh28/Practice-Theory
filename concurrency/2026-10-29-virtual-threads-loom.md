# Virtual Threads (Project Loom) trong Java

- **Date:** 2026-10-29
- **Topic:** concurrency
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> Virtual Threads trong Java 21 là gì? Nó khác gì so với Platform Threads truyền thống và thay đổi mô hình lập trình bất đồng bộ (như CompletableFuture, Reactive) như thế nào?

## 2) Giải thích như trẻ lên 3

Tưởng tượng platform threads truyền thống giống như những chiếc xe tải khổng lồ: chở rất nhiều đồ nhưng tốn tài nguyên, lái xe mệt, đỗ bến nào là chiếm chỗ luôn bến đó. Còn Virtual Threads giống như những chiếc xe máy điện nhỏ xíu: nhẹ tênh, muốn tạo bao nhiêu cũng được, khi nào cần chạy thì nhảy lên đường, lúc dừng chờ đèn đỏ thì nhường xe khác chạy qua.

## 3) Giải thích đơn giản cho dev

- **Platform Thread (OS Thread):** Mapping 1:1 với thread của hệ điều hành. Chi phí tạo lớn (~1MB stack), số lượng giới hạn (vài ngàn).
- **Virtual Thread (User-mode Thread):** Quản lý bởi JVM (scheduler M:N), không gắn cứng 1:1 với OS thread. Chi phí tạo cực kỳ rẻ (vài KB), số lượng lên tới hàng triệu.
- **Blocking behavior:** Khi virtual thread gặp lệnh blocking (I/O, `Thread.sleep`), JVM tự động "unmount" nó khỏi OS thread để OS thread đi chạy virtual thread khác. Khi I/O xong, virtual thread được "mount" trở lại để chạy tiếp.

## 4) Ví dụ code đơn giản

```java
// Tạo hàng triệu Virtual Threads cực kỳ đơn giản
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofMillis(1000)); // Không làm nghẽn OS thread!
            return i;
        });
    });
} // executor.close() đợi tất cả task hoàn thành
```

## 5) Trả lời Middle+

- **Giải quyết bài toán I/O Bound:** Thay vì phải dùng reactive programming (WebFlux, RxJava) phức tạp để xử lý concurrency cao, giờ ta viết code theo mô hình **thread-per-request** truyền thống (blocking style) nhưng hiệu năng tương đương reactive.
- **Không thay đổi cách viết code:** Code dễ đọc, dễ debug bằng stack trace thông thường, dễ dùng chung với các thư viện cũ (JDBC, JPA, HTTP clients đồng bộ).
- **Cách dùng:** Dùng `Thread.ofVirtual().start(runnable)` hoặc `Executors.newVirtualThreadPerTaskExecutor()`.

## 6) Trả lời Senior

- **Khi nào KHÔNG nên dùng Virtual Threads?**
  - **CPU-bound tasks:** Nếu task chỉ tính toán nặng, dùng virtual threads không tăng performance mà còn thêm overhead scheduler (vẫn bị giới hạn bởi số lượng CPU cores).
  - **Pinned threads:** Nếu code có khối `synchronized` hoặc gọi native method/JNI trong lúc blocking, virtual thread sẽ bị "pin" vào OS carrier thread, làm mất khả năng unmount (JVM warning: `jdk.tracePinnedThreads`). Cần thay `synchronized` bằng `ReentrantLock`.
- **Pool connection & Resource limits:** Virtual threads giúp giải quyết bottleneck ở thread count, nhưng các tầng phía sau như Database (PostgreSQL pool), Redis, External APIs vẫn có giới hạn connection. Cần kết hợp connection pooling hợp lý (HikariCP).

## 7) Follow-up / pitfall

- *"Có nên dùng Thread Pool cho Virtual Threads không?"* → Không cần! Thiết kế của virtual threads là tạo mới cho mỗi task (`newVirtualThreadPerTaskExecutor`), không tái sử dụng như platform threads.
- *"Synchronized block có chạy được với virtual threads không?"* → Chạy được, nhưng gây pin thread nếu giữ lock qua điểm blocking I/O. Nên chuyển sang `java.util.concurrent.locks.ReentrantLock`.
- Pitfall: Dùng `ThreadLocal` ồ ạt cho hàng triệu threads dẫn đến tốn memory. Nên thay bằng scoped values (`java.lang.ScopedValue` từ Java 20/21).

## 8) 30 giây tóm tắt miệng

> "Virtual Threads trong Java 21 mang đến mô hình lightweight threads quản lý bởi JVM, cho phép viết code blocking truyền thống với hàng triệu concurrent tasks mà không tốn tài nguyên OS threads. Nó giúp code đơn giản, dễ debug hơn Reactive programming trong các hệ thống I/O-bound, tuy nhiên cần lưu ý tránh dùng cho CPU-bound và tránh pin threads trong khối synchronized."
