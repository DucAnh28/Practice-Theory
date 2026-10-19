# Property-Based Testing trong Java — Test nhiều trường hợp hơn với ít code hơn

- **Date:** 2026-10-19
- **Topic:** testing
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Property-Based Testing (PBT) khác gì với Example-Based Testing truyền thống? Khi nào nên áp dụng PBT trong dự án Java?

## 2) Giải thích như trẻ lên 3

Example-based testing giống như bạn chọn đúng 3 quả táo đỏ để kiểm tra xem "táo có ngọt không". Còn Property-Based Testing giống như bạn đưa máy tự động ném 1.000 quả táo đủ màu, đủ hình dạng vào máy để kiểm tra tính chất chung: "Tất cả các quả táo có đường > 5% đều phải ngọt".

## 3) Giải thích đơn giản cho dev

- **Example-Based Testing:** Viết `assertEquals(expected, actual)` với các giá trị hardcode cụ thể (`1, 2, "abc"`). Dễ sót edge case.
- **Property-Based Testing:** Định nghĩa **tính chất (property)** luôn đúng với mọi input ngẫu nhiên. Framework (như jqwik trong Java) sẽ tự sinh ra hàng trăm test cases (kể cả edge case như null, boundary values, số âm) để kiểm chứng property đó.

```java
// Ví dụ với jqwik: Đảo ngược 2 lần chuỗi phải bằng chính nó
@Property
boolean reverseTwiceIsOriginal(@ForAll String s) {
    return s.equals(new StringBuilder(s).reverse().reverse().toString());
}
```

## 4) Ví dụ code đơn giản

```java
// Giả sử có hàm tính giá sau discount
public int calculateDiscount(int price, int percent) {
    if (price < 0 || percent < 0 || percent > 100) throw new IllegalArgumentException();
    return price * (100 - percent) / 100;
}

// PBT test với jqwik
@Property
boolean discountNeverExceedsOriginalPrice(
    @ForAll @IntRange(min = 0, max = 100_000) int price,
    @ForAll @IntRange(min = 0, max = 100) int percent
) {
    int result = calculateDiscount(price, percent);
    return result <= price;
}
```

## 5) Trả lời Middle+

- **Khác biệt:** Example-based test kiểm tra input/output cụ thể do lập trình viên nghĩ ra. PBT kiểm tra tính bất biến (invariants) hoặc quy luật chung với dữ liệu sinh tự động.
- **Lợi ích:** 
  - Tự động tìm ra edge case kỳ lạ (boundary values, null, overflow) mà dev hay quên.
  - Giảm số lượng test case cần viết thủ công.
- **Tool phổ biến:** **jqwik** (chuẩn cho JUnit 5), QuickCheck (cũ).

## 6) Trả lời Senior

- **Trade-off:** 
  - Viết property khó hơn viết example, đòi hỏi tư duy trừu tượng hóa logic thành toán học/invariants (ví dụ: `inverse(f(x)) == x`, `sort(x)` luôn có kích thước bằng `x`).
  - Chạy chậm hơn unit test thuần do sinh và kiểm tra nhiều dữ liệu.
- **Shrinking:** Khi test thất bại, framework có tính năng *shrinking* (thu hẹp input về giá trị tối giản gây ra lỗi) để dễ debug.
- **Production Concern:** Thích hợp cho các business logic phức tạp: parser, serializer, thuật toán tính toán tài chính, state machine, quy tắc discount/tax. Không phù hợp cho UI test hoặc CRUD đơn giản.
- **Pitfall:** Viết property quá phức tạp đến mức chính property cũng chứa bug tương tự như code cần test.

## 7) Follow-up / pitfall

- "Làm sao tìm được property cho hàm bình thường?" -> Tìm các tính chất: Idempotence (gọi nhiều lần kết quả không đổi), Inverse (có hàm ngược lại), Round-trip (encode rồi decode ra lại giá trị cũ).
- "PBT có thay thế hoàn toàn unit test không?" -> Không. PBT bổ sung cho unit test đối với các business rules phức tạp, còn unit test vẫn tốt cho Happy Path và Integration wiring.

## 8) 30 giây tóm tắt miệng

Property-Based Testing là phương pháp tự động hóa việc sinh hàng trăm test case dựa trên các quy luật (invariants) của code. Thay vì tự nghĩ ví dụ, ta định nghĩa tính chất luôn đúng, giúp tóm gọn edge case ẩn giấu mà test truyền thống dễ bỏ sót.
