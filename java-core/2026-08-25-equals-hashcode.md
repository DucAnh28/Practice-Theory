# Hợp đồng equals() và hashCode()

- **Date:** 2026-08-25
- **Topic:** java-core
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Vì sao khi override `equals()` thường phải override cả `hashCode()`? Điều gì xảy ra trong `HashMap` hoặc `HashSet` nếu vi phạm hợp đồng này?

## 2) Giải thích như trẻ lên 3

Hãy coi `hashCode()` như số ngăn tủ, còn `equals()` là bước kiểm tra hai món đồ có thật sự giống nhau không.

Hai món được xem là giống nhau phải vào cùng ngăn. Nếu chúng vào hai ngăn khác nhau, người tìm sẽ không biết chúng là cùng một món.

## 3) Giải thích đơn giản cho dev

`HashMap` và `HashSet` dùng `hashCode()` để chọn vùng lưu trữ (bucket), sau đó dùng `equals()` để phân biệt các phần tử trong vùng đó.

Hợp đồng quan trọng:

- Nếu `a.equals(b)` là `true`, `a.hashCode()` phải bằng `b.hashCode()`.
- Hai đối tượng có cùng hash code chưa chắc bằng nhau; va chạm (collision) là hợp lệ.
- Các trường tham gia `equals()` nên đồng nhất với các trường tham gia `hashCode()`.

## 4) Ví dụ code đơn giản

```java
import java.util.Objects;

final class Product {
    private final String id;

    Product(String id) {
        this.id = Objects.requireNonNull(id);
    }

    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof Product product)) return false;
        return id.equals(product.id);
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

## 5) Trả lời Middle+

Khi định nghĩa bằng nhau theo logic nghiệp vụ, tôi override cả `equals()` và `hashCode()` bằng cùng tập trường. Nếu chỉ override `equals()`, hai object bằng nhau có thể có hash code khác nhau, khiến `HashSet` giữ phần tử trùng hoặc `HashMap` không tìm thấy value bằng một key tương đương.

## 6) Trả lời Senior

Ngoài hợp đồng trên, tôi ưu tiên khóa bất biến. Nếu trường dùng để tính hash thay đổi sau khi object đã được thêm vào `HashMap` hoặc `HashSet`, object có thể nằm sai bucket và không còn được tìm thấy hoặc xóa đúng cách.

Tôi cũng cân nhắc quan hệ kế thừa: dùng `getClass()` cho equality chỉ giữa đúng cùng class; dùng `instanceof` cần bảo đảm tính đối xứng khi subclass mở rộng trạng thái. Với value object, `record` thường giảm lỗi vì Java sinh `equals()` và `hashCode()` nhất quán.

## 7) Follow-up / pitfall

- Cùng hash code không đồng nghĩa `equals()` phải là `true`.
- Không dùng trường mutable làm identity của key.
- Kiểm tra đủ tính phản xạ, đối xứng, bắc cầu, nhất quán và xử lý `null` của `equals()`.
- Không đưa trường vào `equals()` nhưng bỏ trường đó khỏi `hashCode()`.

## 8) 30 giây tóm tắt miệng

`HashMap` và `HashSet` chọn bucket bằng `hashCode()`, rồi so sánh bằng `equals()`. Vì vậy, hai object bằng nhau bắt buộc có cùng hash code; chiều ngược lại không bắt buộc. Tôi dùng cùng các trường bất biến cho cả hai phương thức và cẩn thận với kế thừa để tránh phá tính đối xứng.