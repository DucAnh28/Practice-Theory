# Docker Multi-stage Build

- **Date:** 2026-08-24
- **Topic:** devops
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Multi-stage build trong Docker là gì? Tại sao production image cho Java service nên dùng multi-stage thay vì build và chạy chung một image?

## 2) Giải thích đơn giản

Nấu ăn đời thường: bạn cần 1 bữa sạch sẽ để mang đi (image chạy production), nhưng khi nấu thì đầy bếp, dính dầu mỡ, có cả nồi xoong chất đống (tool build, JDK, source, cache).

Cách 1 (1 stage): lấy nguyên cả bếp bẩn đóng hộp mang đi → to, nặng, có đồ thừa, có thể dính secrets.

Cách 2 (multi-stage): dùng 1 "bếp tạm" để nấu xong, chỉ chuyển phần thành phẩm (JAR) sang 1 "hộp sạch" nhỏ gọn để mang đi. Bếp tạm vứt đi luôn.

Trong Docker, "bếp tạm" = stage build (có JDK, Maven, source). "Hộp sạch" = stage runtime (chỉ JRE + JAR + app code). Kết quả: image nhỏ, ít attack surface, ít CVE.

## 3) Giải thích đơn giản cho dev

Multi-stage build cho phép 1 `Dockerfile` định nghĩa nhiều `FROM ... AS <name>`. Mỗi stage là một image tạm, có layer riêng. Stage sau `COPY --from=<stage>` chỉ lấy artifact cần thiết từ stage trước, phần thừa (compiler, build cache, source code, `.git`, test) bị bỏ lại.

So sánh image Java điển hình:

| Cách | Base image | Cỡ thường thấy | Gồm gì |
|------|------------|----------------|--------|
| 1-stage | `maven:3.9-eclipse-temurin-17` | ~700MB–1GB | JDK, Maven, `.m2`, source, dependency, JAR |
| Multi-stage runtime | `eclipse-temurin:17-jre` | ~180–250MB | JRE + JAR + app code |

Lợi ích chính:
- Image nhỏ hơn nhiều → pull nhanh, start nhanh, ít chiếm registry.
- Bề mặt tấn công nhỏ: không có compiler, shell tool thừa, không có source/test.
- Bớt leak secret: file `.env`, token Maven, key test nằm ở stage build, không vào image cuối.
- Cache layer tốt hơn vì tách layer source/build ra khỏi layer dependency.

## 4) Ví dụ code đơn giản

```dockerfile
# syntax=docker/dockerfile:1.6

# ---- Stage build: JDK + Maven, sinh ra JAR ----
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /workspace

# Copy pom trước để cache dependency layer
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -e -ntp dependency:go-offline

COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -e -ntp -DskipTests package

# ---- Stage runtime: chỉ JRE + JAR ----
FROM eclipse-temurin:17-jre-jammy AS runtime

# User non-root, an toàn hơn root
RUN groupadd --system app && useradd --system --gid app --home /app app
WORKDIR /app
USER app

COPY --from=build /workspace/target/*.jar /app/app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-XX:+ExitOnOutOfMemoryError", "-jar", "/app/app.jar"]
```

`--mount=type=cache` (BuildKit) mount cache Maven ra ngoài layer, lần build sau không tải lại dependency.

## 5) Trả lời Middle+

Multi-stage build là kỹ thuật tách Dockerfile thành nhiều stage; stage sau chỉ `COPY --from=<stage>` lấy artifact cần thiết. Với Java, build stage dùng image có JDK + Maven/Gradle để compile và đóng gói JAR; runtime stage dùng image JRE nhỏ (ví dụ `eclipse-temurin:17-jre`) chỉ chứa JRE, JAR và config. Kết quả là image production nhỏ hơn 3–5 lần, pull/deploy nhanh hơn, ít CVE vì không mang compiler và tool build, và giảm rủi ro lộ source/secret vì chúng chỉ tồn tại ở stage build.

## 6) Trả lời Senior

Trade-off và concern production cần nhắc:

1. **Reproducibility**: lock version JDK và JRE ở 2 stage cho cùng major (Temurin 17 ↔ Temurin 17-jre). Đừng dùng `latest`, đừng trộn vendor khác nhau giữa build và run.
2. **Reproducible build nâng cao**: bật BuildKit (`DOCKER_BUILDKIT=1`), dùng `--mount=type=cache` cho `.m2`/`.gradle` để tăng tốc và ổn định layer cache; cân nhắc `--mount=type=secret` cho token private registry thay vì `ARG`.
3. **Layer cache hygiene**: copy `pom.xml`/`build.gradle` trước `src/` để dependency layer cache ổn định; thay đổi source không phải tải lại dependency.
4. **Security**: chạy non-root (như ví dụ), drop capabilities, đặt `HEALTHCHECK`, dùng distroless hoặc `*-jre` slim thay vì full JDK. Scan CVE bằng Trivy/Snyk trong CI; pin digest (`@sha256:...`) thay vì tag mutable khi cần audit nghiêm.
5. **Observability**: thêm `-XX:+ExitOnOutOfMemoryError`, `-XX:+HeapDumpOnOutOfMemoryError`, mount volume cho heap dump, expose actuator/Prometheus nếu có.
6. **JVM trong container**: bật `UseContainerSupport` (mặc định từ JDK 10+), set explicit `JAVA_TOOL_OPTIONS`/`-Xmx` để JVM thấy cgroup limit, tránh OOM kill khi K8s chỉ định memory request/limit.
7. **Bề mặt giữ lại cố ý**: nếu cần `jstack`/`jcmd` lúc troubleshoot, có thể giữ `jlink` minimal JDK runtime thay vì JRE thuần; chấp nhận +20–40MB để đổi lấy khả năng debug live.

Một lỗi thường gặp: vẫn để `COPY . .` ở build stage và quên filter `.dockerignore` (khiến `.git`, `.env`, `target`, `node_modules` vào context, phình build context); dùng `openjdk` thay vì `-jre` làm runtime; chạy với `USER root`; quên set JVM heap theo container limit.

## 7) Follow-up / pitfall

- "Multi-stage có làm build chậm hơn không?" → Không đáng kể; cache layer giúp build lại nhanh. Tổng thời gian CI thường giảm vì pull base nhỏ hơn.
- "Có cần distroless không?" → Distroless (`gcr.io/distroless/java17`) cắt thêm shell, gọn hơn `jre` nhưng debug khó hơn. Middle: dùng `*-jre-jammy`. Senior: chuyển sang distroless khi security gate bắt buộc.
- "Build context quá lớn?" → Viết `.dockerignore`: `.git`, `.idea`, `target`, `build`, `*.log`, `.env*`, `node_modules`.
- "Tại sao image vẫn to dù đã multi-stage?" → Có thể do giữ dependency không cần thiết (`provided` scope vẫn vào JAR), dùng `spring-boot-starter` kéo cả Tomcat/Actuator mặc dù chạy trên K8s. Cân nhắc `spring-boot:build-image` (CNB) hoặc tách layer dependencies/application để diff thay đổi ít tải lại.
- "Khác gì với `spring-boot:build-image`?" → CNB tạo image multi-layer tối ưu sẵn cho Spring Boot; vẫn dùng concept multi-stage bên dưới. Middle chọn cách nào cũng được; Senior hiểu bên dưới nó là gì để debug khi build fail.

## 8) 30 giây tóm tắt miệng

"Multi-stage build tách Dockerfile thành 2 phần: stage build dùng JDK + Maven để compile và đóng gói JAR, stage runtime chỉ chứa JRE + JAR. Nhờ `COPY --from=build` chỉ lấy artifact cần thiết, image production nhỏ hơn 3–5 lần, ít CVE hơn vì không mang compiler, và giảm rủi ro lộ source hay secret. Mình kết hợp `eclipse-temurin:17-jre-jammy` làm runtime, BuildKit cache mount cho `.m2`, chạy non-root, set `JAVA_TOOL_OPTIONS` để JVM tôn trọng cgroup limit, và pin digest trong CI để audit."
