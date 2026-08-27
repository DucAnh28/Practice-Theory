# CompletableFuture — Xử lý exception

- **Date:** 2026-08-27
- **Topic:** concurrency
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi dùng `CompletableFuture.supplyAsync(...)` hoặc `thenApply(...)`, làm sao để bắt exception đúng cách? Khác nhau gì giữa `exceptionally`, `handle`, và `whenComplete`?

## 2) Giải thích như trẻ lên 3

Bạn giao việc A cho bạn bè: "Mua trà sữa giúp". Bạn bè có thể:
- Mang trà sữa về OK → dùng bình thường.
- Trượt, té => hét lên "Lỗi!" và tự xử lý: gọi Grab (`exceptionally`).
- Vừa lấy trà sữa, vừa in log thành công/thất bại (`whenComplete`).
- Mở hộp xem mùi vị có ổn không rồi mới in (`handle`).

## 3) Giải thích đơn giản cho dev

- `exceptionally(Function<Throwable, T>)`: bắt lỗi và trả giá trị thay thế.
- `handle(BiFunction<T, Throwable, R>)`: luôn chạy, nhận cả kết quả và exception, quyết định trả gì.
- `whenComplete(BiConsumer<T, Throwable>)`: chạy sau cùng, không biến đổi kết quả, dùng để log/cleanup.

## 4) Ví dụ code

```java
CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("boom");
    return 42;
})
.exceptionally(ex -> {
    System.out.println("fallback: " + ex.getMessage());
    return -1;
})
.thenApply(result -> result * 2)
.whenComplete((res, ex) -> System.out.println("done"));
```

## 5) Trả lời Middle+

Dùng `exceptionally` để bắt exception trong pipeline và trả default value. `handle` khi cần kiểm tra cả success + failure, hỗ trợ biến đổi kết quả. `whenComplete` dùng để side-effect (log, audit) mà không đổi kiểu trả về.

```java
cf.exceptionally(ex -> defaultValue)
  .thenApply(v -> ...);
```

## 6) Trả lời Senior

Chọn `handle` khi cần normalize exception thành domain error (ví dụ `Result<T, AppError>`). Tránh catch `Exception` quá rộng. Khi propagate lên, gói lại bằng `CompletionException`. Nhớ rằng `exceptionally` chỉ chạy khi stage đó lỗi; lỗi từ stage trước sẽ propagate xuống next stage. Check cause: `Throwable cause = (ex instanceof CompletionException) ? ex.getCause() : ex`. Ở production, thêm deadline + supervisor để tránh leak thread.

## 7) Follow-up / pitfall

- `exceptionally` trả default value hợp lệ? check null.
- `thenApply` không bắt exception trong lambda, nó tự propagate.
- Bỏ sót `CompletionException` wrapper khi unwrap.

## 8) 30 giây tóm tắt miệng

`CompletableFuture` có 3 cơ chế xử lý lỗi: `exceptionally` để fallback, `handle` để transform cùng kết quả và error, `whenComplete` để side-effect. Luôn wrap exception bằng `CompletionException` khi viết lambda, nhớ unwrap bằng `.getCause()` khi custom handler.