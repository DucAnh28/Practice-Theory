# Record — data carrier bất biến trong Java

- **Date:** 2026-11-16
- **Topic:** java-core
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Java `record` khác gì so với class bình thường? Khi nào nên dùng record, khi nào không? Nó có đi kèm ràng buộc gì về inheritance, thread-safety hay việc map với JSON/JPA?

---

## 2) Giải thích như trẻ lên 3

Record giống một hộp bưu kiện: bạn đưa vào mấy thông tin lúc tạo hộp, và mấy thông tin đó không đổi được sau này. Java tự sinh giúp bạn phần "đóng hộp" (constructor, getter, equals, hashCode, toString) nên bạn không phải viết tay.

---

## 3) Giải thích đơn giản cho dev

- `record` là **class final** (implicitly `final`, không `extends` được) dùng để gom dữ liệu; vẫn `implements` interface được.
- Mỗi thành phần khai báo trong body gọi là **component**; JVM tự sinh:
  - `private final` field,
  - canonical constructor (public) — có thể ghi đè body để validate,
  - accessor **không phải** `getX()` mà là `x()` (tức là `name()` chứ không `getName()`),
  - `equals()`, `hashCode()`, `toString()` theo **component**.
- Không có setter; field là `final` ⇒ object **immutable** ⇒ an toàn chia sẻ giữa nhiều thread.
- `static` members, nested types, `static` initializer block, `instanceof` pattern, `switch` pattern vẫn dùng được (instance initializer thì không).
- Có thể implement interface; **không** có thể extend class khác (trừ `java.lang.Record`).
- Compact constructor dùng để chuẩn hoá/validate dữ liệu ngay khi khởi tạo.

---

## 4) Ví dụ code đơn giản

```java
public record Money(BigDecimal amount, String currency) {

    // Compact constructor: chuẩn hoá + validate
    public Money {
        if (amount == null || currency == null) {
            throw new IllegalArgumentException("amount/currency bắt buộc");
        }
        if (amount.scale() > 2) {
            throw new IllegalArgumentException("scale tối đa 2");
        }
        if (currency.isBlank()) {
            throw new IllegalArgumentException("currency bắt buộc");
        }
        currency = currency.toUpperCase(Locale.ROOT);
    }

    // Overload constructor tiện lợi
    public Money(BigDecimal amount) {
        this(amount, "VND");
    }

    public Money plus(Money other) {
        // kiểm tra cùng currency — business rule quan trọng
        if (!this.currency.equals(other.currency)) {
            throw new IllegalStateException("Không cộng khác loại tiền tệ");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

Money m = new Money(new BigDecimal("100.00"), "vnd");
System.out.println(m.amount());      // 100.00
System.out.println(m.currency());    // VND  (đã chuẩn hoá)
System.out.println(m);               // Money[amount=100.00, currency=VND]

// equals/hashCode do record sinh ra:
System.out.println(m.equals(new Money(new BigDecimal("100.00"), "VND"))); // true
```

```java
// Pattern matching (Java 16+) — dùng rất tự nhiên với record
static String describe(Object o) {
    return switch (o) {
        case Money m when m.amount().signum() > 0 -> "Tiền: " + m.amount() + " " + m.currency();
        case Money m                            -> "Số tiền 0";
        case null                              -> "null";
        default                               -> "Khác";
    };
}
```

---

## 5) Trả lời Middle+

> `record` là kiểu dữ liệu bất biến, compiler sinh sẵn field `final`, canonical constructor, accessor, `equals`, `hashCode`, `toString`. Tôi dùng nó cho DTO/response/value object nơi dữ liệu chỉ đọc, không cần inheritance hay setter. Vì field là `final` nên object an toàn khi share giữa các thread. Lưu ý accessor là `x()` chứ không phải `getX()`, và phải dùng `equals`/`hashCode` do record cung cấp khi đưa vào `HashMap`/`HashSet` hoặc so sánh trong test.

---

## 6) Trả lời Senior

> **Trade-off:** record tiết kiệm code nhưng không phải "class bất biến hoàn hảo" — các field dạng mutable bên trong (ví dụ `List` không dùng `List.copyOf`) vẫn có thể bị mutate sau khi khởi tạo, phá vỡ tính bất biến và làm hỏng `hashCode` khi đã dùng làm key trong `HashMap`. Với collection field nên dùng `List.copyOf(...)`/`Map.copyOf(...)` trong compact constructor, hoặc dùng `Set`/`List` bất biến thật sự.
>
> **Jackson:** Jackson đã hỗ trợ record từ 2.12; với `@JsonProperty` trên component, và `@JsonCreator` cho constructor nếu có nhiều constructor. Nếu project dùng Jackson cũ hơn, cần bật `MapperFeature`/`ParameterNamesModule`.
>
> **JPA:** JPA spec **không** hỗ trợ record làm entity (entity cần no-arg constructor + field mutable để Hibernate set giá trị). Record chỉ dùng cho DTO/projection; entity vẫn là class thường. Hibernate 6 hỗ trợ record ở vai trò embeddable/value type, nhưng đừng lệ thuộc vào đó nếu project còn chạy JPA/Hibernate cũ.
>
> **Jib/GraalVM/AOT:** record thường được ưu tiên cho config properties (`@ConfigurationProperties`) vì binding tự sinh, nhưng cần test contract và cẩn thận với nested record + defaults.
>
> **Serialization:** Java native serialization (`ObjectOutputStream`) vẫn hoạt động nhưng record là `final` nên subclass không được — đây là ưu điểm về an toàn. Với Jackson, `equals`/`hashCode` sinh sẵn giúp cache (L1/Ehcache) và dedupe request dễ hơn.

---

## 7) Follow-up / pitfall

- **Accessor không phải `getX()`** — đây là điểm hay bị hỏi: `record User(String name)` → `user.name()`, không có `getName()`. Với Lombok `@Getter` thì lại là `getName()` → khi map DTO giữa hai thế giới dễ lệch.
- **`equals`/`hashCode` theo component** — nếu thêm component mới (ví dụ `createdAt` không ổn định), `equals` sẽ đổi hành vi ⇒ dùng làm key map sẽ hỏng.
- **Compact constructor phải gán lại field**: `currency = currency.toUpperCase();` (gán vào tham số), không phải `this.currency = ...`.
- **Collection mutable trong record** — xem pitfall ở phần Senior.
- **`equals` với `instanceof` hay `getClass()`:** record dùng `instanceof` (đúng cho final class), nên subclassing không phải vấn đề.
- **Câu hỏi xoáy:** "Record có phải là functional interface / có hỗ trợ inheritance không?" → Không, record là final, không có từ khoá `extends`/`implements` ngoài interface.
- **Câu hỏi xoáy:** "Dùng record cho entity JPA được không?" → Không, record không có no-arg constructor và field không mutable ⇒ JPA không map được.

---

## 8) 30 giây tóm tắt miệng

> "Record là class bất biến trong Java 16+, compiler tự sinh field final, canonical constructor, accessor kiểu `x()`, cùng `equals`/`hashCode`/`toString` theo component. Tôi dùng cho DTO/value object để giảm boilerplate và có thread-safety miễn phí. Vài pitfall: accessor không phải `getX()`, collection field phải copy để thật sự bất biến, và record không dùng được làm entity JPA vì thiếu no-arg constructor và field mutable."
