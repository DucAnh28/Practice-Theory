# JPA Entity States: Merge vs Persist

- **Date:** 2026-10-07
- **Topic:** jpa-hibernate
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Khi nào dùng `persist()` và khi nào dùng `merge()` trong JPA? Có gì khác biệt về entity state?"

## 2) Giải thích như trẻ lên 3

Bạn có 1 con búp bê:
- **persist()**: Bạn mua búp bê mới, chưa có tên, bạn đặt tên và đưa nó vào tủ đồ chơi. Từ giờ búp bê này thuộc về bạn.
- **merge()**: Bạn tìm thấy búp bê cũ ngoài sân (có tên rồi), bạn đưa nó vào tủ. Nhưng tủ đồ chơi sẽ kiểm tra xem có búp bê cùng tên trong tủ không, rồi cập nhật hoặc thêm mới.

## 3) Giải thích đơn giản cho dev

JPA entity có 4 trạng thái:
1. **Transient**: Object mới tạo, chưa có ID, chưa được persistence context quản lý
2. **Managed**: Có trong persistence context, mọi thay đổi sẽ auto flush vào DB
3. **Detached**: Đã từng managed nhưng session đóng hoặc entity bị evict
4. **Removed**: Đánh dấu xóa, sẽ DELETE khi flush

**persist()**: Chuyển entity **Transient → Managed**. Chỉ dùng cho entity mới, không có ID.

**merge()**: Copy data từ entity (thường **Detached**) vào 1 entity **Managed** mới/cũ trong context. Trả về instance Managed, còn object gốc vẫn Detached.

## 4) Ví dụ code đơn giản

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String name;
}

// Case 1: persist() - entity mới
User newUser = new User();
newUser.setName("Alice");
em.persist(newUser);  // newUser giờ là Managed, ID sẽ được generate
// newUser vẫn là reference đang dùng

// Case 2: merge() - entity detached
User detached = new User();
detached.setId(1L);
detached.setName("Bob");
User managed = em.merge(detached);  // detached vẫn là Detached
managed.setName("Charlie");  // thay đổi này sẽ flush vào DB
// detached.getName() vẫn là "Bob"
```

## 5) Trả lời Middle+

**persist():**
- Dùng cho entity mới (transient), chưa có ID
- Entity được truyền vào trở thành managed luôn
- Ném exception nếu entity đã có ID hoặc đã managed
- Cascade: nếu có relationship, entity con cũng được persist

**merge():**
- Dùng cho entity detached hoặc muốn "upsert"
- Trả về 1 managed instance mới, object gốc vẫn detached
- Nếu ID không tồn tại trong DB → INSERT
- Nếu ID đã có → SELECT rồi UPDATE
- An toàn hơn khi không chắc entity state

**Lưu ý:** Sau `merge()`, phải dùng return value, đừng tiếp tục dùng object gốc.

## 6) Trả lời Senior

**Production concerns:**

1. **Performance:**
   - `merge()` có overhead: SELECT trước khi UPDATE (nếu entity chưa có trong context)
   - `persist()` nhanh hơn cho bulk insert vì không cần check DB

2. **Detached entity trong web app:**
   - Controller nhận DTO, map sang entity → thường là transient/detached
   - Nếu dùng `persist()` cho entity có ID → exception
   - `merge()` an toàn hơn nhưng có thể ghi đè lên data mới từ concurrent request
   - Best practice: Load entity từ DB trong transaction, rồi update từng field cần thiết

3. **Identity:**
   ```java
   User original = new User();
   original.setId(1L);
   User managed = em.merge(original);
   System.out.println(original == managed);  // false!
   ```
   Reference khác nhau → nếu code khác giữ reference cũ, sẽ không thấy update

4. **Cascade pitfalls:**
   - `merge()` với cascade ALL/MERGE: tất cả child entities cũng bị merge → có thể ghi đè unintended data
   - `persist()` cascade: tạo mới toàn bộ graph, dễ control hơn cho insert flow

**Trade-off:**
- REST API update: nên load entity hiện tại, validate, rồi set fields; tránh merge() blind
- Batch insert: dùng `persist()` + `flush()` + `clear()` theo batch
- Optimistic locking: `@Version` works với cả 2, nhưng merge() phải ensure version field đúng

## 7) Follow-up / pitfall

**Q: Nếu gọi `persist()` 2 lần cho cùng 1 object?**
A: Lần 2 không làm gì (entity đã managed). Nhưng nếu persist entity đã có ID → `PersistenceException`.

**Q: Merge() có trigger SELECT không cần thiết không?**
A: Có. Nếu entity đã có trong context (by ID), Hibernate reuse; nếu không, phải SELECT. Dùng `session.contains()` hoặc `em.find()` trước để avoid.

**Q: Tại sao sau merge() phải dùng return value?**
A: Vì JPA spec không bảo đảm modify object gốc. Hibernate đôi khi modify in-place nếu entity đã trong context, nhưng không nên rely vào đó.

**Common mistake:**
```java
user.setName("New Name");
em.merge(user);  // ❌ bỏ qua return value
// user reference có thể không reflect changes nếu merge tạo instance mới
```

Đúng:
```java
user = em.merge(user);  // ✅
```

## 8) 30 giây tóm tắt miệng

"persist() cho entity mới, biến nó thành managed ngay. merge() cho detached entity, copy data vào managed instance và trả về, object gốc vẫn detached. Trong production web app, tôi tránh merge() blind vì dễ ghi đè data; thường load entity hiện tại từ DB rồi update từng field cần thiết, giúp kiểm soát được concurrent update và validate đúng business logic."
