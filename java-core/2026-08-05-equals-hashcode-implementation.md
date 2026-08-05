# Cách triển khai equals() và hashCode()

- **Date:** 2026-08-05
- **Topic:** java-core
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Nếu tôi cung cấp một lớp Employee có các trường id, name, salary, làm thế nào để bạn đảm bảo hai Employee objects có cùng id được coi là bằng nhau bất kể name hay salary thay đổi? Hãy giải thích tại sao cả equals() lẫn hashCode() đều quan trọng.

## 2) Giải thích như trẻ lên 3

Hãy tưởng tượng Employee là một chiếc hộp có mã vạch (id) và nội dung (name, salary). Hai hộp có cùng mã vạch = cùng một người, nên chúng giống nhau dù nội dung khác nhau.

## 3) Giải thích đơn giản cho dev

Mặc định equals() so sánh địa chỉ bộ nhớ (có phải cùng object không?). overriden equals() kiểm tra tất cả các trường quan trọng (thường là khóa chính). hashCode() phân phối các object vào các bucket trong HashMap dựa trên các trường giống nhau.

## 4) Ví dụ code đơn giản

```java
class Employee {
    private final int id;
    private String name;
    private double salary;
    
    public Employee(int id, String name, double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Employee employee = (Employee) o;
        return id == employee.id;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

## 5) Trả lời Middle+

Sử dụng các trường duy nhất (khóa chính) trong equals() để đảm bảo contract: nếu a.equals(b) thì a.hashCode() == b.hashCode(). Giữ hashCode() ổn định theo thời gian.

## 6) Trả lời Senior

1. Contract: equals()+hashCode() phải nhất quán, tuân thủ reflexivity, symmetry, transitivity, consistency.
2. Khóa phụ: thêm các trường không-null vào hashCode() để tránh NullPointerExceptions.
3. Immutability: các trường dùng trong equals/hashCode() nên final/immutable; mutating sau khi đưa vào Set/Map sẽ làm hỏng cấu trúc.
4. Performance: ưu tiên cache hashCode, tránh tái tính toán.
5. Cảnh báo: đừng mix thói quen equals() chính xác với hashCode() sai.

## 7) Follow-up / pitfall

- Khi nào nên sử dụng comparesTo (Comparable) thay vì equals? (Sắp xếp vs. tìm kiếm)
- Sử dụng @EqualsAndHashCode của Lombok an toàn không? (Phụ thuộc vào design)
- Collection nào sử dụng equals() chứ không phải hashCode()? (TreeSet)
- Impact của bad hashCode lên HashMap? (Phân bố xấu dẫn đến O(n) lookup)

## 8) 30 giây tóm tắt miệng

Hai Employee objects giống nhau nếu chúng có cùng mã số định danh, bất kể name/salary. Triển khai equals() theo id, hashCode() theo id (Objects.hash(id)). Giữ cả hai method nhất quán để HashMap hoạt động đúng. Lưu ý các trường không-null, tính ổn định, hiệu năng và tuân thủ contract (reflexivity, symmetry, transitivity, consistency).