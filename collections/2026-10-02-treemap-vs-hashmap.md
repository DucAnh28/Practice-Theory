# TreeMap vs HashMap: khi nào cần thứ tự?

- **Date:** 2026-10-02
- **Topic:** collections
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

`TreeMap` và `HashMap` khác nhau thế nào? Khi nào chọn `TreeMap` thay vì `HashMap`?

## 2) Giải thích như trẻ lên 3

`HashMap` giống túi xáo trộn: bỏ vào nhanh, lấy ra nhanh, nhưng không biết thứ tự. `TreeMap` giống hàng sách xếp theo tên: lấy theo tên vẫn nhanh, và luôn biết quyển đứng trước, sau, hay ở giữa khoảng nào.

## 3) Giải thích đơn giản cho dev

| Đặc điểm | `HashMap` | `TreeMap` |
|---|---|---|
| Cấu trúc nội bộ | Hash table (mảng + linked list / red-black tree bucket) | Red-black tree |
| Thứ tự key | Không đảm bảo | Sorted theo `Comparable` / `Comparator` |
| `get` / `put` / `remove` | O(1) trung bình | O(log n) |
| `null` key | Cho phép 1 null key | **Không** cho phép null key |
| Navigable ops | Không có | `floorKey`, `ceilingKey`, `subMap`, `headMap`, `tailMap`... |
| Memory | Ít hơn | Mỗi node 3 con trỏ (left, right, parent) + màu |

Dùng `TreeMap` khi cần duyệt key theo thứ tự hoặc truy vấn theo khoảng.

## 4) Ví dụ code đơn giản

```java
// Leaderboard: score → username, muốn lấy top/bottom dễ dàng
TreeMap<Integer, String> leaderboard = new TreeMap<>();
leaderboard.put(100, "An");
leaderboard.put(250, "Bình");
leaderboard.put(175, "Cúc");

// key lớn nhất
System.out.println(leaderboard.lastKey());       // 250

// những người có score từ 100 đến 200
System.out.println(leaderboard.subMap(100, true, 200, true));
// {100=An, 175=Cúc}

// key gần nhất <= 180
System.out.println(leaderboard.floorKey(180));   // 175
```

Cùng use case với `HashMap` sẽ phải `sort()` thủ công mỗi lần truy vấn.

## 5) Trả lời Middle+

Tôi dùng `HashMap` mặc định vì O(1) và ít overhead. Chuyển sang `TreeMap` khi cần key có thứ tự — ví dụ duyệt theo range, tìm phần tử gần nhất (`floorKey`/`ceilingKey`), hoặc build timeline/leaderboard. `TreeMap` implements `NavigableMap` nên có sẵn các thao tác khoảng mà `HashMap` không có. Đánh đổi là mọi thao tác đều O(log n) thay vì O(1).

## 6) Trả lời Senior

Chọn `TreeMap` hay `HashMap` phụ thuộc access pattern, không phải "sự an toàn":

- **Khi nào `TreeMap` thực sự cần:** scheduler (key = timestamp), sliding window aggregation, rate limiter dạng token bucket theo time range, prefix/range query. Nếu chỉ cần duyệt key sorted một lần, `HashMap` + `sort` rẻ hơn.
- **Comparator:** `TreeMap` xác định equality qua `compareTo`/`Comparator`, **không** qua `equals`. Nếu `compareTo` trả về 0 nhưng `equals` trả về false, key bị coi là trùng — đây là trap phổ biến với custom `Comparator`.
- **Thread safety:** cả hai đều không thread-safe. Concurrent sorted map → `ConcurrentSkipListMap` (lock-free, performance tốt hơn synchronized `TreeMap`).
- **Memory:** mỗi node `TreeMap` giữ thêm 3 con trỏ + bit màu; với map lớn (triệu entry), `TreeMap` tốn đáng kể hơn `HashMap`.
- **Iteration:** `TreeMap` cho phép fail-fast iterator theo thứ tự; `HashMap` không đảm bảo thứ tự và thứ tự có thể thay đổi sau resize.

## 7) Follow-up / pitfall

- Dùng `Comparator` trả về 0 cho hai key "khác" → key thứ hai bị ghi đè thầm lặng.
- Đưa null key vào `TreeMap` → `NullPointerException` ngay khi insert (khác `HashMap`).
- Dùng `TreeMap` mà chỉ cần lookup đơn giản → chấp nhận O(log n) không cần thiết.
- Quên `ConcurrentSkipListMap` khi cần concurrent sorted map, thay vào đó wrap `TreeMap` với `synchronizedMap` → coarse lock, bottleneck.
- `subMap` / `headMap` / `tailMap` trả về **view** (không copy) → sửa view ảnh hưởng map gốc.

## 8) 30 giây tóm tắt miệng

`HashMap` mặc định: O(1), không quan tâm thứ tự. `TreeMap` khi cần key sorted hoặc range query: O(log n), có `floorKey`/`ceilingKey`/`subMap`. Lưu ý `TreeMap` so sánh qua `compareTo` chứ không qua `equals`, không nhận null key, và concurrent thì dùng `ConcurrentSkipListMap` thay vì wrap `TreeMap`.
