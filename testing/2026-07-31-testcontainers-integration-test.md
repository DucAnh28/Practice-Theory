# Testcontainers trong Integration Test

- **Date:** 2026-07-31
- **Topic:** testing
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Testcontainers là gì? Khi nào nên dùng thay vì mock database / H2 in-memory? Trade-off chính khi chạy trong CI?

## 2) Giải thích như trẻ lên 3

Mock database giống như giả đồ chơi cho bé — trông giống thật nhưng không có ruột. Testcontainers bê nguyên cái đồ chơi thật (Postgres, Redis, Kafka…) vào phòng thí nghiệm nhỏ, chạy trong Docker, dùng xong thì dẹp. Đắt hơn chút nhưng tin được.

## 3) Giải thích đơn giản cho dev

Testcontainers = thư viện Java (JVM) tự spawn Docker container thật từ `GenericContainer` / các module (`PostgreSQLContainer`, `KafkaContainer`, `RedisContainer`…) cho từng test class hoặc `@BeforeAll`. Lifecycle gắn với JUnit `@Container` (Ryuk tự dọn dẹp khi JVM thoát).

So sánh:

| Phương pháp | Mức tin | Tốc độ | Setup |
|---|---|---|---|
| Mock (Mockito) | Thấp (chỉ behavior) | Nhanh | Dễ |
| H2 in-memory | Trung bình (SQL dialect khác Postgres) | Nhanh | Dễ |
| Embedded (HSQLDB, Embedded Redis) | Trung bình | Nhanh | Dễ |
| Testcontainers (Docker thật) | Cao (cùng engine với prod) | Chậm hơn | Cần Docker |
| Shared DB dev/staging | Rủi ro data, không clean | Nhanh nhất | Khó đảm bảo |

## 4) Ví dụ code đơn giản

```java
@Testcontainers
@SpringBootTest
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("orders")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired OrderRepository repo;

    @Test
    void findByStatus_uses_index() {
        repo.save(new Order("pending"));
        assertThat(repo.findByStatus("pending")).hasSize(1);
    }
}
```

## 5) Trả lời Middle+

> Testcontainers là thư viện JUnit cho phép mỗi integration test tự khởi động một Docker container thật (Postgres, Redis, Kafka…) để chạy test. Mình dùng nó khi cần verify SQL thật, dialect Postgres, JSONB, hoặc message broker — những thứ mock/H2 không tái hiện được. Annotation `@Testcontainers` + `@Container` quản lý lifecycle, Ryuk tự dọn khi test xong.

## 6) Trả lời Senior

- **Vì sao không chỉ mock?** SQL dialect, plan thật, constraint, deadlock — mock giấu hết. Testcontainers chạy cùng engine prod → bug liên quan PG/Redis/Kafka hiện đúng chỗ.
- **Trade-off CI:** cần Docker daemon (DinD cho GitHub Actions: `docker/setup-buildx-action` + `docker.sock` hoặc service container). Startup mỗi container ≈ 5–15s → dùng singleton container (`withReuse(true)`) hoặc `@Container static` để share giữa các test class.
- **Cost:** tốn RAM/CPU. Giải thiểu bằng `testcontainers.reuse.enable=true` và profile riêng.
- **Network:** container test join network của app; dùng `Network.SHARED` khi cần compose nhiều service (Postgres + Redis + Kafka cùng lúc).
- **State & idempotency:** container mới mỗi lần chạy → sạch state, không leak data như shared DB.
- **Wait strategy:** dùng `Wait.forListeningPort()` + log/wait predicate cho service chậm khởi động (Kafka, Elasticsearch).
- **Production concern:** tránh hard-code image version → pin minor (`postgres:16.2-alpine`) để chống surprise.
- **Alternatives khi không có Docker:** LocalStack cho AWS, Toxiproxy cho network failure simulation.

## 7) Follow-up / pitfall

- Không bật Docker trên CI → test fail không phải vì code. Fix: pre-flight check `@EnabledIfSystemProperty(named = "docker.available", matches = "true")`.
- Container khởi động chậm → dùng `@Container static` (chia sẻ) thay vì `instance` (mỗi test mới).
- Image pull lặp đi lặp lại → bật reuse mode, hoặc pre-pull trong CI cache.
- Dynamic property đặt trước Spring context init → chỉ dùng `@DynamicPropertySource`, không hard-code `application-test.yml`.
- DB migration (Flyway/Liquibase) chạy trên container thật → phát hiện bug SQL mà H2 bỏ qua.

## 8) 30 giây tóm tắt miệng

> Testcontainers là thư viện JUnit sinh Docker container thật cho integration test. Mình dùng khi cần verify Postgres, Redis, Kafka y như production — những thứ H2 hay mock không bắt được. Trade-off là cần Docker trên CI và chậm hơn mock, nên thường share container tĩnh giữa các test, kết hợp `@DynamicPropertySource` để Spring Boot nối vào. Khi CI không có Docker thì fallback sang H2 hoặc gate test bằng `@EnabledIf`.
