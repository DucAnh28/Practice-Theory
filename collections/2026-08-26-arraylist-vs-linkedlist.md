# ArrayList và LinkedList: chọn loại nào?

- **Date:** 2026-08-26
- **Topic:** collections
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

So sánh `ArrayList` và `LinkedList`. Khi nào nên dùng từng loại?

## 2) Giải thích như trẻ lên 3

- `ArrayList` giống dãy hộp có số thứ tự: tìm hộp số 10 rất nhanh, nhưng chèn hộp vào giữa phải dời các hộp sau.
- `LinkedList` giống đoàn tàu: mỗi toa nối với toa trước và sau. Nối thêm toa dễ khi đã đứng đúng chỗ, nhưng muốn tìm toa số 10 phải đi qua từng toa.

## 3) Giải thích đơn giản cho dev

| Thao tác | `ArrayList` | `LinkedList` |
|---|---:|---:|
| Truy cập theo index | O(1) | O(n) |
| Thêm cuối | O(1) trung bình | O(1) |
| Chèn/xóa giữa | O(n) do dịch phần tử | O(1) sau khi có node; tìm node vẫn O(n) |
| Bộ nhớ/cache CPU | Ít hơn, locality tốt | Tốn node và con trỏ, locality kém |

Trong ứng dụng thông thường, `ArrayList` thường là lựa chọn mặc định.

## 4) Ví dụ code đơn giản

```java
List<String> names = new ArrayList<>();
names.add("An");
names.add("Bình");
System.out.println(names.get(1)); // O(1)

Deque<String> queue = new LinkedList<>();
queue.addLast("job-1");
queue.addLast("job-2");
System.out.println(queue.removeFirst());
```

Nếu cần queue/deque, thường ưu tiên `ArrayDeque` hơn `LinkedList` vì ít overhead và cache locality tốt hơn.

## 5) Trả lời Middle+

Tôi chọn `ArrayList` khi cần duyệt tuần tự, truy cập index hoặc phần lớn thao tác là thêm cuối. `LinkedList` chỉ có lợi cho chèn/xóa O(1) khi đã có vị trí qua `ListIterator`; nếu phải tìm theo index thì tổng thể vẫn O(n). Vì vậy, mặc định tôi dùng `ArrayList` và chỉ đổi khi workload đo được cho thấy cần thiết.

## 6) Trả lời Senior

Không nên chọn chỉ dựa vào Big-O. `ArrayList` dùng vùng nhớ liên tục nên cache locality tốt, ít allocation và ít áp lực GC hơn. `LinkedList` tạo một node cho mỗi phần tử, tốn thêm con trỏ và pointer chasing nên thường chậm hơn trong thực tế. Khi kích thước biết trước, đặt initial capacity cho `ArrayList` để giảm resize. Với queue/deque, ưu tiên `ArrayDeque`; với truy cập đồng thời, chọn collection chuyên dụng thay vì tự khóa một `LinkedList`.

## 7) Follow-up / pitfall

- Nói “chèn giữa `LinkedList` luôn O(1)” nhưng bỏ qua chi phí O(n) để tìm node.
- Dùng `LinkedList.get(i)` trong vòng lặp, dễ thành O(n²).
- Nghĩ `ArrayList` thread-safe; thực tế cần đồng bộ hoặc collection phù hợp.

## 8) 30 giây tóm tắt miệng

`ArrayList` là lựa chọn mặc định vì truy cập index O(1), duyệt nhanh và ít overhead. `LinkedList` chỉ chèn/xóa O(1) khi đã có node; tìm vị trí vẫn O(n), lại tốn bộ nhớ và cache locality kém. Với queue/deque tôi thường chọn `ArrayDeque`, còn quyết định tối ưu cuối cùng dựa trên workload và benchmark thực tế.
