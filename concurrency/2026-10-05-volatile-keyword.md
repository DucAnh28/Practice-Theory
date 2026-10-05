# Từ khóa Volatile trong Java

- **Date:** 2026-10-05
- **Topic:** concurrency
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Giải thích từ khóa `volatile` trong Java. Nó giải quyết vấn đề gì? Khi nào dùng nó thay vì `synchronized`?

## 2) Giải thích như trẻ lên 3

Tưởng tượng anh chị em sống trong 1 ngôi nhà. Em có 1 chiếc bảng ghi nhớ trên tường (variable). Nếu không có quy tắc gì, mỗi người có thể nhớ lại trong đầu giá trị cũ của bảng và không liên tục nhìn bảng thật. `volatile` buộc mọi người phải nhìn bảng thật mỗi lần, không dùng bộ nhớ trong đầu riêng.

## 3) Giải thích đơn giản cho dev

`volatile` là modifier cho biến. Nó đảm bảo:
- **Visibility:** Khi thread này write, tất cả thread khác sẽ thấy giá trị mới ngay.
- **Ordering:** Không có reorder instruction xung quanh read/write của biến `volatile`.

Nó **không** đảm bảo atomicity. `volatile long x; x++;` vẫn race condition.

## 4) Ví dụ code đơn giản

```java
// Không volatile — thread có thể cache giá trị cũ
public class NoVolatile {
    private boolean running = true;
    
    public void stopLoop() {
        running = false;  // thread chính đổi
    }
    
    public void loop() {
        while (running) {  // worker thread có thể không thấy false
            // ...
        }
    }
}

// Volatile — thread sẽ thấy ngay
public class WithVolatile {
    private volatile boolean running = true;
    
    public void stopLoop() {
        running = false;
    }
    
    public void loop() {
        while (running) {  // worker thread sẽ thấy false ngay
            // ...
        }
    }
}
```

## 5) Trả lời Middle+

`volatile` dùng khi biến được share giữa nhiều thread mà chỉ cần đọc/ghi đơn giản, không cần toàn bộ synchronized block. Ví dụ: flag dừng thread, config được load lại, status field. Không dùng nó cho counter hay field phức tạp. Muốn compound operation (đọc-rồi-sửa-rồi-ghi)? Dùng `AtomicInteger`, `synchronized` hay lock.

## 6) Trả lời Senior

`volatile` cấp memory barrier (acquire/release semantics tùy CPU), không full mutual exclusion. Tốc độ nhanh hơn `synchronized` vì không có lock overhead. Nhưng:
- Không ngăn được concurrent modify (ví dụ `x++`).
- Chỉ đảm bảo visibility + ordering của biến đó, không state khác.
- Nếu cần đảm bảo consistent snapshot của nhiều biến, vẫn cần synchronized/lock.

Double-checked locking dùng `volatile` để tránh lock overhead sau lần đầu init:

```java
class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {  // check 1 (fast path, no lock)
            synchronized (Singleton.class) {
                if (instance == null) {  // check 2 (đảm bảo thread-safe)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

`volatile` trên `instance` buộc thread khác thấy initialization hoàn toàn (happens-before).

## 7) Follow-up / pitfall

**Q:** `volatile` có tương đương với `final`?  
**A:** Không. `final` không thay đổi. `volatile` cho phép thay đổi nhưng visibility. Combination `volatile final` vô nghĩa.

**Q:** Tại sao `volatile long x = 0; x++;` không an toàn?  
**A:** `x++` là 3 bước: load → increment → store. Giữa load và store, thread khác có thể write. `volatile` chỉ đảm bảo từng bước thấy giá trị mới, không atomic toàn bộ.

**Pitfall:** Lạm dụng `volatile` thay vì `AtomicInteger`. Rồi mới đến tìm race condition khó.

## 8) 30 giây tóm tắt miệng

"Volatile là từ khóa đảm bảo visibility—khi thread này ghi, thread kia thấy ngay. Nó không đảm bảo atomicity, nên chỉ dùng cho read/write đơn giản, không compound operation. Double-checked locking dùng nó để tránh lock overhead. Nếu cần atomic operation, dùng AtomicInteger hay synchronized thay vì volatile + compound."
