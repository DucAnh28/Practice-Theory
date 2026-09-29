# CI/CD Pipeline với GitHub Actions

- **Date:** 2026-09-29
- **Topic:** devops
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> "Bạn thiết lập CI/CD pipeline cho một Java Spring Boot service như thế nào? Hãy mô tả từng stage và lý do chọn chúng."

---

## 2) Giải thích như trẻ lên 3

Giống như dây chuyền sản xuất trong nhà máy:
- Code mới vào → máy tự kiểm tra chất lượng (CI)
- Nếu đạt → máy tự đóng gói và chuyển ra kệ hàng (CD)
- Không cần người làm thủ công từng bước, ít lỗi hơn, nhanh hơn.

---

## 3) Giải thích đơn giản cho dev

**CI (Continuous Integration):** Mỗi lần push code, tự động chạy build + test để phát hiện lỗi sớm.  
**CD (Continuous Delivery/Deployment):** Sau khi CI xanh, tự động build Docker image, push registry, deploy lên môi trường target.

GitHub Actions dùng file YAML trong `.github/workflows/`, trigger theo event (push, PR, tag).

---

## 4) Ví dụ code đơn giản

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build & Test
        run: mvn -B verify --no-transfer-progress

      - name: Upload test report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: surefire-reports
          path: target/surefire-reports/

  docker-push:
    needs: build-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build fat JAR
        run: mvn -B package -DskipTests --no-transfer-progress

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myorg/myapp:latest
            myorg/myapp:${{ github.sha }}
```

```dockerfile
# Dockerfile (multi-stage)
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY target/*.jar app.jar

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/app.jar .
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 5) Trả lời Middle+

"CI/CD pipeline của tôi thường có 2 job tách biệt:

**Job 1 — build-and-test:** chạy trên mọi push/PR, dùng `mvn verify` để compile + chạy unit test + integration test. Cache Maven dependencies để giảm thời gian build.

**Job 2 — docker-push:** chỉ chạy khi merge vào `main`, build Docker image với multi-stage để image nhỏ, tag theo `github.sha` để traceable, push lên registry.

Credentials (Docker token, deploy key) lưu trong GitHub Secrets, không hardcode trong workflow."

---

## 6) Trả lời Senior

"Ngoài pipeline cơ bản, ở production tôi bổ sung:

**Security scanning:**
- `trivy` scan Docker image để phát hiện CVE trước khi push
- `OWASP Dependency-Check` hoặc `snyk` scan Maven dependencies

**Test phân tầng:**
- Unit test chạy luôn (nhanh)
- Integration test (Testcontainers) chạy khi build trên CI, skip ở local nếu muốn với `-DskipITs`
- Contract test (Pact) nếu có nhiều service phụ thuộc nhau

**Deployment strategy:**
- Tag image với cả `latest` và `git sha` — `latest` cho deploy nhanh, `sha` để rollback chính xác
- Dùng environment protection rule trên GitHub: branch `main` deploy staging tự động, branch `release/*` cần manual approval để lên prod

**Observability trong pipeline:**
- Upload Surefire/Failsafe reports khi fail để debug ngay trên CI
- Slack/Telegram notification khi pipeline fail trên `main`

**Trade-off cần nhớ:**
- `cache: maven` giúp tăng tốc nhưng đôi khi stale cache làm CI pass còn local fail — cần có job weekly xóa cache
- Multi-stage Dockerfile giảm image size ~3–5x nhưng cần đảm bảo JRE image có đủ tool cần thiết (timezone data, `curl` cho healthcheck)"

---

## 7) Follow-up / pitfall

**Câu hỏi xoáy:**
- "Làm sao bạn đảm bảo image production không chứa source code hay test dependency?"
  → Multi-stage build: stage builder compile, stage final chỉ copy JAR
- "Nếu integration test cần DB thì bạn làm thế nào trên CI?"
  → Dùng `services` block trong GitHub Actions để spin up PostgreSQL/MySQL container, hoặc Testcontainers tự quản lý
- "Làm thế nào rollback khi deploy xong mà bị lỗi?"
  → Tag image theo `git sha`, rollback = redeploy tag cũ; không dùng `latest` để rollback vì `latest` luôn bị ghi đè

**Pitfall hay gặp:**
- Dùng `mvn clean install` trên CI thay vì `mvn verify` → cài artifact vào local repo không cần thiết, tốn thời gian
- Quên `--no-transfer-progress` → log CI toàn progress bar spam, khó đọc
- Hardcode version Java trong Dockerfile không khớp với `setup-java` → build pass CI nhưng runtime khác
- `docker/build-push-action` mặc định push cả `linux/amd64` và `linux/arm64` nếu cấu hình multi-platform — nếu không cần thì thêm `platforms: linux/amd64` để tránh build chậm

---

## 8) 30 giây tóm tắt miệng

"Pipeline của tôi chia 2 phase: CI chạy trên mọi PR — build và test để bắt lỗi sớm; CD chỉ chạy khi merge main — build Docker image multi-stage, tag theo commit SHA, push registry. Secrets lưu trong GitHub Secrets. Ở production còn thêm security scan image với Trivy, manual approval gate trước khi lên prod, và notification khi fail. Điểm quan trọng là tag theo SHA thay vì chỉ dùng latest để rollback được chính xác."
