# Docker Registry và Tagging Strategy

- **Date:** 2026-10-21
- **Topic:** devops
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> "Bạn quản lý Docker image trong registry như thế nào? Tag theo version hay commit hash? Làm sao biết image nào đang chạy production?"

---

## 2) Giải thích như trẻ lên 3

Giống như kho hàng có nhiều phiên bản sản phẩm:
- Mỗi lần làm sản phẩm mới, gắn tem ghi rõ số phiên bản và ngày sản xuất
- Tem `latest` = phiên bản mới nhất, nhưng không biết chính xác là bản nào
- Tem có số (v1.2.3) hoặc mã (abc123) = biết chính xác bản đó, lấy lại được nếu cần

---

## 3) Giải thích đơn giản cho dev

Docker registry (Docker Hub, ECR, GCR, Harbor) lưu image, mỗi image có nhiều tag.

**Tag phổ biến:**
- `latest` — image mới nhất, dễ pull nhưng không biết version cụ thể
- `v1.2.3` — semantic version, rõ ràng cho release
- `abc1234` — git commit SHA ngắn, trace được đúng code gốc
- `main-abc1234` — branch + commit, biết build từ nhánh nào

**Vấn đề `latest`:** có thể bị ghi đè, rollback không an toàn. Production nên dùng tag bất biến (immutable).

---

## 4) Ví dụ code đơn giản

```yaml
# .github/workflows/docker-push.yml
- name: Extract metadata
  id: meta
  run: |
    BRANCH=${GITHUB_REF#refs/heads/}
    SHORT_SHA=${GITHUB_SHA::7}
    echo "branch=$BRANCH" >> $GITHUB_OUTPUT
    echo "sha=$SHORT_SHA" >> $GITHUB_OUTPUT
    
- name: Build & push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      myorg/myapp:${{ steps.meta.outputs.branch }}-${{ steps.meta.outputs.sha }}
      myorg/myapp:${{ steps.meta.outputs.sha }}
      myorg/myapp:latest
```

**Kết quả:**
```
myorg/myapp:main-a3f7c21
myorg/myapp:a3f7c21
myorg/myapp:latest
```

**Semantic version tag (từ Git tag):**
```yaml
on:
  push:
    tags:
      - 'v*.*.*'

- name: Extract version
  id: version
  run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

- name: Build & push
  uses: docker/build-push-action@v5
  with:
    tags: |
      myorg/myapp:${{ steps.version.outputs.VERSION }}
      myorg/myapp:latest
```

**Pull image cụ thể trong Kubernetes:**
```yaml
# deployment.yaml
spec:
  containers:
  - name: app
    image: myorg/myapp:a3f7c21  # immutable tag
    imagePullPolicy: IfNotPresent
```

---

## 5) Trả lời Middle+

"Tôi tag image theo 3 cấp độ:

**1. Commit SHA (`a3f7c21`)** — mỗi push CI tự động tag theo git commit. Đây là tag bất biến, dùng để trace lại code gốc và rollback chính xác.

**2. Branch + SHA (`main-a3f7c21`)** — biết image build từ nhánh nào, hữu ích khi có nhiều nhánh (dev, staging, main).

**3. `latest`** — tag động, luôn trỏ image mới nhất. Dùng cho development hoặc testing, không dùng production.

Khi deploy production, chỉ định tag commit SHA cụ thể trong manifest Kubernetes hoặc docker-compose, không dùng `latest`. Rollback = đổi tag về SHA cũ và redeploy."

---

## 6) Trả lời Senior

"Ngoài tagging strategy, tôi chú ý:

**Registry lifecycle policy:**
- Docker Hub giới hạn image pulls và storage → dùng retention policy xóa image cũ sau 90 ngày hoặc giữ 10 tags gần nhất
- ECR/GCR hỗ trợ lifecycle rule: xóa untagged image (dangling layers), giữ 5 tag mới nhất cho mỗi prefix

**Immutable tags:**
- ECR cho phép bật image tag immutability → một tag chỉ push được 1 lần, không ghi đè
- Trade-off: an toàn hơn nhưng không thể sửa tag nếu push nhầm, phải xóa và push lại

**Semantic versioning cho release:**
- Khi tạo Git tag `v1.2.3`, CI tự tag image tương ứng
- Dùng cho customer-facing release, dễ communicate với team khác (PM, QA)
- Commit SHA vẫn giữ để developer trace code

**Security scanning integration:**
- Scan image ngay sau build (Trivy, Clair, Snyk)
- Nếu phát hiện critical CVE, block push hoặc gắn label `vulnerable: true`
- Có team dùng admission controller (OPA, Kyverno) chặn deploy image có CVE critical

**Multi-registry strategy:**
- Môi trường regulated: mirror image từ public registry (Docker Hub) sang private registry (Artifactory, Nexus) để kiểm soát supply chain
- Cache image gần cluster (regional registry) để pull nhanh, giảm egress cost

**Audit log:**
- Track image nào deploy lên production qua annotation trong Kubernetes Deployment hoặc log trong CD tool (ArgoCD, Flux)
- Dùng `kubectl describe deployment` + `image:` field hoặc label `app.version: v1.2.3`

**Trade-off thực tế:**
- Tag nhiều = dễ trace nhưng tốn storage và phức tạp hơn khi query
- Semantic version đẹp nhưng cần discipline: phải bump version đúng theo convention
- Immutable tag an toàn nhưng khi hotfix gấp, không thể force-push `latest` được nữa"

---

## 7) Follow-up / pitfall

**Câu hỏi xoáy:**
- "Nếu image `latest` bị ai đó vô tình push lên production, làm sao phát hiện và ngăn chặn?"
  → Dùng admission controller (Kyverno policy) chặn deploy image tag `latest`; monitoring alert khi image deployed không match pattern `vX.Y.Z` hoặc SHA
  
- "Làm thế nào biết image nào đang chạy trên production nếu chỉ nhìn registry?"
  → Không thể biết chỉ từ registry. Phải check Kubernetes deployment manifest (`kubectl get deploy -o yaml`) hoặc logs từ CD system (ArgoCD sync history)

- "Registry bị full, bạn xử lý thế nào?"
  → Cấu hình lifecycle policy xóa image cũ; check dangling image (`docker image prune`); nếu dùng ECR/GCR thì bật auto cleanup untagged images

**Pitfall hay gặp:**
- Push nhiều tag cùng lúc trong CI nhưng một tag fail → image có một số tag thiếu, gây confusion
- Dùng `imagePullPolicy: Always` với tag SHA → mỗi lần restart pod đều pull lại, tốn bandwidth và chậm; nên dùng `IfNotPresent` vì SHA immutable
- Quên xóa untagged images (dangling layers từ build cũ) → registry phình to, tốn tiền storage
- Tag semantic version không sync với Git tag → image `v1.2.3` không match commit thực tế, khó trace
- Dùng `latest` trong production rồi quên → đột nhiên pod restart pull image mới, gây incident

---

## 8) 30 giây tóm tắt miệng

"Tôi tag image theo commit SHA để trace code gốc, thêm branch prefix nếu cần, và tag semantic version cho release chính thức. Production luôn deploy tag bất biến (SHA hoặc version cụ thể), không dùng `latest` vì nó thay đổi liên tục. Registry có lifecycle policy để xóa image cũ tự động, giảm storage cost. Nếu môi trường yêu cầu cao, bật immutable tag ở registry level và dùng admission controller chặn deploy image không đúng pattern."
