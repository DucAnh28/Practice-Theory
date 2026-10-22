# String Interning và String Pool trong Java

- **Date:** 2026-10-22
- **Topic:** java-core
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

`String a = "hello"` và `String b = new String("hello")` — hai cái này có `==` nhau không? Tại sao? String Pool là gì và nó hoạt động như thế nào trong JVM?

---

## 2) Giải thích như trẻ lên 3

Tưởng tượng có một bảng tên (String Pool) dán trên tường. Khi em viết `"hello"`, Java nhìn vào bảng xem đã có "hello" chưa — nếu có thì dùng lại, không in thêm tờ mới. Nhưng nếu em dùng `new String("hello")`, Java in ra một tờ mới riêng và không dán lên bảng — dù nội dung y hệt.

---

## 3) Giải thích đơn giản cho dev

- **String Pool** (hay *intern pool*) nằm trong vùng nhớ **Heap** (từ Java 7 trở đi; trước đó nằm ở PermGen).
- String literal `"hello"` → JVM tìm trong pool, nếu có thì trả về reference cũ, nếu không thì tạo mới và đưa vào pool.
- `new String("hello")` → luôn tạo object mới trên heap, **không** vào pool tự động.
- `String.intern()` → đưa string đang có vào pool (hoặc trả về reference đã có trong pool).
- `==` so sánh reference. `equals()` so sánh nội dung. Đây là nguồn gốc của rất nhiều bug.

---

## 4) Ví dụ code đơn giản

```java
String a = "hello";
String b = "hello";
String c = new String("hello");
String d = c.intern();

System.out.println(a == b);        // true  — cùng pool reference
System.out.println(a == c);        // false — c là object mới trên heap
System.out.println(a == d);        // true  — d.intern() trả về pool reference
System.out.println(a.equals(c));   // true  — nội dung giống nhau

// Kết quả compile-time concatenation cũng vào pool:
String e = "hel" + "lo";           // constant folding → "hello" → pool
System.out.println(a == e);        // true

// Runtime concatenation thì không:
String x = "hel";
String f = x + "lo";               // tạo object mới
System.out.println(a == f);        // false
```

---

## 5) Trả lời Middle+

> String Pool là vùng nhớ đặc biệt trong Heap, lưu các String literal duy nhất để tái sử dụng, tiết kiệm bộ nhớ. Khi dùng literal `"hello"`, JVM tự động interning — trả về cùng một reference nếu đã tồn tại. `new String("hello")` bỏ qua pool và tạo object mới. So sánh String nên dùng `equals()`, không dùng `==`, vì `==` kiểm tra reference chứ không kiểm tra nội dung.

---

## 6) Trả lời Senior

> Từ Java 7+, String Pool chuyển từ PermGen sang Heap, giúp GC có thể thu hồi các interned String không còn được tham chiếu — tránh `OutOfMemoryError` kiểu PermGen ngày xưa.
>
> **Trade-off của `intern()`:** Gọi `intern()` thủ công có thể tối ưu memory khi xử lý lượng lớn String trùng lặp (ví dụ: parse CSV hàng triệu dòng có cùng giá trị). Tuy nhiên, JVM dùng hashtable nội bộ cho pool — nếu intern quá nhiều String khác nhau, lookup có thể tốn CPU và gây GC pressure ngược lại.
>
> **Production concern:** Các framework như Jackson, Hibernate đôi khi intern field name hoặc column name tự động. Nếu dữ liệu input đến từ user (ví dụ JSON key động), không nên intern — đây là vector tiềm ẩn cho memory exhaustion.
>
> **JVM flag:** `-XX:+StringDeduplication` (G1 GC) cho phép GC tự dedup char array của String trùng nội dung trên heap mà không cần dùng pool — thay thế tốt cho `intern()` trong nhiều trường hợp.

---

## 7) Follow-up / pitfall

- **`==` vs `equals()`** — lỗi kinh điển nhất: `if (status == "active")` sẽ không đáng tin cậy; phải dùng `equals()` hoặc `"active".equals(status)`.
- **`intern()` trong vòng lặp lớn** — gọi intern() cho mọi String parse được từ file sẽ gây bottleneck ở hashtable nội bộ JVM.
- **String constant pool và switch-case** — `switch (str)` dùng `equals()` bên dưới (không dùng `==`), nên hoạt động đúng kể cả với `new String(...)`.
- **Câu hỏi xoáy:** "String immutable có liên quan gì đến String Pool không?" → Immutability là điều kiện cần để interning an toàn: nếu String mutable, một chỗ thay đổi sẽ ảnh hưởng tất cả reference trỏ vào cùng pool entry.
- **Câu hỏi xoáy:** "Từ Java 7 lên Java 8/11/17 có gì thay đổi với String không?" → Java 9: String đổi backing từ `char[]` sang `byte[]` (Compact Strings) để tiết kiệm memory với Latin-1 content.

---

## 8) 30 giây tóm tắt miệng

> "String Pool là vùng nhớ trong Heap lưu các String literal để tái sử dụng. Literal `"hello"` tự động vào pool; `new String("hello")` thì không. So sánh String luôn dùng `equals()` vì `==` so reference. Nếu cần ép vào pool thủ công thì gọi `intern()`, nhưng cẩn thận với dữ liệu động vì có thể gây memory pressure. Từ Java 7 pool nằm trên Heap nên GC có thể dọn được, khác với thời PermGen."
