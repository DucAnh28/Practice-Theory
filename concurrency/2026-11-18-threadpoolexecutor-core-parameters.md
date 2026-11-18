# ThreadPoolExecutor — 4 tham số cốt lõi và cái bẫy queue hàng loạt

- **Date:** 2026-11-18
- **Topic:** concurrency
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> Cho biết `ThreadPoolExecutor` chạy task theo thứ tự nào: tạo thread mới, đẩy vào queue, hay reject? Tại sao `Executors.newFixedThreadPool(10)` vẫn có thể làm hệ thống chết dù không vượt quá 10 thread?

---

## 2) Giải thích như trẻ lên 3

Tưởng tượng quầy cà phê có 10 nhân viên pha máy:
- Khách đến mà đang có người rảnh tay → nhân viên đó pha ngay.
- Hết người rảnh tay → khách được ghi vào **sổ chờ** (queue).
- Sổ chờ đầy → thuê thêm nhân viên (tối đa theo `maxPoolSize`).
- Thuê hết mà vẫn đông → **khách phải ra về** (reject).

Cái bẫy: nếu sổ chờ là cuốn sổ vô hạn trang, thì sẽ **luôn có chỗ để ghi**, nên quầy **không bao giờ thuê thêm người** — dù có cấu hình `maxPoolSize` rất lớn. Khách tích tụ trong sổ chờ và phục vụ chậm dần.

---

## 3) Giải thích đơn giản cho dev

```java
new ThreadPoolExecutor(
    corePoolSize,      // số thread "nền" luôn sẵn sàng
    maximumPoolSize,   // trần thread, chỉ đạt được khi queue ĐÃ ĐẦY
    keepAliveTime,     // thời gian thread non-core được giữ sống
    TimeUnit.SECONDS,
    workQueue,         // hàng đợi task
    threadFactory,
    rejectedHandler    // xử lý khi không còn chỗ nào
);
```

**Thứ tự xử lý một task (đây là điều hay bị nhớ sai):**

1. `pool.size() < corePoolSize` → tạo **thread mới** (core), chạy task luôn.
2. Nếu không được → `workQueue.offer(task)` → **đẩy vào queue**.
3. Queue **đầy** mà `pool.size() < maximumPoolSize` → tạo thêm **thread non-core**, chạy task luôn.
4. Vẫn không xong (pool đã max, queue đầy) → **reject** theo `RejectedExecutionHandler`.

**Vì sao `maximumPoolSize` thường "vô dụng"?**
Vì `Executors.newFixedThreadPool(n)` dùng `LinkedBlockingQueue` **không giới hạn** → bước 2 luôn thành công → bước 3 (tăng thread) không bao giờ chạy → chỉ có `corePoolSize` thread chạy thật.

**Các thứ cần nhớ thêm:**
- `keepAliveTime` chỉ áp dụng cho **thread non-core** (thread dư ra sau khi vượt core). Thread core sống tới hết chương trình, trừ khi bật `allowCoreThreadTimeOut(true)`.
- `prestartCoreThread()` / `prestartAllCoreThreads()` để warm up, tránh cold start lần đầu.
- Thread "core" chỉ là **core của pool**, không phải thread của hệ điều hành.

---

## 4) Ví dụ code đơn giản

```java
import java.util.concurrent.*;

public class PoolDemo {

    public static void main(String[] args) throws Exception {
        // (1) Queue HÀNG LOẠT -> maximumPoolSize không bao giờ được dùng
        ThreadPoolExecutor p1 = new ThreadPoolExecutor(
                2, 10, 60, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>());          // unbounded -> nguy hiểm
        submit(p1, 100);
        System.out.println("p1 poolSize=" + p1.getPoolSize());   // luôn = 2
        p1.shutdown();

        // (2) Queue có giới hạn -> vượt queue sẽ tăng thread tới maximumPoolSize
        ThreadPoolExecutor p2 = new ThreadPoolExecutor(
                2, 10, 60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(5),            // giới hạn 5
                new ThreadPoolExecutor.CallerRunsPolicy()); // backpressure
        submit(p2, 10);
        System.out.println("p2 poolSize=" + p2.getPoolSize());   // > 2
        p2.shutdown();
    }

    static void submit(ThreadPoolExecutor p, int n) {
        for (int i = 0; i < n; i++) {
            p.execute(() -> {
                try { Thread.sleep(50); } catch (InterruptedException ignored) { }
            });
        }
    }
}
```

```java
// (3) Rejection handler: khác nhau ở hành vi
// AbortPolicy (mặc định)      -> ném RejectedExecutionException
// CallerRunsPolicy            -> chạy task ngay trên thread gọi (tự backpressure)
// DiscardPolicy               -> im lặng bỏ qua (mất task, rất khó phát hiện)
// DiscardOldestPolicy         -> bỏ task cũ nhất trong queue rồi thử lại
```

---

## 5) Trả lời Middle+

> `ThreadPoolExecutor` xử lý task theo thứ tự: **đủ core thread → đẩy vào queue → queue đầy thì mới tăng thread tới `maximumPoolSize` → vẫn không xong thì reject**. Vì vậy `Executors.newFixedThreadPool(n)` với `LinkedBlockingQueue` không giới hạn sẽ **không bao giờ tạo quá `n` thread**: task thừa nằm trong queue, độ trễ tăng tuyến tính và có thể dẫn tới OOM. Muốn `maximumPoolSize` thật sự có tác dụng thì phải dùng queue có giới hạn (`ArrayBlockingQueue`/`LinkedBlockingQueue(capacity)`) — khi đó queue đầy sẽ tạo thêm thread. Với hệ thống thật, tôi luôn tự new `ThreadPoolExecutor` với queue có giới hạn + `RejectedExecutionHandler` rõ ràng, thay vì dùng `Executors.*` mặc định.

---

## 6) Trả lời Senior

> **Failure mode chính:** pool bị "nghẽn cứng" vì queue không giới hạn — symptom là p99 latency leo thang trong khi CPU vẫn rảnh, và cuối cùng là OOM do queue chứa hàng triệu task/response. Fix đúng là **bounded queue + rejection policy có chủ đích**, không phải tăng thread.
>
> **Backpressure:** `CallerRunsPolicy` là lựa chọn mặc định rất tốt cho API server: khi pool bận, request được xử lý ngay trên thread của Tomcat/Netty → tự giảm tốc độ nhận request, và **không mất task**. Nhược điểm: thread gọi bị block → nếu là Netty event loop thì tuyệt đối không được dùng `CallerRunsPolicy` cho task chạy trên event loop (sẽ chặn cả event loop đó).
>
> **Core vs max sizing:** `corePoolSize` = số worker "nền" (bằng số CPU cho CPU-bound task, hoặc bằng số connection I/O đồng thời cho I/O-bound); `maximumPoolSize` là **biên an toàn** khi queue đầy, không phải mục tiêu. Với task I/O-bound (blocking JDBC/HTTP), pool nên lớn hơn số core CPU và giới hạn thật sự nằm ở **connection pool DB/HTTP client** chứ không phải ở thread count.
>
> **Observability:** cần expose `getActiveCount()`, `getQueue().size()`, `getCompletedTaskCount()` qua Micrometer (`ExecutorServiceMetrics.monitor`) để có cảnh báo khi queue đầy. Log pool metrics định kỳ, vì "queue đầy" là failure mode im lặng.
>
> **Isolation:** không dùng chung pool cho task không liên quan — một pool bị nghẽn sẽ kéo theo mọi thứ dùng chung. Tách pool theo workload (CPU-bound vs I/O-bound) và theo tầng hệ thống.
>
> **Spring Boot:** `spring.task.execution.core-size/max-queue-size` cũng dính đúng cái bẫy này — `max-queue-size` mặc định là `Integer.MAX_VALUE` nên `max-size` không bao giờ chạm tới; phải set tường minh số nhỏ.
>
> **Java 21+:** với workload I/O-bound, `Executors.newVirtualThreadPerTaskExecutor()` bỏ hẳn vấn đề sizing/không có queue; nhưng vẫn cần bounded concurrency ở tầng resource (semaphore, connection pool) và `RejectedExecutionException` vẫn xảy ra nếu dùng `submit` trên executor bị shutdown.

---

## 7) Follow-up / pitfall

- **`Executors.newFixedThreadPool` / `newSingleThreadExecutor` dùng unbounded queue** → không giới hạn được số task pending; đây là lý do phổ biến nhất gây OOM trong service. `newCachedThreadPool` thì ngược lại tạo thread không giới hạn (`SynchronousQueue` + `maximumPoolSize = Integer.MAX_VALUE`) → dễ dẫn tới hàng nghìn thread và rút ruột app.
- **Thứ tự thật không phải "tạo thread tới max rồi mới queue"** — ai cũng nhầm. Luôn là **core → queue → max**.
- **`keepAliveTime` không áp dụng cho core thread** → dùng `allowCoreThreadTimeOut(true)` nếu muốn core thread tự tắt khi rảnh (tiết kiệm tài nguyên cho service biến động).
- **`prestartCoreThread()`** để tránh lần đầu request phải chờ tạo thread (cold start).
- **`CallerRunsPolicy` trên event loop** (Netty/Reactor) → block event loop, gây latency spike toàn hệ thống. Chỉ dùng cho pool chạy ngoài event loop.
- **`shutdown()` vs `shutdownNow()`**: `shutdown()` ngừng nhận task mới nhưng chạy nốt task đang queue; `shutdownNow()` interrupt và trả về task chưa chạy — cần xử lý lại các task đó, nếu không sẽ mất message (với Kafka/SQS consumer cần `ack` thủ công).
- **`submit()` nuốt exception** trong `Future` → task fail âm thầm. Dùng `execute()` + `UncaughtExceptionHandler`, hoặc `submit` + `future.get()`, và luôn bật `setUncaughtExceptionHandler`.
- **Thread bị "chết" sau khi ném `Error`** (vd `OutOfMemoryError`) → thread pool có thể không thay thế thread đó, dẫn tới capacity giảm dần theo thời gian. Cần monitoring `poolSize` vs `corePoolSize`.
- **`ThreadFactory` tùy biến** nên đặt tên thread theo pattern (`worker-1`) và gắn `UncaughtExceptionHandler` — giúp debug stack trace dễ hơn nhiều so với `pool-1-thread-3` mặc định.
- **Câu hỏi xoáy:** "Tăng `maximumPoolSize` có giải quyết được bottleneck không?" → Thường **không**: nếu bottleneck là DB connection lock hoặc downstream latency, thêm thread chỉ làm tăng hàng đợi và áp lực lên hệ thống phía sau.

---

## 8) 30 giây tóm tắt miệng

> "`ThreadPoolExecutor` xử lý task theo thứ tự core thread → đẩy vào queue → queue đầy mới tăng thread tới `maximumPoolSize` → không xong thì reject. Vì `Executors.newFixedThreadPool` dùng `LinkedBlockingQueue` không giới hạn nên `maximumPoolSize` thường không bao giờ được dùng, và task dồn ứ trong queue gây OOM lẫn latency tăng. Cách đúng là tự khởi tạo `ThreadPoolExecutor` với bounded queue, `CallerRunsPolicy` để backpressure (trừ khi chạy trên event loop), và monitoring `activeCount` + `queue.size()`. Với Java 21+ và task I/O-bound thì virtual threads giải quyết phần sizing, nhưng connection pool và backpressure vẫn phải kiểm soát thủ công."
