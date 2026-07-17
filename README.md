# Practice Interview — Java Developer

Luyện phỏng vấn Java (Middle+ / Senior). Mỗi ngày làm việc (T2–T6) 1 note ngắn, push theo topic folder.

## Cấu trúc

| Folder | Nội dung |
|--------|----------|
| `java-core/` | Language, OOP, equals/hashCode, generics… |
| `collections/` | List/Map/Set, complexity, thread-safe collections |
| `concurrency/` | Thread, lock, concurrent utils, CompletableFuture |
| `jvm-gc/` | Memory model, GC, tuning cơ bản |
| `java-spring/` | Spring Boot, DI, Bean, AOP, Transaction |
| `jpa-hibernate/` | JPA, N+1, fetch, cache |
| `sql/` | Index, transaction isolation, query |
| `kafka/` | Topic, partition, consumer group, offset |
| `redis/` | Cache, data structure, eviction |
| `messaging/` | Queue pattern, RabbitMQ/Kafka so sánh |
| `microservices/` | SAGA, circuit breaker, idempotency |
| `system-design/` | Design API/service ngắn |
| `security/` | JWT, OAuth2, common pitfalls |
| `testing/` | Unit/integration, Testcontainers |
| `devops/` | Docker, CI cơ bản cho BE |

## Format mỗi file

```text
YYYY-MM-DD-topic-slug.md
```

Nội dung:

1. Câu hỏi phỏng vấn
2. Giải thích đơn giản
3. Trả lời Middle+
4. Trả lời Senior
5. Follow-up / pitfall

## Automation

OpenClaw cron: **T2–T6**, timezone `Asia/Saigon`, tạo 1 note/ngày → commit + push repo này → gửi tóm tắt Telegram.

## Rotation

Thứ tự topic xoay vòng (xem `.state/rotation.json`). Tránh lặp topic liên tiếp khi có thể.
