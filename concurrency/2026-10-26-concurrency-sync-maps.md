# Concurrency: Synchronized Maps vs Synchronized Blocks

- **Date:** 2026-10-26
- **Topic:** concurrency
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Làm sao để đồng bộ hóa truy cập vào Map khi nhiều thread đọc/ghi cùng lúc?

## 2) Giải thích như trẻ lên 3

Giống như chia bàn ăn: mỗi người chỉ ăn một phần cùng lúc, tránh việc ai ăn hai món cùng lúc.

## 3) Giải thích đơn giản cho dev

Sử dụng `synchronized` trên phương thức Map để đảm bảo chỉ một thread tại thời điểm nào cũng sửa đổi map. Hoặc dùng `ReentrantLock` để kiểm soát quyền truy cập linh hoạt hơn.

## 4) Ví dụ code đơn giản

```java
// Sử dụng synchronized trên phương thức
public class SafeMap {
    private Map<String, Integer> map = new HashMap<>();
    
    public synchronized void add(String key, int value) {
        map.put(key, value);
    }
    
    public int get(String key) {
        return map.get(key);
    }
}
```

## 5) Trả lời Middle+

`synchronized` trên phương thức đảm bảo atomicity nhưng có thể gây contention. Với high-concurrency, `ReentrantLock` với `tryLock` + timeout tốt hơn để tránh deadlock.

## 6) Trả lời Senior

Trade-off: `synchronized` đơn giản nhưng không hỗ trợ timeout hay interrupt. `ReentrantLock` cho phép `tryLock()` để detect contention, và có `fairness` để điều chỉnh order lock. Cần tránh deadlock khi nhiều lock được sử dụng cùng lúc (quy tắc: luôn lấy lock theo thứ tự cố định).

## 7) Follow-up / pitfall

**Deadlock:** Nếu thread A giữ lock1 và muốn lock2, trong khi thread B giữ lock2 và muốn lock1 → deadlock. Giải pháp: luôn lấy lock theo thứ tự cố định (ví dụ: lock theo key order).
