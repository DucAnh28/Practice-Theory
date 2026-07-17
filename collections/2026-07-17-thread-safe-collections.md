# Thread-safe Collections trong Java

- **Date:** 2026-07-17
- **Topic:** collections
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Các collection nào trong Java là thread-safe? Khi nào dùng `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`? Khác gì `Collections.synchronizedList`?

## 2) Giải thích đơn giản

- **Thread-safe** = nhiều thread đọc/ghi cùng lúc vẫn không corrupt data (trong contract của class đó).
- `ArrayList` / `HashMap` **không** thread-safe.
- Legacy: `Vector`, `Hashtable` — sync cả method, an toàn nhưng chậm, ít dùng mới.
- Wrapper: `Collections.synchronizedList/Map` — 1 lock cho cả collection; iterate vẫn phải tự `synchronized`.
- `java.util.concurrent.*` — thiết kế cho multi-thread, thường mịn hơn (CAS, lock theo bin, snapshot…).

### Bảng chọn nhanh

| Nhu cầu | Chọn |
|---------|------|
| Map multi-thread | `ConcurrentHashMap` |
| Map sorted multi-thread | `ConcurrentSkipListMap` |
| List đọc nhiều, ghi ít | `CopyOnWriteArrayList` |
| Producer / consumer | `LinkedBlockingQueue` / `ArrayBlockingQueue` |
| Set concurrent thường | `ConcurrentHashMap.newKeySet()` |

## 3) Trả lời Middle+

Em sẽ tách 3 nhóm:

1. **Legacy sync:** `Vector`, `Hashtable` — method-level synchronized, dễ hiểu nhưng throughput kém.
2. **Synchronized wrappers:** `Collections.synchronizedXxx` — bọc collection thường; mọi operation qua 1 mutex. Iterator không fail-fast an toàn multi-thread trừ khi lock thủ công khi iterate.
3. **`java.util.concurrent`:**
   - `ConcurrentHashMap`: map mặc định cho multi-thread; Java 8+ dùng CAS + lock theo bin; **không cho null key/value**.
   - `CopyOnWriteArrayList`: mỗi lần write copy mảng mới; iterator là snapshot → hợp listener/config ít đổi.
   - `BlockingQueue` (`LinkedBlockingQueue`, `ArrayBlockingQueue`): `put`/`take` block khi full/empty → pattern producer-consumer.

Em **không** dùng `HashMap` share giữa threads. Cần map concurrent → `ConcurrentHashMap`. Cần list concurrent ghi nhiều → cân nhắc design (thường queue), không mặc định COW.

## 4) Trả lời Senior

Senior không chỉ liệt kê class — nói **consistency model + cost + API trap**:

- **`ConcurrentHashMap`**
  - Không global lock; retrieval thường non-blocking; update per-bin.
  - Aggregate ops (`size`, `isEmpty`) **weakly consistent** — không snapshot toàn map tại 1 thời điểm.
  - Compound actions (`check-then-act`) **không atomic** trừ khi dùng `putIfAbsent`, `compute`, `merge`, `replace`.
  - Null bị cấm một phần vì ambiguous với “absent” trong concurrent read.

- **`CopyOnWriteArrayList`**
  - Write path O(n) copy → chỉ khi write hiếm.
  - Iterator không reflect mutation sau khi tạo; không `ConcurrentModificationException` theo nghĩa fail-fast.
  - Dùng sai (write dày) → GC pressure + latency spike.

- **Synchronized wrapper vs concurrent**
  - Wrapper = coarse lock, dễ starve, iterate phải lock đúng monitor.
  - Concurrent collections = scalability tốt hơn, nhưng semantics khác (weakly consistent iteration).

- **Blocking queues**
  - Bounded queue = backpressure.
  - Unbounded `LinkedBlockingQueue` default capacity `Integer.MAX_VALUE` → risk OOM nếu producer nhanh hơn consumer.

- **Thiết kế**
  - Ưu tiên **immutability / thread confinement / message passing** trước shared mutable collection.
  - Shared state cần invariant rõ; collection thread-safe **không** thay cho business-level atomicity.

## 5) Follow-up / pitfall

- Vì sao `ConcurrentHashMap` không cho `null`?
- `synchronizedList` iterate thiếu lock → có thể thấy structural corruption / CME?
- `HashMap` resize multi-thread (đặc biệt pre-Java 8) có thể loop — lesson gì?
- Khi nào `ConcurrentSkipListMap` hơn CHM?
- `computeIfAbsent` có thể reentrant deadlock không (version-dependent) — biết để tránh heavy work trong mapping function.

## 6) 30 giây tóm tắt miệng

> “Collection thường không thread-safe. Multi-thread map dùng ConcurrentHashMap, nhớ compound action phải atomic API. List đọc nhiều ghi ít mới dùng CopyOnWrite. Producer-consumer dùng BlockingQueue có bound để backpressure. Synchronized wrapper chỉ là coarse lock, không phải concurrent cao cấp.”
