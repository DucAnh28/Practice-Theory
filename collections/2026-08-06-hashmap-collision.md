# HashMap xử lý va chạm như thế nào?

- **Date:** 2026-08-06
- **Topic:** collections
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Java `HashMap` xử lý thế nào khi nhiều key rơi vào cùng một bucket?

## 2) Giải thích như trẻ lên 3

Hãy tưởng tượng tủ có nhiều ngăn. `hashCode()` chỉ ngăn cần mở. Nếu hai món cùng vào một ngăn, nhãn `equals()` giúp tìm đúng món.

## 3) Giải thích đơn giản cho dev

- **Bucket (ngăn chứa):** vị trí trong mảng nội bộ của `HashMap`.
- `hashCode()` được trộn bit để chọn bucket.
- Các key cùng bucket tạo **collision (va chạm)**.
- `HashMap` so sánh hash trước, rồi dùng `equals()` để nhận diện đúng key.
- Java 8 có thể đổi danh sách liên kết dài thành cây đỏ-đen để giảm thời gian tìm kiếm trong bucket xấu.

| Trường hợp | Chi phí tìm kiếm gần đúng |
|---|---:|
| Phân bố hash tốt | O(1) trung bình |
| Bucket là danh sách dài | O(n) |
| Bucket đã treeify | O(log n) |

## 4) Ví dụ code đơn giản

```java
record Key(int id) {
    @Override
    public int hashCode() {
        return 1; // cố ý tạo collision
    }
}

Map<Key, String> map = new HashMap<>();
map.put(new Key(1), "A");
map.put(new Key(2), "B");

System.out.println(map.get(new Key(2))); // B, nhờ equals()
```

## 5) Trả lời Middle+

`HashMap` dùng hash để chọn bucket. Khi collision, nó duyệt các entry trong cùng bucket và dùng `equals()` để tìm key. Từ Java 8, bucket quá dài có thể chuyển thành cây đỏ-đen. Vì vậy phải cài đặt `equals()` và `hashCode()` đúng hợp đồng, đồng thời không thay đổi các field tham gia tính hash khi key đang nằm trong map.

## 6) Trả lời Senior

Hiệu năng O(1) chỉ là trung bình và phụ thuộc chất lượng hash, capacity, load factor. Treeification chỉ xảy ra khi bucket đủ dài và bảng đủ lớn; nếu bảng còn nhỏ, `HashMap` ưu tiên resize. Trong production cần tránh key mutable, hash phân bố kém và input không tin cậy gây collision hàng loạt. `HashMap` không thread-safe; concurrent access cần `ConcurrentHashMap` hoặc đồng bộ phù hợp.

## 7) Follow-up / pitfall

- Chỉ override `equals()` nhưng không override `hashCode()` làm lookup sai.
- Sửa field của key sau `put()` có thể khiến `get()` không tìm thấy entry.
- Collision không có nghĩa key bị ghi đè; chỉ key `equals()` nhau mới thay value.
- `HashMap` không bảo đảm thứ tự duyệt.

## 8) 30 giây tóm tắt miệng

`HashMap` dùng hash để chọn bucket; nếu nhiều key cùng bucket thì dùng hash và `equals()` để tìm đúng key. Java 8 có thể treeify bucket dài để cải thiện từ O(n) xuống O(log n). O(1) chỉ là trung bình, nên key cần bất biến, `equals()`/`hashCode()` đúng hợp đồng và hash phân bố tốt.
