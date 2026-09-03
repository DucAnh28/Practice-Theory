# `orphanRemoval` và `CascadeType.REMOVE` trong JPA

- **Date:** 2026-09-03
- **Topic:** jpa-hibernate
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

`orphanRemoval = true` khác gì `CascadeType.REMOVE`? Khi nào nên dùng mỗi loại?

## 2) Giải thích như trẻ lên 3

Hãy coi `Order` là một chiếc hộp và `OrderItem` là món đồ trong hộp:

- **orphanRemoval:** lấy món đồ ra khỏi hộp thì món đó bị bỏ đi.
- **Cascade REMOVE:** vứt cả chiếc hộp thì các món đồ bên trong cũng bị bỏ.

## 3) Giải thích đơn giản cho dev

| Cơ chế | Khi nào xóa entity con? |
|---|---|
| `orphanRemoval = true` | Khi entity con bị loại khỏi quan hệ với entity cha |
| `CascadeType.REMOVE` | Khi gọi `remove()` cho entity cha |

Hai cơ chế có thể dùng cùng nhau nhưng không thay thế nhau. Chúng phù hợp khi entity con thuộc vòng đời của entity cha và không nên tồn tại độc lập.

## 4) Ví dụ code đơn giản

```java
@OneToMany(mappedBy = "order",
           cascade = CascadeType.ALL,
           orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();

public void removeItem(OrderItem item) {
    items.remove(item);
    item.setOrder(null); // giữ hai phía của quan hệ đồng bộ
}
```

Xóa item khỏi `items` sẽ khiến JPA xóa bản ghi `OrderItem` khi flush. Xóa `Order` cũng lan truyền thao tác xóa vì có cascade.

## 5) Trả lời Middle+

`orphanRemoval` xóa entity con khi nó không còn được tham chiếu bởi quan hệ của entity cha. `CascadeType.REMOVE` chỉ lan truyền thao tác xóa từ cha xuống con. Tôi dùng chúng cho quan hệ sở hữu chặt, ví dụ `Order`–`OrderItem`, và luôn cập nhật cả hai phía của quan hệ trong helper method.

## 6) Trả lời Senior

Chỉ bật hai cơ chế này khi aggregate boundary (ranh giới cụm dữ liệu) rõ ràng và entity con không được chia sẻ. Với collection lớn, cascade delete có thể tạo nhiều câu SQL và giữ nhiều entity trong persistence context; bulk delete hoặc xóa ở database có thể phù hợp hơn nhưng sẽ bỏ qua lifecycle callback và trạng thái trong context. Cần kiểm tra SQL thực tế, transaction rollback, khóa ngoại và optimistic locking trước khi dùng ở production.

## 7) Follow-up / pitfall

- Dùng `orphanRemoval` cho entity con được nhiều entity cha dùng chung có thể xóa nhầm dữ liệu.
- Chỉ sửa một phía của quan hệ hai chiều khiến trạng thái object và database lệch nhau.
- Thay toàn bộ collection thay vì sửa collection đang được JPA quản lý có thể gây hành vi khó đoán.
- Bulk delete không đồng bộ persistence context hiện tại.

## 8) 30 giây tóm tắt miệng

`orphanRemoval` xóa entity con khi nó bị tách khỏi quan hệ; `CascadeType.REMOVE` xóa con khi cha bị xóa. Tôi chỉ dùng chúng khi con thuộc hoàn toàn vòng đời của cha, đồng bộ cả hai phía quan hệ và kiểm tra chi phí SQL, khóa ngoại cùng transaction trong production.
