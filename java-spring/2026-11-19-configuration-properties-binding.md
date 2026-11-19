# Spring Boot — `@ConfigurationProperties` vs `@Value` và validate config

- **Date:** 2026-11-19
- **Topic:** java-spring
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> Trong Spring Boot, khi nào dùng `@Value` và khi nào dùng `@ConfigurationProperties`? Làm sao để config sai (thiếu field, sai kiểu dữ liệu, thiếu biến môi trường bắt buộc) bị phát hiện **lúc khởi động** chứ không phải lúc request đến?

---

## 2) Giải thích như trẻ lên 3

Tưởng tượng cấu hình nhà bếp:
- `@Value` là **dán từng nhãn**: `@Value("${fuel.level}")` — bạn phải biết trước tên biến và gõ tay mỗi chỗ.
- `@ConfigurationProperties` là **một tờ giấy khai đủ thông tin**: viết một class `FuelConfig` gồm `level`, `type`, `capacity` → Spring tự đổi toàn bộ cấu hình prefix thành 1 object, kiểu an toàn, có tên rõ ràng.

Nếu bán xăng mà không có xăng, quầy lỗi ngay lúc mở cửa (validate lúc start) thay vì để khách đến rồi mới lộn xộn.

---

## 3) Giải thích đơn giản cho dev

**`@Value`** — chỉ là text substitution:

```java
@Value("${app.timeout-ms}")
private int timeoutMs;                 // không có type-safe, không group, không validate
```

- Phải khai báo lại từng biến ở mọi nơi dùng.
- Không biết trước giá trị mặc định hay không.
- Refresh (`@RefreshScope`) hoạt động, nhưng nhiều bean không refresh được.
- File YAML/JSON không map được vào object có kiểu.

**`@ConfigurationProperties`** — bind prefix thành object:

```java
@ConfigurationProperties(prefix = "app")
@Validated                       // bắt buộc để validate
public record AppProperties(
        @Min(100) @Max(60_000) int timeoutMs,
        @NotBlank String mode,
        @DefaultValue("false") boolean async
) {}
```

**Cơ chế bind (relaxed binding):**
- `app.timeout-ms` ↔ `app.timeoutMs` ↔ `APP_TIMEOUT_MS` (env var) ↔ `APP_TIMEOUTMS`
- Ưu tiên nguồn: **env var / system property > application.yml > @Value default**.
- Kiểu được ép an toàn: `"true"`, `"123"`, `"10s"` (Spring Boot 2+ hỗ trợ `Duration`), list `"a,b,c"`.

**Validate:**
- Cần **`@Validated`** trên class + constraint annotation trên field (`@Min`, `@NotBlank`, `@Size`…) → dùng `spring-boot-starter-validation`.
- Không có `@Validated` thì constraint **không được kiểm tra** — đây là cái bẫy phổ biến nhất.

---

## 4) Ví dụ code đơn giản

```yaml
# application.yml
app:
  timeout-ms: 5000
  mode: async
  retry:
    max-attempts: 3
```

```java
// 1) Class dùng setter + @ConfigurationProperties
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    @Min(100) @Max(60_000)
    private int timeoutMs = 5000;

    @NotBlank
    private String mode = "sync";

    public int getTimeoutMs() { return timeoutMs; }
    public void setTimeoutMs(int v) { this.timeoutMs = v; }
    public String getMode() { return mode; }
    public void setMode(String v) { this.mode = v; }
}
```

```java
// 2) Kích hoạt: @EnableConfigurationProperties(AppProperties.class)
//    hoặc @ConfigurationPropertiesScan trên application class
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```java
// 3) Inject như một bean bình thường
@Service
public class PaymentService {
    private final AppProperties props;

    public PaymentService(AppProperties props) {   // constructor injection
        this.props = props;
    }

    public void pay() {
        long t = props.getTimeoutMs();
        System.out.println("timeout=" + t + ", mode=" + props.getMode());
    }
}
```

```java
// 4) Test: dùng @EnableConfigurationProperties + ApplicationContextRunner (nhanh, không cần context đầy đủ)
new ApplicationContextRunner()
    .withUserConfiguration(TestConfig.class)
    .withPropertyValues("app.timeout-ms=50")     // vi phạm @Min(100)
    .run(ctx -> assertThat(ctx).hasFailed());      // fail lúc startup
```

---

## 5) Trả lời Middle+

> Tôi dùng `@Value` cho **số ít** giá trị đơn lẻ (port, feature flag 1 dòng), còn `@ConfigurationProperties` cho **nhóm config** — vì nó type-safe, gom được thành 1 object, có tên rõ ràng, dễ test và dễ thêm constraint. Để validate thì bắt buộc có **cả hai**: `@Validated` trên class và `@EnableConfigurationProperties`/`@ConfigurationPropertiesScan` để class được đăng ký. Nếu thiếu `@Validated`, các annotation như `@Min`/`@NotNull` sẽ **không bao giờ được kiểm tra** — đây là lỗi tôi hay gặp nhất. Ngoài ra Spring Boot hỗ trợ bind kiểu `Duration`/`DataSize` (`30s`, `512MB`) nên không cần tự parse string.

---

## 6) Trả lời Senior

> **Vì sao validate lúc startup quan trọng:** cấu hình sai là **failure mode im lặng** — app vẫn boot, traffic vẫn vào, nhưng timeout = 0, pool size âm, endpoint sai → lỗi rải rác ở production. `ConfigurationPropertiesBindException` / `BindValidationException` lúc startup đổi "sập lúc chạy" thành "fail fast lúc deploy" — tốt hơn nhiều.
>
> **Quan trọng: `@Value` không hỗ trợ validation.** Không có cách nào để Spring kiểm tra `@Value("${app.timeout}")` có hợp lệ hay không. Đây là lý do chính để migrate khỏi `@Value`.
>
> **Type-safe với `record` + `@ConstructorBinding`:** record bắt buộc dùng constructor binding, không cần setter, không có trạng thái nửa vời → an toàn với multithread. Với class có setter, có nguy cơ bean bị mutate sau khi inject.
>
> **Config secret:** không bind password/token trực tiếp vào record dạng `String`; dùng `SecretKey`/`String` từ Vault/KMS/ENV, và tuyệt đối không log `toString()` của object chứa secret (record tự sinh `toString` in ra plaintext!). Đây là lỗi phổ biến khi chuyển từ class sang record.
>
> **Profile & môi trường:** `application-{profile}.yml` + `@Profile`; cần "bắt buộc theo môi trường" thì dùng **profile group** (`spring.profiles.group.prod=prod,cloud`) để config validate bằng cùng một bộ rule, tránh mỗi env một bộ validate riêng.
>
> **Refresh runtime:** `@ConfigurationProperties` + `@RefreshScope` cần bean là **proxy** (`spring.cloud.context.refresh.enabled`), và bean có thể bị re-create → không nên giữ reference cứng tới nó ở nơi khác. Với `@Value` trên `final` field thì **không refresh được**. Nếu cần reload nhẹ thì `EnvironmentChangeEvent` + `Environment` là nguồn sự thật.
>
> **Observability:** expose cấu hình hiện tại (đã mask secret) qua Actuator `/actuator/configprops` — giúp debug "tại sao chạy production lại lấy giá trị dev". Audit log lại config đọc từ đâu (env vs file) khi deploy.
>
> **Type coercion & quota:** Spring ép `"123"` → `int`; nhưng `"12abc"` sẽ fail startup (tốt). Cần ràng buộc đơn vị dùng `Duration`/`DataSize` thay vì số trần để tránh bug "milliseconds vs seconds".

---

## 7) Follow-up / pitfall

- **Thiếu `@Validated`** → constraint không bao giờ chạy, ai cũng tưởng đã validate. Đây là pitfall số 1.
- **Quên `@EnableConfigurationProperties` / `@ConfigurationPropertiesScan`** → field luôn null/0, không có lỗi gì cả (silent). Dùng `@ConfigurationPropertiesScan` ở package gốc cho gọn.
- **Dùng `@Value("${app.timeout-ms}")` cho `int`** → lỗi type chỉ lộ khi app chạy tới chỗ đó.
- **Đặt `@Value` trên `final` field** → không refresh được (Spring không set final field khi refresh).
- **`@Value` với property không tồn tại** → fail startup với `IllegalArgumentException`; nếu muốn optional phải ghi default `${x:defaultValue}`.
- **Biểu thức SpEL trong `@Value`**: `"#{${app.mode} == 'async'}"` — dễ lỗi, và SpEL không lazy theo property refresh.
- **Prefix không khớp** → `app.timeout-ms` không map vào `timeoutMs` nếu bạn đặt sai tên field (vd `timeoutMS`), sẽ ra lỗi "not bound" nhưng người mới hay tưởng là Spring bug.
- **List/Map config**: dùng `List<String>` hoặc `Map<String,String>`; cấu hình list phải đặt dạng `- item` trong YAML, không phải `"a,b,c"` (khi cần bind list từ env thì `APP_ROOTS=a,b,c` vẫn được).
- **Binder khác nhau giữa version**: Spring Boot 2.x dùng `RelaxedPropertyResolver` cũ; Boot 3 đổi cơ chế bind + constructor binding cho record → cẩn thận khi nâng version, đặc biệt với `@ConstructorBinding` trên class có nhiều constructor.
- **`@ConfigurationProperties` trên class có `@Component`** → chạy được nhưng làm mất tính năng `configuration-processor`/metadata và khó quản lý; nên tách riêng và dùng `@EnableConfigurationProperties`.
- **Record + secret**: `toString()` của record in ra plaintext → không log cả object config.
- **Câu hỏi xoáy:** "Config nên đổi runtime hay cần restart?" → Trả lời: config hạ tầng (port, URL) nên restart để an toàn; chỉ flag/limit mới hot-reload, và hot-reload cũng cần audit + rollback.

---

## 8) 30 giây tóm tắt miệng

> "`@Value` chỉ nên dùng cho vài giá trị đơn lẻ, còn config nhóm thì dùng `@ConfigurationProperties` vì type-safe, gom gọn và hỗ trợ validate. Muốn validate thì cần **cả hai**: `@Validated` trên class và `@EnableConfigurationProperties` hoặc `@ConfigurationPropertiesScan` để đăng ký bean — thiếu một trong hai là validation không chạy hoặc bean không được tạo. Giá trị sai sẽ làm app **fail lúc startup** thay vì lỗi rải rác lúc chạy. Nếu dùng record thì bắt buộc constructor binding và phải cẩn thận `toString()` làm lộ secret. Với runtime refresh thì `@RefreshScope` cần proxy, và `@Value` trên field `final` sẽ không refresh được."
