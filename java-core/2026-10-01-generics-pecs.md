# Generics PECS: `extends` và `super`

- **Date:** 2026-10-01
- **Topic:** java-core
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi nào dùng `? extends T`, khi nào dùng `? super T` trong Java Generics?

## 2) Giải thích như trẻ lên 3

Giỏ cho mình lấy quả ra thì nhận giỏ chứa táo; giỏ để mình bỏ táo vào thì nhận giỏ đựng trái cây nói chung.

## 3) Giải thích đơn giản cho dev

PECS: *Producer Extends, Consumer Super*. `List<? extends Number>` đọc ra `Number` được, nhưng không thêm `Integer` được vì danh sách thực tế có thể là `List<Double>`. `List<? super Integer>` thêm `Integer` được, nhưng đọc ra chỉ chắc chắn là `Object`.

## 4) Ví dụ code đơn giản

```java
import java.util.*;

class Demo {
    static <T> void copy(List<? extends T> source, List<? super T> target) {
        for (T item : source) target.add(item);
    }

    public static void main(String[] args) {
        List<Number> numbers = new ArrayList<>();
        copy(List.of(1, 2), numbers);
        System.out.println(numbers); // [1, 2]
    }
}
```

## 5) Trả lời Middle+

Dùng `extends` cho nguồn chỉ đọc, `super` cho đích ghi. Ví dụ copy từ `List<Integer>` sang `List<Number>` mà vẫn giữ kiểm tra kiểu lúc biên dịch.

## 6) Trả lời Senior

Wildcard giúp API nhận nhiều kiểu danh sách mà không ép caller tạo bản sao hay cast không an toàn. Chọn theo chiều luồng dữ liệu; nếu vừa đọc vừa ghi cùng kiểu cụ thể, dùng tham số kiểu `T` thay vì wildcard. Hàm copy ở trên chỉ thêm vào đích, không xóa dữ liệu sẵn có.

## 7) Follow-up / pitfall

- Vì sao `List<Integer>` không gán trực tiếp cho `List<Number>`? Generic bất biến; nếu gán được, ta có thể thêm `Double` vào danh sách số nguyên.
- Đừng gọi `add(1)` trên `List<? extends Number>`: kiểu phần tử cụ thể chưa biết.

## 8) 30 giây tóm tắt miệng

PECS: nguồn tạo giá trị dùng `extends`, đích nhận giá trị dùng `super`. `extends` đọc an toàn theo kiểu cha; `super` ghi an toàn theo kiểu con. Ví dụ sao chép Integer sang danh sách Number mà không cần cast.
