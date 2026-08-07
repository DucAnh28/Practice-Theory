# volatile: visibility không đồng nghĩa atomicity

- **Date:** 2026-08-07
- **Topic:** concurrency
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Trong Java, `volatile` giải quyết vấn đề gì? Vì sao `volatile int count` vẫn không làm `count++` an toàn khi nhiều thread cùng chạy?

## 2) Giải thích như trẻ lên 3

Hãy tưởng tượng nhiều người cùng nhìn một bảng điểm:

- `volatile` buộc mọi người đọc điểm mới nhất trên bảng.
- Nhưng thao tác `count++` gồm **đọc → cộng 1 → ghi lại**.
- Hai người có thể cùng đọc `10`, cùng ghi `11`; một lần tăng bị mất.

## 3) Giải thích đơn giản cho dev

`volatile` bảo đảm:

- **Visibility (tính nhìn thấy):** thread đọc thấy giá trị mới nhất đã được thread khác ghi.
- **Ordering (thứ tự):** ngăn một số phép reorder quanh lần đọc/ghi volatile.

`volatile` không biến chuỗi thao tác read-modify-write thành **atomic (nguyên tử, không thể bị chen ngang)**. Vì vậy `count++`, check-then-act và cập nhật phụ thuộc giá trị cũ vẫn có race condition.

| Nhu cầu | Công cụ phù hợp |
|---|---|
| Cờ trạng thái độc lập | `volatile boolean` |
| Bộ đếm cạnh tranh vừa phải | `AtomicInteger` |
| Nhiều biến phải đổi cùng một invariant | `synchronized` hoặc `Lock` |
| Bộ đếm thống kê cạnh tranh cao | `LongAdder` |

## 4) Ví dụ code đơn giản

```java
import java.util.concurrent.atomic.AtomicInteger;

class Counter {
    private final AtomicInteger count = new AtomicInteger();

    void increment() {
        count.incrementAndGet();
    }

    int get() {
        return count.get();
    }
}
```

## 5) Trả lời Middle+

`volatile` bảo đảm visibility và một phần ordering theo Java Memory Model, phù hợp cho cờ như `shutdownRequested`. Nó không bảo đảm atomicity cho thao tác ghép. `count++` có ba bước nên có thể mất update; dùng `AtomicInteger.incrementAndGet()` hoặc khóa nếu cần bảo vệ nhiều trạng thái liên quan.

## 6) Trả lời Senior

Chọn primitive theo invariant, không theo cú pháp. `AtomicInteger` dùng CAS (compare-and-set) nên tránh blocking nhưng có thể retry nhiều khi contention cao; bộ đếm metrics có thể dùng `LongAdder`, đổi lại `sum()` không phải snapshot nguyên tử tuyệt đối. Nếu cập nhật nhiều field phải nhất quán, CAS trên một biến chưa đủ; dùng lock hoặc gom state bất biến vào một `AtomicReference`. Cần đo contention và kiểm tra correctness bằng stress test, không dựa vào test chạy vài lần.

## 7) Follow-up / pitfall

- `volatile` reference chỉ làm reference visible; không tự làm object bên trong thread-safe.
- `AtomicInteger.get()` rồi `set()` dựa trên giá trị đã đọc vẫn có race; dùng API atomic như `compareAndSet()` hoặc `updateAndGet()`.
- Double-checked locking cần field instance là `volatile` để publication đúng.

## 8) 30 giây tóm tắt miệng

`volatile` giúp thread thấy giá trị mới nhất và thiết lập ordering, nhưng không làm thao tác ghép thành atomic. Vì `count++` là đọc, cộng rồi ghi nên vẫn mất update. Tôi dùng volatile cho cờ độc lập, AtomicInteger cho một biến cập nhật nguyên tử, và lock khi nhiều trạng thái phải giữ chung invariant.
