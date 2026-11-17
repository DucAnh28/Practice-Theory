# List.of vs Arrays.asList vs new ArrayList — khác nhau ở chỗ nào?

- **Date:** 2026-11-17
- **Topic:** collections
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Ba cách tạo list này khác nhau thế nào: `List.of(...)`, `Arrays.asList(...)`, `new ArrayList<>(...)`? Cái nào cho phép `add`/`set`, cái nào chấp nhận `null`, cái nào an toàn để trả về từ API public?

---

## 2) Giải thích như trẻ lên 3

Ba cách đóng hộp danh sách:
- `new ArrayList<>(...)` → cái **hộp thật**, bạn thêm/bớt thoải mái.
- `Arrays.asList(...)` → cái **hộp có nắp chặt**, không thêm/bớt được nhưng vẫn xếp lại được bên trong.
- `List.of(...)` → cái **hộp bê tông**, không thêm/bớt/xếp lại được, và không nhận `null`.

Nhớ câu: "asList = sửa được nhưng không thêm; of = đóng băng hoàn toàn."

---

## 3) Giải thích đơn giản cho dev

| | `new ArrayList<>(...)` | `Arrays.asList(...)` | `List.of(...)` |
|---|---|---|---|
| Kiểu thật | `ArrayList` | `Arrays$ArrayList` | `ImmutableCollections$ListN` |
| `add`/`remove` | ✅ | ❌ `UnsupportedOperationException` | ❌ |
| `set(i, x)` | ✅ | ✅ | ❌ |
| `size` cố định? | ❌ | ✅ (fixed-size) | ✅ |
| Cho phép `null` | ✅ | ✅ | ❌ (`NullPointerException`) |
| `equals` với `List` khác | ✅ | ✅ | ✅ |
| `contains(x)` dùng `equals` | ✅ | ✅ | ✅ (khác `List.of` về null) |
| `hashCode` | theo `List` | theo `List` | theo `List` |
| Serializable | ✅ | ✅ | ✅ |

Ba điều cần nhớ:

1. **`Arrays.asList` chỉ là "view"** — nó bọc mảng gốc, `set(i, x)` sẽ ghi thẳng vào mảng đó. Thay đổi mảng gốc sau đó là thay đổi list và ngược lại.
2. **`List.of` bất biến theo cấu trúc** (structurally immutable) — không thể thêm/sửa/xoá phần tử, nhưng **phần tử bên trong vẫn có thể mutable** (xem pitfall).
3. **`List.of` cấm `null`** vì API dùng `null` làm "không có giá trị" (`Optional`-ish semantics); `Arrays.asList` và `ArrayList` cho phép.

```java
// Nhanh gọn: so sánh 3 "kiểu list" theo contract
List<String> a = new ArrayList<>(List.of("x"));
List<String> b = Arrays.asList("x");
List<String> c = List.of("x");
```

---

## 4) Ví dụ code đơn giản

```java
import java.util.*;
import java.util.function.Supplier;

public class ListDemo {

    static <T> List<T> create(Supplier<List<T>> factory) {
        return factory.get();
    }

    public static void main(String[] args) {
        // --- 1. Arrays.asList: fixed-size, nhưng set() được, và gắn với mảng gốc
        Integer[] arr = {1, 2, 3};
        List<Integer> fixed = Arrays.asList(arr);
        fixed.set(0, 99);            // OK -> [99, 2, 3]
        arr[1] = 88;                 // đổi mảng gốc -> list cũng đổi
        System.out.println(fixed);   // [99, 88, 3]

        try {
            fixed.add(4);            // UnsupportedOperationException
        } catch (UnsupportedOperationException e) {
            System.out.println("asList: không add được");
        }

        // --- 2. List.of: bất biến + cấm null
        List<String> immutable = List.of("a", "b");
        try {
            immutable.add("c");
        } catch (UnsupportedOperationException e) {
            System.out.println("of: không add được");
        }
        try {
            List.of("a", null);       // NullPointerException ngay lúc tạo
        } catch (NullPointerException e) {
            System.out.println("of: không nhận null");
        }

        // --- 3. new ArrayList<>(...) : thật sự mutable, copy để không dính mảng gốc
        List<Integer> safe = new ArrayList<>(Arrays.asList(arr));
        safe.set(0, 1);
        arr[0] = 100;                  // mảng gốc đổi, safe không bị ảnh hưởng
        System.out.println(safe);      // [1, 88, 3]

        // --- 4. Dùng factory cho cấu hình: đổi nguồn cấu hình, không sửa code khác
        List<String> cfg = create(() -> List.of("prod"));
        System.out.println(cfg);
    }
}
```

```java
// 5. Bẫy "bất biến giả": List.of chứa mutable element
List<StringBuilder> builders = List.of(new StringBuilder("a"));
builders.get(0).append("b");   // KHÔNG ném exception -> list đã "bị mutate"
System.out.println(builders.get(0)); // ab
// Muốn bất biến thật: List.copyOf(...) cũng chỉ copy cấu trúc,
// phải tự copy phần tử: List.of(new StringBuilder("a").toString())
```

---

## 5) Trả lời Middle+

> `Arrays.asList` trả về list **fixed-size** view trên mảng: `set()` được nhưng `add`/`remove` ném `UnsupportedOperationException`, và nó dính vào mảng gốc nên phải copy bằng `new ArrayList<>(...)` trước khi muốn sửa. `List.of` trả về list **bất biến** (không add/set/remove được) và **không chấp nhận `null`** — dùng nó cho hằng số, default config, hoặc giá trị trả về từ method để tránh caller sửa bậy. `new ArrayList<>(...)` là list thật, mutable, có thể chứa `null`. Cả ba đều implement `List` nên `equals`/`hashCode` giống nhau, đổi vào `HashSet` hay `HashMap` không bị khác biệt.

---

## 6) Trả lời Senior

> **Trade-off & failure mode:** `Arrays.asList` là lỗi kinh điển của production — trong code review tôi luôn soi chỗ nhận `List` từ method mà bên trong dùng `Arrays.asList`, vì `add()` sẽ ném exception **lúc runtime** chứ không phải lúc compile; và nếu ai đó `set()` vào đó, nó ghi thẳng vào mảng dùng chung ở nơi khác (side-effect ngoài ý muốn). `List.of` không phải là "immutable collection" tuyệt đối: nó chỉ **structurally immutable** — phần tử bên trong vẫn mutate được nếu là mutable object, tạo ra vi phạm tính bất biến mà rất khó phát hiện (ví dụ list các `StringBuilder` trong cache).
>
> **Null semantics:** `List.of` cấm `null` (fail-fast bằng `NullPointerException` lúc khởi tạo) — đây là điều tốt cho config/constant, nhưng là breaking change nếu migrate từ `Arrays.asList` với dữ liệu có `null` từ DB/JSON. Với list có thể chứa `null`, dùng `Arrays.asList` hoặc `Collections.unmodifiableList(new ArrayList<>(...))` (bản `unmodifiableList` **vẫn chứa `null`** và **không copy**).
>
> **Performance:** `List.of` tối ưu cho list nhỏ (1–2 phần tử dùng `List12`), không có mảng backing riêng; `Arrays.asList` không copy ⇒ O(1) và zero allocation; `new ArrayList<>(...)` copy O(n). Với list lớn cần gọi `Collections.unmodifiableList` thay vì `List.copyOf` để tránh copy thêm một lần — nhưng phải chấp nhận semantics khác (`copyOf` cũng cấm `null`).
>
> **JPA/Hibernate:** `List.of` là immutable ⇒ Hibernate không thể lazy-persist nó (entity có quan hệ `@OneToMany` cần collection mutable); dùng `new ArrayList<>()` cho entity, đừng dùng `List.of` cho quan hệ. Ngược lại, projection/dto read-only thì `List.of` rất hợp.
>
> **Spring `@ConfigurationProperties`:** binding list tạo ra list mutable của Spring; nếu muốn trả ra ngoài dùng `List.copyOf` để đóng băng.
>
> **Observability:** lỗi "UnsupportedOperationException" giữa trời đất thường xuất hiện ở tầng async/consumer do list được tạm bởi framework rồi bị code của ta sửa — nên bắt stack trace và log `class` của list khi debug, không chỉ log message.

---

## 7) Follow-up / pitfall

- **`Arrays.asList` không copy mảng** — `set()` đổi cả mảng gốc. Nếu cần tách biệt: `new ArrayList<>(Arrays.asList(arr))` hoặc `new ArrayList<>(List.of(arr))` (cái này copy).
- **`Arrays.asList` với `int[]` không hoạt động như mong đợi** — `Arrays.asList(int[])` tạo list 1 phần tử kiểu `int[]`, không phải list các số nguyên. Phải `Arrays.stream(intArray).boxed().toList()` hoặc `Arrays.asList(Integer[])`.
- **`List.of` cấm `null`** → `NullPointerException` **ngay lúc gọi**, không phải lúc `add`. Dùng `List.ofNullable(x)` (Java 9+) nếu cần một giá trị có thể null.
- **`List.of` trả về list bất biến nhưng không phải `Serializable`-friendly cho mọi use case** — bản thân nó implement `Serializable` nhưng element phải serializable.
- **`Collections.unmodifiableList` KHÔNG copy** — mà chỉ chặn thao tác qua view; nếu giữ reference tới list gốc vẫn sửa được và nhìn xuyên thấy thay đổi. `List.copyOf` mới copy (và cấm `null`).
- **`Collections.unmodifiableList` cho phép `null`**, `List.copyOf`/`List.of` thì không → đừng đổi qua loa giữa hai loại mà quên kiểm tra dữ liệu có `null`.
- **`equals` giữa `List.of` và `ArrayList` vẫn `true`** nếu cùng phần tử — nhưng `List.of` với `null` sẽ throw, còn `Arrays.asList(null)` tạo list chứa `null`; `contains(null)` trên `List.of` **không** ném (khác với `contains` trên `Map`/`Set` không cho null key).
- **`remove(Object)` vs `remove(int)`** trên `Arrays.asList` — trước Java 8 từng là cái bẫy ambiguity; hiện `remove(int)` là theo index và sẽ ném `UnsupportedOperationException`.
- **Câu hỏi xoáy:** "List.of có thread-safe không?" → Có, vì không thể mutate; nhưng **không** đồng nghĩa với mọi thao tác composite (ví dụ `list.contains(x) && list.remove(x)` vẫn cần lock nếu list thay đổi).
- **Câu hỏi xoáy:** "Tạo list rỗng bằng cách nào?" → `List.of()` (không dùng `Arrays.asList()` — trả về list fixed-size rỗng nhưng add sẽ throw).

---

## 8) 30 giây tóm tắt miệng

> "`new ArrayList<>(...)` là list thật mutable và copy; `Arrays.asList(...)` là view fixed-size trên mảng — `set` được nhưng `add`/`remove` throw `UnsupportedOperationException`, và nó dính vào mảng gốc; `List.of(...)` là list bất biến, cấm `null`, dùng cho constant/default config và giá trị trả về. Ba loại đều có `equals`/`hashCode` giống nhau. Pitfall lớn nhất: `Arrays.asList` không copy, và `List.of` chỉ bất biến về cấu trúc — phần tử mutable bên trong vẫn sửa được. Với entity JPA hay collection dùng chung nhiều thread, tôi luôn dùng `new ArrayList<>` bên trong và `List.copyOf` ở biên API."
