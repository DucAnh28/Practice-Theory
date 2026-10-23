# ConcurrentHashMap vs Collections.synchronizedMap

- **Date:** 2026-10-23
- **Topic:** collections
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi nào nên dùng `ConcurrentHashMap` thay vì `Collections.synchronizedMap()` trong ứng dụng Java đa luồng? Mechanics khác nhau thế nào?

## 2) Giải thích như trẻ lên 3

- `Collections.synchronizedMap`: Giống như một cái phòng đọc sách chỉ có một chìa khóa. Bất kỳ ai muốn đọc hay viết đều phải cầm chìa khóa đó, người khác phải đứng đợi ngoài cửa.
- `ConcurrentHashMap`: Giống như thư viện chia thành nhiều khu vực (hoặc nhiều tủ sách). Nhiều người có thể xem và lấy sách ở các khu vực khác nhau cùng một lúc mà không chen húp nhau.

## 3) Giải thích đơn giản cho dev

- **`Collections.synchronizedMap()`**: Là wrapper bao ngoài một `Map` thường. Nó dùng cơ chế khóa toàn bộ map (Lock-ordering/Intrinsic lock trên đối tượng wrapper) cho mọi thao tác (`get`, `put`, `remove`). Gây nghẽn (contention) cực lớn khi traffic cao.
- **`ConcurrentHashMap`**: Dùng cơ chế khóa hạt mịn (Fine-grained locking) kết hợp `CAS` (Compare-And-Swap) và `synchronized` ở cấp độ từng bucket/node (từ Java 8 trở đi). Nhờ đó, các thao tác đọc (`get`) không cần khóa (lock-free), còn thao tác ghi (`put`) chỉ khóa bucket chứa key tương ứng.

## 4) Ví dụ code đơn giản

```java
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class MapThreadSafetyDemo {
    public static void main(String[] args) {
        // SynchronizedMap: Khóa toàn bộ Map khi truy cập
        Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>());
        syncMap.put("key1", "val1");

        // ConcurrentHashMap: Thread-safe hiệu năng cao
        Map<String, String> concurrentMap = new ConcurrentHashMap<>();
        concurrentMap.put("key1", "val1");

        // Atomic operation tích hợp sẵn trong ConcurrentHashMap
        concurrentMap.computeIfAbsent("key2", k -> "val2");
    }
}
```

## 5) Trả lời Middle+

- **Đồng bộ hóa (Synchronization)**: `Collections.synchronizedMap()` dùng một lock chung cho toàn bộ map. `ConcurrentHashMap` khóa theo từng bucket (Segment ở Java 7, CAS + synchronized ở Java 8+).
- **Hiệu năng Đọc/Ghi**: `ConcurrentHashMap` hỗ trợ nhiều thread đọc và ghi đồng thời mà không nghẽn. `synchronizedMap` nghẽn khi số lượng thread tăng.
- **Null Values**: `ConcurrentHashMap` không cho phép `null` làm key hoặc value (để tránh mập mờ giữa "key không tồn tại" và "value bằng null" trong môi trường concurrent). `synchronizedMap` cho phép `null` nếu map gốc hỗ trợ.
- **Atomic Operations**: `ConcurrentHashMap` cung cấp các hàm nguyên tử như `putIfAbsent()`, `computeIfPresent()`, `replace()`. Với `synchronizedMap`, phải tự viết block `synchronized(map)` bên ngoài để đảm bảo tính nguyên tử cho chuỗi thao tác check-then-act.

## 6) Trả lời Senior

- **Java 8 Mechanics**: Java 8 bỏ Segment, dùng mảng `Node<K,V>[]` kết hợp CAS cho node đầu tiên của bucket. Nếu bucket đã có phần tử, nó dùng `synchronized` khóa đúng node gốc (head node) của bucket đó.
- **Iterator & Fail-Safe**: `ConcurrentHashMap` trả về *Weakly Consistent Iterator*. Không ném `ConcurrentModificationException` khi map bị sửa đổi trong lúc duyệt, phản ánh trạng thái map tại hoặc sau thời điểm tạo iterator. Trong khi đó, `synchronizedMap` dùng *Fail-Fast Iterator*, phải tự lock bên ngoài khi iterate để tránh lỗi.
- **Size Calculation**: Hàm `size()` của `ConcurrentHashMap` không lock toàn bộ map mà dùng cơ chế cộng dồn biến đếm bất đồng bộ (`CounterCell`), trả về giá trị ước lượng gần đúng nếu có concurrent update.
- **Production Trade-off**: Dùng `ConcurrentHashMap` cho đọc/ghi song song tần suất cao. Tuy nhiên, nếu cần lock toàn bộ map để thực hiện atomic snapshot/clear hoặc duy trì thứ tự khắt khe, `Collections.synchronizedMap` hoặc khóa ngoài mới đáp ứng được.

## 7) Follow-up / pitfall

- **Pitfall**: Nghĩ rằng `ConcurrentHashMap` làm cho chuỗi các thao tác tự động thread-safe.
  - *Sai*: `if (!map.containsKey(k)) map.put(k, v);` vẫn bị race condition!
  - *Sửa*: Dùng `map.putIfAbsent(k, v)` hoặc `map.computeIfAbsent()`.
- **Câu hỏi xoáy**: Tại sao `ConcurrentHashMap` lại cấm `null` key và `null` value?
  - *Trả lời*: Vì trong đa luồng, nếu `map.get(key)` trả về `null`, ta không thể phân biệt key chưa tồn tại hay key có giá trị `null` mà không gọi `containsKey()`. Nhưng gọi `containsKey()` ngay sau `get()` thì map có thể đã bị thay đổi bởi thread khác.

## 8) 30 giây tóm tắt miệng

`Collections.synchronizedMap` khóa toàn bộ map gây nghẽn hiệu năng, trong khi `ConcurrentHashMap` dùng CAS và khóa hạt mịn theo bucket giúp đọc/ghi song song mượt mà. `ConcurrentHashMap` cấm `null`, không ném `ConcurrentModificationException` khi iterate và cung cấp các hàm atomic như `computeIfAbsent`.

---
*ponytail: concise note focusing on key interview mechanics.*
