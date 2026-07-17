# CompletableFuture — Chaining & Error Handling

- **Date:** 2026-07-18
- **Topic:** concurrency
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> Anh/chị dùng CompletableFuture trong Java để xử lý một chuỗi tác vụ bất đồng bộ (gọi API A → xử lý kết quả → gọi API B) như thế nào? Khi một bước fail thì xử lý ra sao?

## 2) Giải thích như trẻ lên 3

Tưởng tượng bạn đi mua trà sữa: xếp hàng mua (API A) → lấy trà sữa về pha thêm topping (xử lý) → mang đến bàn cho bạn gái (API B). Nếu quán hết trà sữa, bạn không bỏ cuộc mà gọi đồ uống khác thay thế (fallback). Cái chuỗi "mua → topping → giao" đó là CompletableFuture chaining.

## 3) Giải thích đơn giản cho dev

`CompletableFuture` là công cụ lập trình bất đồng bộ trong Java. Thay vì viết callback hell (lồng nhau tùm lum), bạn dùng phương thức chain như:

- `thenApply(fn)` — biến đổi kết quả đồng bộ sau khi có data
- `thenCompose(fn)` — nối tiếp với một async task khác (flatMap của Future)
- `thenAccept(consumer)` — xử lý kết quả cuối cùng, không trả về giá trị
- `exceptionally(fn)` — bắt lỗi, trả về giá trị fallback
- `handle((result, ex) -> ...)` — xử lý cả success lẫn failure trong một chỗ

Quan trọng: `thenApply` vs `thenApplyAsync` — async chạy trong ForkJoinPool.commonPool(), sync chạy trong thread của stage trước đó.

## 4) Ví dụ code đơn giản

```java
// Gọi API A → parse JSON → gọi API B, có fallback nếu lỗi
CompletableFuture.supplyAsync(() -> callApiA(userId))             // step 1: async call
    .thenApply(response -> parseUserId(response))                  // step 2: sync transform
    .thenCompose(id -> callApiBAsync(id))                          // step 3: another async call
    .exceptionally(ex -> {
        log.error("Chain failed, using fallback", ex);
        return "default-value";
    })
    .thenAccept(result -> System.out.println("Done: " + result));  // step 4: consume

// handle() — một chỗ xử lý cả success và failure
future.handle((result, ex) -> {
    if (ex != null) return recoverFrom(ex);
    return enrich(result);
});
```

## 5) Trả lời Middle+

Dùng `thenApply` để transform kết quả đồng bộ, `thenCompose` để nối với async call tiếp theo (tránh nested `CompletableFuture<CompletableFuture<T>>`). Dùng `exceptionally` để catch exception và trả fallback. Nếu cần log lỗi nhưng không muốn fallback, dùng `whenComplete`. Có thể combine nhiều future độc lập bằng `allOf()` + `join()`.

Hiểu khác biệt `thenApply` (sync transform) với `thenCompose` (async chaining) là điểm phỏng vấn hay hỏi.

## 6) Trả lời Senior

**Trade-off:**
- `supplyAsync` không chỉ định executor → dùng `ForkJoinPool.commonPool()`. Trong production, common pool có thể bị block bởi task khác → nên dùng custom `Executor` với pool size riêng.
- `thenApply` chạy trên thread của stage trước; nếu function đó block (DB query), nó sẽ chiếm thread async pool → dùng `thenApplyAsync` với custom executor.

**Production concern:**
- `exceptionally` nuốt exception, mất stack trace → dùng `handle()` để log + rethrow khi cần.
- Chain dài không có timeout → dùng `orTimeout()` (Java 9+) hoặc `completeOnTimeout()`.
- CompletableFuture không hỗ trợ cancellation thực sự tốt như reactive streams.

**Observability:**
- Gắn MDC context vào thread qua executor wrapper để trace log xuyên suốt chain.
- Ghi metric cho từng stage (latency, error rate) nếu dùng trong service quan trọng.

## 7) Follow-up / pitfall

**Q: `thenApply` vs `thenApplyAsync` — dùng cái nào khi nào?**
- `thenApply`: synchronous, chạy trên thread của stage trước. Tốt nếu function nhanh (parse JSON, map DTO).
- `thenApplyAsync`: chạy trên thread pool riêng. Dùng khi function có thể block (DB, I/O) để không chiếm thread async chính.

**Q: Khi nào `thenCompose` thay vì `thenApply`?**
- Khi function return một `CompletionStage` / `CompletableFuture`. Nếu dùng `thenApply` sẽ bị `CompletableFuture<CompletableFuture<T>>` lồng nhau.

**Pitfall:**
- Gọi `get()` block main thread → phá hỏng mô hình async. Dùng `join()` cũng block nhưng throw unchecked exception.
- `allOf()` return `CompletableFuture<Void>` — không có kết quả. Phải `join()` từng future riêng sau khi `allOf().join()`.

## 8) 30 giây tóm tắt miệng

> CompletableFuture cho phép chain các tác vụ bất đồng bộ mà không bị callback hell. `thenApply` transform sync, `thenCompose` nối async task, `exceptionally` handle lỗi với fallback. Trong production cần chú ý: dùng custom executor thay vì common pool, thêm timeout cho chain, và dùng `handle()` để xử lý cả success/failure mà không nuốt exception.