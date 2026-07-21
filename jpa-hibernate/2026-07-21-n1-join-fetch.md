# N+1 Query và Join Fetch

- **Date:** 2026-07-21
- **Topic:** jpa-hibernate
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

N+1 query trong JPA/Hibernate là gì? Làm sao phát hiện và xử lý bằng `JOIN FETCH`?

## 2) Giải thích đơn giản

Tưởng tượng: bạn hỏi DB "cho danh sách 10 user". Xong mỗi user lại hỏi thêm "posts của user này đâu?" → tổng cộng 1 + 10 = 11 câu hỏi. Đó là **N+1 query problem** (vấn đề N+1 truy vấn).

| Cách | Số query (10 user) | Ý nghĩa |
|------|-------------------:|---------|
| Lazy load từng cái | 1 + 10 = 11 | Chậm, dễ nổ production |
| `JOIN FETCH` | 1 | Lấy user + posts cùng lúc |

- **Lazy loading** (tải trễ): quan hệ chưa lấy ngay; chạm field mới query.
- **Join fetch**: bảo Hibernate join bảng con ngay trong query gốc.

## 3) Trả lời Middle+

N+1: 1 query lấy list entity cha + N query lấy association (thường vì `LAZY` + chạm collection trong loop).

Cách xử lý phổ biến:
- JPQL/HQL: `JOIN FETCH`
- `@EntityGraph` / `EntityGraph`
- Batch size (`@BatchSize` / `default_batch_fetch_size`) khi không muốn join hết

Luôn bật SQL log hoặc datasource proxy lúc dev để đếm query thật.

## 4) Trả lời Senior

Default nên để association `LAZY`; fetch theo use case, không flip `EAGER` global.

Trade-off `JOIN FETCH`:
- Giảm round-trip, nhưng Cartesian product / duplicate parent khi fetch nhiều collection cùng lúc
- Khó kết hợp pagination (`setFirstResult`/`setMaxResults`) với `JOIN FETCH` collection — dễ sai page size
- Collection lớn: cân nhắc DTO projection, 2 query (ids rồi fetch), hoặc batch fetch thay vì một join phình

Production checklist: SQL metrics, slow query, tránh map entity lazy ra ngoài transaction (LazyInitializationException), không tin "chạy local nhanh là ổn".

## (tuỳ chọn) Ví dụ code

```java
// N+1: mỗi user.getPosts() có thể phát sinh thêm SELECT
List<User> users = em.createQuery("select u from User u", User.class)
    .getResultList();
users.forEach(u -> u.getPosts().size());

// 1 query: fetch kèm posts
List<User> usersFetched = em.createQuery(
    "select distinct u from User u left join fetch u.posts where u.active = true",
    User.class
).getResultList();
```

## (tuỳ chọn) Follow-up / pitfall

- `join fetch` 2 collection (`posts` + `roles`) cùng query → risk nhân bản row / memory.
- Pagination + fetch collection: thường tách query hoặc dùng window/DTO.
- Open-session-in-view che N+1 đến lúc traffic cao mới lộ.

## 5) 30 giây tóm tắt miệng

N+1 là 1 query cha cộng N query con khi đụng lazy association trong vòng lặp. Fix hay dùng: `JOIN FETCH` hoặc entity graph để lấy đủ data một lần; nhớ trade-off duplicate và pagination. Production thì đếm SQL thật, đừng để EAGER lung tung.
