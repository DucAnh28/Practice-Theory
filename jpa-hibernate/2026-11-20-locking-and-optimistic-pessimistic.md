# Optimistic vs Pessimistic Locking trong JPA/Hibernate

- **Date:** 2026-11-20
- **Topic:** jpa-hibernate
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi hai request đồng thời sửa cùng một record (ví dụ trừ số dư, cộng voucher), làm sao tránh mất update (lost update)? Khi nào dùng optimistic locking, khi nào dùng pessimistic locking?

## 2) Giải thích như trẻ lên 3

Optimistic locking: hai đứa cùng điền vào một cái phiếu giấy, ai viết xong đóng bút lại thì đứa còn lại phải viết lại từ đầu vì bản giờ không còn khớp.

Pessimistic locking: ai giữ cái phiếu thì người khác phải chờ, không ai được động vào.

## 3) Giải thích đơn giản cho dev

- **Optimistic**: không khóa dòng khi đọc. Thêm cột `@Version`; khi UPDATE, Hibernate ghi kèm điều kiện `where id = ? and version = ?`. Nếu `update count = 0` nghĩa là có người đã sửa trước → ném `OptimisticLockException` (Spring: `ObjectOptimisticLockingFailureException`). App tự retry với backoff.
- **Pessimistic**: khóa dòng ngay khi đọc bằng `SELECT ... FOR UPDATE` (row lock tầng DB, khóa đến khi transaction commit/rollback). Tốn tài nguyên connection lâu hơn, chặn ghi trên cùng dòng, không cần version.
- Chọn: đọc nhiều ghi ít, contention thấp → optimistic. Tranh chấp nóng trên ít dòng (số dư, stock) → pessimistic hoặc optimistic + retry.
- Trong hệ thống phân tán, cần lock ở tầng Redis/DB row-level thật sự, vì `@Version` chỉ bảo vệ trong 1 DB instance.

## 4) Ví dụ code đơn giản

```java
@Entity
class Account {
    @Id Long id;
    @Version Long version;      // tự tăng mỗi lần update
    @Column(nullable = false)
    BigDecimal balance;
}
```

```java
// Optimistic: ném exception nếu có người sửa trước
@Transactional
void withdraw(Long id, BigDecimal amount) {
    Account a = repo.findById(id).orElseThrow();
    a.setBalance(a.getBalance().subtract(amount));
    repo.save(a); // UPDATE ... WHERE id=? AND version=?
}
```

```java
// Pessimistic: khóa dòng trước khi sửa
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id = :id")
Optional<Account> findForUpdate(@Param("id") Long id);

@Transactional
void withdrawPessimistic(Long id, BigDecimal amount) {
    Account a = accountRepo.findForUpdate(id).orElseThrow();
    a.setBalance(a.getBalance().subtract(amount));
}
```

Retry (Spring):

```java
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class,
           maxAttempts = 3, backoff = @Backoff(delay = 50))
@Transactional
void transfer(Long from, Long to, BigDecimal amount) { ... }
```

## 5) Trả lời Middle+

Để tránh lost update khi 2 transaction cùng sửa 1 record: optimistic locking bằng cột `@Version` — Hibernate update kèm điều kiện version, nếu không khớp thì ném `OptimisticLockException` và app retry; pessimistic locking dùng `SELECT ... FOR UPDATE` để khóa dòng ngay. Optimistic phù hợp khi contention thấp, pessimistic khi nhiều giao dịch tranh chấp cùng dòng (ví dụ trừ số dư). Với giao dịch chỉ đọc thì không cần lock.

## 6) Trả lời Senior

Chọn giữa optimistic/pessimistic dựa trên contention và tác động business khi phải retry:

- Optimistic: rẻ, không giữ DB connection lâu, không chặn, nhưng lỗi phát hiện tại commit (fail-late) → nếu business flow dài (đã gửi email/đã trừ ví ngoài) thì phải có outbox/idempotency để retry không gây tác dụng phụ. Retry cần jitter + giới hạn số lần.
- Pessimistic: phát hiện sớm, chặn writer, nhưng giữ row lock + connection suốt transaction → deadlock nếu thứ tự lock không nhất quán, lock timeout, và chịu bottleneck khi transaction dài. Cần timeout cấu hình (`@QueryHints`/`jakarta.persistence.lock.timeout`) và index đúng để `FOR UPDATE` không quét rộng.
- Multi-instance/partitioned DB: row lock chỉ có tác dụng trong 1 DB; multi-primary hoặc sharding phải dùng cơ chế ở tầng khác (Redis lock, saga, idempotency key).
- Observability: log/counter số lần `OptimisticLockException` để phát hiện hotspot sớm.
- Với read-heavy, dùng optimistic + version hoặc snapshot isolation; nhớ `isolation = REPEATABLE_READ` (MySQL mặc định) đã gây hiện tượng write skew.

## 7) Follow-up / pitfall

- `@Version` trên entity kế thừa có thể bị cấu hình sai → version không tăng → optimistic lock vô hiệu.
- Dùng `save()` / `merge()` sai (merge là `SELECT` rồi `UPDATE` → có thể ghi đè thay đổi của người khác).
- `PESSIMISTIC_WRITE` không hoạt động khi đọc bằng `findById` thuần; phải dùng query có `@Lock` + transaction bao quanh (nếu không có transaction, lock bị release ngay).
- Ném exception khi commit (optimistic) vs khi query (pessimistic) → cả hai đều cần xử lý ở tầng API, trả 409 Conflict thay vì 500.
- `@Lock` + `Pageable` (phân trang) là combo sai — Spring Data không hỗ trợ lock với phân trang.
- Đừng giữ pessimistic lock khi gọi service bên ngoài (HTTP call, MQ publish) — giữ transaction lâu, dễ deadlock.

## 8) 30 giây tóm tắt miệng

Lost update xảy ra khi 2 transaction cùng đọc 1 record rồi ghi đè nhau. Optimistic locking thêm cột `@Version`, update kèm điều kiện version, thất bại thì ném exception và retry — rẻ, hợp khi ít tranh chấp. Pessimistic locking dùng `SELECT ... FOR UPDATE` để khóa dòng ngay — phát hiện sớm nhưng giữ lock lâu, dễ deadlock, cần timeout và thứ tự lock nhất quán. Với hệ thống nhiều instance, cần thêm idempotency/outbox bên cạnh.
