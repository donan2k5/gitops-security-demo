# Lab 2.2 · Trivy + Cosign + Policy Controller — giải thích chi tiết

Giả định bạn chưa biết các thuật ngữ dưới đây — mỗi mục đều giải thích bằng
ví dụ trước khi đi vào kỹ thuật. Mục tiêu cuối: cluster **chỉ chạy** image
đã scan sạch CVE nặng và đã được ký, mọi thứ khác bị admission chặn.

## 0. Vài thuật ngữ nền tảng

- **Image layer**: một Docker image không phải 1 file đặc — nó là 1 chuỗi
  các "lớp" (layer), mỗi lớp là 1 file `.tar.gz` chứa phần thay đổi filesystem
  (ví dụ lớp 1 = hệ điều hành gốc, lớp 2 = cài Python, lớp 3 = copy code app).
  Khi `docker build`, mỗi dòng `RUN`/`COPY` trong Dockerfile thường tạo ra 1
  layer mới.
- **Manifest**: 1 file JSON nhỏ mô tả "image này gồm những layer nào, theo
  thứ tự nào, layer nào có hash gì". Khi bạn `docker pull`, registry trả
  manifest này trước, Docker mới biết cần kéo những layer nào.
- **Digest**: chính là `sha256(manifest JSON)` — một chuỗi hash cố định,
  dạng `sha256:abcd1234...`. Khác với **tag** (`:latest`, `:0.0.3` — chỉ là
  1 cái tên do người gắn, có thể bị đổi để trỏ sang image khác bất kỳ lúc
  nào), digest là bất biến: nội dung image đổi 1 byte thôi, digest đổi
  ngay. Đây là lý do mọi thứ "bảo mật" (ký, verify) trong bài này đều dùng
  digest, không dùng tag.
- **OCI registry**: chính là GHCR (`ghcr.io`) ở đây — về bản chất là 1 web
  server hiểu 1 chuẩn API tên "OCI Distribution Spec": nhận `PUT` để lưu
  layer/manifest, trả `GET` theo digest hoặc tag. Cosign tận dụng đúng API
  này để lưu **chữ ký** — chữ ký cũng chỉ là 1 "manifest" khác nằm cạnh
  manifest của image, không phải cơ chế riêng.
- **Admission webhook**: xem lại [`docs/gatekeeper.md`](gatekeeper.md) mục 6.3
  — cùng cơ chế, chỉ khác code chạy bên trong webhook là của Sigstore
  policy-controller thay vì Gatekeeper.

## 1. CI làm gì, theo đúng code trong `build-push.yml`

### Bước 1 — build local, chưa push đi đâu cả

```yaml
- name: Build image (local, not pushed yet)
  uses: docker/build-push-action@v6
  with:
    context: ./src/api
    push: false
    load: true
    tags: ${{ steps.meta.outputs.tags }}
```

`push: false` + `load: true` nghĩa là: build xong, **nạp image vào Docker
daemon của chính runner GitHub Actions** (1 máy ảo tạm, sống trong vài phút
rồi bị xoá), không đẩy đi đâu cả. Nếu đảo ngược thứ tự — push trước, scan
sau — thì ngay cả khi job fail ở bước scan, image **đã kịp nằm trên
registry công khai** rồi, ai có quyền pull vẫn lấy được image chứa CVE nặng
trong lúc job đang chạy. Build local trước giải quyết đúng vấn đề này: image
chỉ tồn tại trong máy ảo tạm, biến mất khi job xong, không ai ngoài thấy nó
nếu scan fail.

### Bước 2 — Trivy quét image local đó, fail job nếu có CVE nặng

```yaml
- name: Scan image with Trivy (fail on HIGH/CRITICAL)
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}
    severity: "HIGH,CRITICAL"
    exit-code: "1"
    ignore-unfixed: false
```

Trivy không "chạy" image lên để xem nó làm gì — nó **bóc** từng layer
(đọc các file `.tar.gz` tạo ra ở bước build), liệt kê toàn bộ package đã
cài (gói hệ điều hành như `apt`/`apk`, và dependency ngôn ngữ như file
`requirements.txt` đã COPY vào image), rồi so từng cặp `(tên package,
version)` với 1 database CVE mà Trivy tự tải về (nguồn từ NVD, GitHub
Security Advisory...). `severity: "HIGH,CRITICAL"` nghĩa là chỉ CVE được
chính database đó gắn nhãn mức này mới tính; `exit-code: "1"` là cách Trivy
CLI báo cho GitHub Actions "có vấn đề, dừng pipeline lại" — không có field
này, Trivy chỉ in cảnh báo ra log rồi vẫn để job pass như thường.

### Bước 3 — chỉ khi scan pass mới push thật, lấy digest

```yaml
- name: Push Docker image
  id: build
  uses: docker/build-push-action@v6
  with:
    context: ./src/api
    push: true
    tags: ${{ steps.meta.outputs.tags }}
```

Step này có `id: build` — sau khi `docker/build-push-action` `PUT` xong
layer + manifest lên GHCR, registry trả lại digest của manifest đó trong
response, action lưu vào output `digest`, dùng được ở các step sau qua
`steps.build.outputs.digest`.

### Bước 4 — ký theo digest, không theo tag

```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@v3

- name: Sign image digest with Cosign
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
  run: |
    cosign sign --yes --key env://COSIGN_PRIVATE_KEY \
      "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}"
```

Cú pháp `@${{ steps.build.outputs.digest }}` (tương đương
`@sha256:abcd1234...`) — dùng `@digest` thay vì `:0.0.3` — nói với cosign
"ký đúng bản build vừa tạo ra ở Bước 3, không phải bất kỳ thứ gì đang/sẽ
được gắn tag `0.0.3` trong tương lai" (xem lý do kỹ hơn ở mục "Vì sao ký
theo digest" cuối file).

## 2. `cosign sign` thực chất làm gì dưới OCI registry

Đây là phần hay bị tưởng nhầm là "phép màu". Thực tế chỉ có 3 việc:

1. Cosign tính lại digest của image (`sha256:<digest>`), không tin digest do
   ai khai báo — tự pull manifest về tính hash để chắc chắn.
2. Cosign dùng **private key** (file `cosign.key`, là 1 cặp khoá ECDSA —
   thuật toán mã hoá bất đối xứng, giống RSA nhưng khoá ngắn hơn) để tạo ra
   1 **chữ ký số** (signature) trên chuỗi digest đó. Về bản chất đây là
   phép toán: `signature = sign(private_key, digest)` — chỉ ai có đúng
   private key mới tạo được signature khớp với 1 digest cụ thể; verify thì
   không cần private key, chỉ cần public key.
3. Cosign **đẩy chữ ký này lên chính registry**, dưới dạng 1 manifest mới,
   nằm cùng repository `ghcr.io/<owner>/w10-api`, nhưng dùng 1 **tag được
   suy ra từ digest gốc**: `sha256-<digest-hex>.sig`. Bạn có thể tự xem nó:
   ```bash
   crane manifest ghcr.io/<owner>/w10-api:sha256-<digest-hex>.sig
   ```
   Manifest này có 1 layer chứa đúng giá trị `signature` (Base64) +
   payload mô tả digest nào được ký + (tuỳ chọn) certificate nếu dùng
   keyless signing (repo này dùng key thường, không dùng keyless/Rekor nên
   bỏ qua phần certificate).

→ Cosign **không sửa gì trên image gốc**, không cần quyền viết đặc biệt
ngoài quyền push registry thông thường — "ký" chỉ là push thêm 1 object nhỏ
bên cạnh, registry chỉ thấy đó là một manifest khác như mọi manifest khác.

## 3. Policy Controller verify như thế nào lúc Pod được tạo

### 3.1. Webhook chỉ được gọi cho namespace có label — lọc ngay tại tầng API server

Khác với Gatekeeper (namespace bị loại trừ bằng logic Rego *bên trong*
webhook, xem [`docs/gatekeeper.md`](gatekeeper.md) mục 6.3), Sigstore policy-controller cấu
hình ngay trong chính `ValidatingWebhookConfiguration` (do Helm chart tạo
lúc cài, App `policy-controller`, sync-wave `-2`) một field tên
`namespaceSelector`:

```yaml
namespaceSelector:
  matchExpressions:
    - key: policy.sigstore.dev/include
      operator: In
      values: ["true"]
```

`kube-apiserver` đọc field này **trước khi** quyết định có gọi webhook hay
không — nếu namespace đang tạo Pod không có label `policy.sigstore.dev/include:
"true"`, API server **thậm chí không gửi request HTTP nào tới
policy-controller**. Đây là lý do file `policies/cluster-image-policy.yaml`
mở đầu bằng comment:

```yaml
# ClusterImagePolicy: chi enforce tren namespace co label policy.sigstore.dev/include=true
# Khong gan label cho namespace demo trong GitOps - xem docs/runbook-image-sign-verify.md
# (phai gan SAU khi image dau tien da duoc ky, neu khong se chan luon app dang chay)
```

— chưa label namespace thì `ClusterImagePolicy` tồn tại trong cluster nhưng
**không ai gọi tới nó cả**, hoàn toàn vô hại; gắn label vào là bật cơ chế
lọc ở tầng API server, ảnh hưởng ngay tức khắc tới mọi Pod sắp tạo trong
namespace đó.

### 3.2. Trong webhook: resolve tag → digest → tải chữ ký → verify

Khi webhook được gọi (namespace đã có label), policy-controller nhận 1
`AdmissionReview` JSON — định dạng giống y hệt mục 6.3 của
[`docs/gatekeeper.md`](gatekeeper.md) (cũng là chuẩn admission webhook của Kubernetes, không
riêng gì Sigstore), chứa `request.object` = toàn bộ Pod sắp tạo. Nó đọc
field `spec.containers[].image` (ví dụ `ghcr.io/.../w10-api:0.0.3`), rồi:

1. **Resolve tag → digest**: gọi registry API hỏi "tag `0.0.3` hiện trỏ tới
   digest nào" (đây cũng là lúc policy-controller hoạt động như 1
   **mutating** webhook — nó có thể sửa lại field `image` trong Pod thành
   dạng `...@sha256:<digest>` trước khi Pod thật sự được tạo, để tránh
   trường hợp tag bị đổi trỏ sau khi đã admission-pass).
2. **Tìm xem `glob` có khớp image này không**:
   ```yaml
   spec:
     images:
       - glob: "ghcr.io/donan2k5/w10-api:**"
   ```
   Nếu không khớp, policy này không áp dụng, image được cho qua (có thể
   vẫn bị `ClusterImagePolicy` khác chặn nếu có).
3. **Tải manifest chữ ký** bằng đúng quy ước ở mục 2: tự suy ra tag
   `sha256-<digest>.sig` từ digest ở bước 1, gọi registry `GET` về.
4. **Verify**: lấy public key trong `authorities[0].key.data`:
   ```yaml
   authorities:
     - name: cosign-key
       key:
         data: |
           -----BEGIN PUBLIC KEY-----
           <nội dung cosign.pub>
           -----END PUBLIC KEY-----
   ```
   (chính là nội dung `signing/cosign.pub`, file PEM chứa public key ECDSA),
   chạy phép toán `verify(public_key, digest, signature)`. Đây là phép toán
   đối nghịch với bước "ký" ở mục 2 — chỉ trả `true` nếu signature đúng là
   được tạo ra bởi private key tương ứng với public key này, trên đúng
   digest này.
5. Trả `AdmissionReview.response.allowed = true/false` về `kube-apiserver`
   — giống cơ chế chung mọi admission webhook: `false` thì Pod không bao
   giờ được ghi vào etcd, `kubectl run`/`apply` thấy lỗi 403 ngay tại dòng
   lệnh.

### Vì sao "ký theo digest" lại chặn được cả trường hợp đổi tag sau

Giả sử ai đó (hoặc 1 pipeline khác) push đè tag `0.0.3` bằng 1 image độc hại
chưa từng được ký. Digest của image mới này khác hoàn toàn digest cũ (digest
= hash nội dung, nội dung khác thì hash khác). Khi Pod dùng tag `0.0.3` được
tạo, bước 3.2-1 resolve ra **digest mới** (của image độc hại) — bước 3.2-3
sẽ tìm tag `sha256-<digest-mới>.sig`, **không tồn tại** trên registry (chưa
ai ký digest này) → verify fail → admission reject. Đây chính là lý do
"ký theo digest" an toàn hơn "ký theo tag": kẻ tấn công không thể tự tạo ra
1 signature hợp lệ cho digest mới mà không có private key.

## 4. Tổng kết: data đi qua những định dạng nào

```
Dockerfile + source code (src/api/)
   │  docker build
   ▼
Image layers (.tar.gz) + manifest JSON  -- trong Docker daemon của runner
   │  Trivy đọc layer, liệt kê package, so với CVE DB (JSON nội bộ Trivy)
   ▼
[PASS] -> docker push -> registry trả lại digest (sha256:...)
   │  cosign sign: digest --(ECDSA private key)--> signature (Base64)
   ▼
Manifest chữ ký (tag "sha256-<digest>.sig") -- lưu trên cùng registry,
cạnh manifest image gốc
   │  ... thời gian sau, có người tạo Pod dùng image này ...
   ▼
kube-apiserver nhận CREATE Pod -> check namespaceSelector của webhook
   │  (nếu namespace có label) gửi AdmissionReview JSON cho policy-controller
   ▼
policy-controller: tag -> digest (gọi registry) -> tải manifest .sig (gọi registry)
   -> verify(public_key PEM, digest, signature) -- phép toán ECDSA
   ▼
AdmissionReview.response.allowed = true/false  -- trả lại cho kube-apiserver
   │
   ▼
Pod được ghi vào etcd (allowed=true) hoặc bị từ chối (allowed=false)
```

Không có bước nào ở trên cần "tin tưởng" registry — registry chỉ là chỗ lưu
trữ thuần (giống ổ đĩa mạng), toàn bộ phần "đáng tin" nằm ở phép toán
ECDSA verify cuối cùng, dựa trên public key bạn tự kiểm soát trong
`policies/cluster-image-policy.yaml`.
