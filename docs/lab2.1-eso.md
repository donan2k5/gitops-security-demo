# Lab 2.1 · ESO (External Secrets Operator) — giải thích chi tiết

## ESO là gì

External Secrets Operator (ESO) là một Kubernetes operator: nó đọc secret từ một nguồn
bên ngoài (AWS Secrets Manager, GCP Secret Manager, Vault, v.v.) và **tự ghi lại** thành
một `Secret` object chuẩn của Kubernetes trong cluster. Ứng dụng vẫn chỉ biết đọc K8s
Secret như thường, không cần biết AWS Secrets Manager là gì — ESO là lớp đứng giữa, lo
việc đồng bộ.

ESO hoạt động qua 2 custom resource:

- **`SecretStore`** (hoặc `ClusterSecretStore`): khai báo "nguồn secret ở đâu, auth bằng
  gì" — ví dụ region AWS + IAM role, hoặc trong demo này là provider `fake` (giá trị nằm
  ngay trong spec, không cần AWS thật).
- **`ExternalSecret`**: khai báo "lấy key nào từ SecretStore nào, map vào K8s Secret tên
  gì, key gì, và bao lâu thì check lại nguồn một lần" (`refreshInterval`).

## Cấu trúc file trong repo này

```
eso/
├── secret-store.yaml     # SecretStore "fake-store" — nguồn secret (fake provider)
└── external-secret.yaml  # ExternalSecret "db-credentials" — map fake-store → K8s Secret
```

### `secret-store.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: fake-store
  namespace: demo
spec:
  provider:
    fake:
      data:
        - key: db-password
          value: "initial-password-v1"   # <- đây là "giá trị trên AWS" trong demo
          version: v1
```

Provider `fake` là provider built-in của ESO dùng cho test/demo — giá trị secret nằm
ngay trong spec của `SecretStore` thay vì gọi ra AWS Secrets Manager thật. Production sẽ
đổi block `provider.fake` thành `provider.aws` (kèm `region`, `auth` IAM role/key) —
toàn bộ phần `ExternalSecret` phía dưới giữ nguyên không đổi.

### `external-secret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: demo
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: fake-store
    kind: SecretStore
  target:
    name: db-credentials       # <- tên K8s Secret sẽ được tạo ra
    creationPolicy: Owner
  data:
    - secretKey: password      # <- key trong K8s Secret
      remoteRef:
        key: db-password       # <- key bên SecretStore (nguồn)
```

`secretKey: password` + `remoteRef.key: db-password` chính là phần "map key AWS → key
K8s" mà đề bài nói tới: giá trị nằm ở `db-password` trên nguồn, được ESO ghi vào field
`password` của K8s Secret `db-credentials`.

## Rotate hoạt động như thế nào (luồng end-to-end)

```
AWS Secrets Manager (hoặc fake provider)
        │  1. đổi giá trị tại nguồn
        ▼
ESO controller (poll mỗi refreshInterval = 30s)
        │  2. phát hiện version/giá trị đổi → ghi lại
        ▼
K8s Secret "db-credentials" (namespace demo, field "password")
        │  3. kubelet đồng bộ lại nội dung volume mount (chu kỳ riêng ~60-90s,
        │     theo flag --sync-frequency của kubelet, KHÔNG phải của ESO)
        ▼
Pod "api" — file /etc/secrets/db/password trong container
        │  4. app đọc lại file này (nếu code đọc file mỗi request/mỗi lần cần,
        │     sẽ thấy giá trị mới ngay, không cần restart)
        ▼
Pod KHÔNG bị restart trong suốt quá trình trên
```

ESO không tự đẩy giá trị vào pod — nó chỉ chịu trách nhiệm cho bước 1→2 (đồng bộ K8s
Secret object). Bước 3 (Secret → file trong pod) là cơ chế có sẵn của kubelet, không
phải tính năng của ESO.

## Đọc secret ở đâu trong app/pod

Xem `app-api/rollout.yaml`:

```yaml
volumeMounts:
  - name: db-credentials
    mountPath: /etc/secrets/db
    readOnly: true
volumes:
  - name: db-credentials
    secret:
      secretName: db-credentials   # <- chính K8s Secret mà ExternalSecret tạo/owner
```

Secret được mount làm **volume**, không phải env var. Trong container, giá trị nằm ở
file `/etc/secrets/db/password`. Đây là điểm bắt buộc của bài: nếu dùng
`env: valueFrom: secretKeyRef`, giá trị chỉ được Kubernetes set một lần lúc container
khởi động (qua API server), đổi Secret sau đó **không** có tác dụng gì tới process đang
chạy — phải restart pod mới đọc được giá trị mới. Mount volume thì khác: kubelet coi
Secret-volume là một watch, định kỳ resync lại nội dung file trong container mà không
đụng tới process/pod đang chạy, nên app chỉ cần đọc lại file (không phải đọc 1 lần lúc
start) là thấy giá trị mới.

## Secret nằm ở đâu (vị trí thực tế)

- **Nguồn gốc (source of truth)**: `spec.provider.fake.data[].value` trong
  `eso/secret-store.yaml` (demo) — hoặc secret thật trong AWS Secrets Manager (production).
- **Bản sao trong cluster**: K8s `Secret` tên `db-credentials`, namespace `demo`, do ESO
  tạo và "owner" (field `creationPolicy: Owner` — nếu xoá `ExternalSecret`, Secret này bị
  xoá theo). Xem bằng:
  ```bash
  kubectl get secret db-credentials -n demo -o yaml
  kubectl get secret db-credentials -n demo -o jsonpath='{.data.password}' | base64 -d
  ```
- **Bản trong pod**: file `/etc/secrets/db/password` bên trong container `api`, do kubelet
  mount + đồng bộ từ K8s Secret ở trên.

## CRD cần có trước

ESO không phải tài nguyên built-in của Kubernetes — `SecretStore` và `ExternalSecret` là
**CustomResourceDefinition (CRD)** do ESO operator đăng ký khi cài đặt (qua Helm chart
`external-secrets`). Nếu apply `SecretStore`/`ExternalSecret` mà operator chưa cài, API
server không biết "kind: SecretStore" là gì → lỗi:

```
error: resource mapping not found for name: "fake-store" namespace: "demo" from "secret-store.yaml":
no matches for kind "SecretStore" in version "external-secrets.io/v1beta1"
```

Các CRD chính ESO đăng ký (liên quan tới bài này): `secretstores.external-secrets.io`,
`externalsecrets.external-secrets.io` (còn có `clustersecretstores...`,
`pushsecrets...` nhưng không dùng trong demo này).

## Vì sao phải chạy ESO operator trước khi apply SecretStore/ExternalSecret

Đây là quan hệ "CRD trước, Custom Resource sau": Custom Resource (`SecretStore`,
`ExternalSecret`) chỉ tồn tại được trong cluster nếu CRD định nghĩa nó đã được đăng ký.
Trong repo này được tách bằng 2 ArgoCD Application + sync-wave:

```
argocd/apps/eso.yaml          # cài operator ESO (Helm chart) — sync-wave "-2"
argocd/apps/eso-config.yaml   # sync eso/ (SecretStore + ExternalSecret) — sync-wave "-1"
```

ArgoCD áp dụng các Application theo thứ tự wave tăng dần: wave `-2` (operator + CRD) chạy
và phải **healthy** xong trước khi wave `-1` (`eso-config`) được sync. Nếu gộp chung 1
Application/1 lượt sync (không tách wave), ArgoCD có thể cố apply `SecretStore` cùng lúc
hoặc trước khi CRD của Helm chart kịp đăng ký xong → ra đúng lỗi `no matches for kind
"SecretStore"` ở trên, đặc biệt rõ trên cluster mới hoàn toàn (chưa có CRD nào sẵn).

## "< 60s" được đảm bảo như thế nào

Có 2 tầng độ trễ cộng lại:

1. **ESO → K8s Secret**: tối đa `refreshInterval` = **30s** (ESO poll nguồn theo chu kỳ
   này; nếu vừa đổi giá trị xong 1 giây trước khi ESO poll, phải chờ tới gần 30s mới được
   ESO nhận ra).
2. **K8s Secret → file trong pod**: kubelet đồng bộ lại nội dung Secret volume theo chu kỳ
   resync riêng của nó, mặc định khoảng vài chục giây (thường dưới 60s với cấu hình mặc
   định của kubelet, không cấu hình trong repo này vì là hành vi built-in).

Tổng cộng worst-case ~30s (ESO) + một khoảng ngắn (kubelet) vẫn nằm dưới ngưỡng 60s yêu
cầu của đề bài trong điều kiện vận hành bình thường. Đây là lý do `refreshInterval`
không được chọn dài hơn (ví dụ 1-2 phút sẽ tự nó đã vượt 60s, chưa kể cộng thêm độ trễ
kubelet) và cũng không cần chọn ngắn hơn nhiều (vài giây) vì sẽ chỉ tốn thêm API call tới
nguồn secret (tính phí/rate-limit trên AWS thật) mà không cải thiện được gì thêm — phần
cộng thêm của kubelet là cố định, không phụ thuộc `refreshInterval`.

## Mổ sâu: dưới Kubernetes đang thực sự xảy ra gì (giả định bạn chưa biết gì)

Phần trên giải thích "luồng ý tưởng". Phần này coi như bạn chưa từng nghe
"operator", "CRD", "controller", "etcd" là gì — đi từng bước thật, kèm
**định dạng dữ liệu thật** đang được truyền ở mỗi bước và **dòng code/file
nào trong repo này** tương ứng.

### Vài thuật ngữ cần biết trước (giải thích bằng ví dụ, không học thuật)

- **etcd**: Kubernetes không tự "nhớ" gì cả — mọi object (Pod, Secret,
  Deployment...) thực chất là 1 dòng dữ liệu lưu trong 1 database tên
  `etcd`. Khi bạn `kubectl get secret X`, lệnh đó thực ra hỏi `kube-apiserver`,
  rồi `kube-apiserver` đọc lại từ etcd và trả về.
- **CRD (CustomResourceDefinition)**: Kubernetes mặc định chỉ biết các kind
  có sẵn như `Pod`, `Secret`, `Deployment`. `SecretStore` và `ExternalSecret`
  **không phải** kind có sẵn — chúng chỉ tồn tại vì ESO, lúc cài đặt, tự gọi
  API tạo ra 1 CRD mới, nói với `kube-apiserver`: "từ giờ kind tên
  `ExternalSecret` cũng hợp lệ, đây là schema của nó". Không có CRD này thì
  `kubectl apply` file `external-secret.yaml` sẽ bị từ chối ngay từ
  `kube-apiserver` (lỗi `no matches for kind`).
- **Controller / Operator**: là 1 chương trình (ở đây chính là pod
  `external-secrets` chạy trong namespace `eso-system`) chạy một vòng lặp vô
  tận: "hỏi API server có gì mới/đổi không → nếu có, làm gì đó → ngủ/đợi tín
  hiệu tiếp". ESO là một Operator — vòng lặp của nó là: đọc `ExternalSecret`,
  gọi ra nguồn secret, rồi viết lại `Secret`. Bản thân `kube-apiserver`
  **không tự đi lấy secret từ AWS** — nó không biết AWS là gì, tất cả là do
  process Operator này chủ động làm.
- **Watch / informer**: thay vì Operator phải hỏi liên tục "có gì mới
  chưa?" (poll), Kubernetes API hỗ trợ Operator "đăng ký nghe" (watch) — mỗi
  khi 1 object loại nó quan tâm bị tạo/sửa/xoá, API server tự đẩy 1 sự kiện
  qua 1 kết nối HTTP đang mở (giống một dạng long-polling). Đây là cách
  Operator biết "có ExternalSecret mới" mà không cần hỏi dồn dập.

### Bước 1 — `SecretStore` chỉ là dữ liệu nằm im trong etcd, chưa có gì chạy

Khi bạn `kubectl apply -f eso/secret-store.yaml`, dữ liệu thật được gửi đi là
1 object JSON (YAML chỉ là cách người viết, API server luôn nhận/lưu dạng
JSON nội bộ) đại loại:

```json
{
  "apiVersion": "external-secrets.io/v1beta1",
  "kind": "SecretStore",
  "metadata": { "name": "fake-store", "namespace": "demo" },
  "spec": { "provider": { "fake": { "data": [
    { "key": "db-password", "value": "initial-password-v1" }
  ]}}}
}
```

`kube-apiserver` chỉ làm 2 việc: (1) check JSON này khớp schema CRD đã đăng
ký không, (2) ghi nguyên văn xuống etcd. **Không có gì "chạy" ở bước này** —
đây chỉ là lưu dữ liệu, giống ghi 1 dòng vào database.

### Bước 2 — ESO controller (đang chạy sẵn, watch liên tục) nhận được sự kiện

Pod của ESO operator (xem `kubectl get pods -n eso-system`) đã mở sẵn 1 watch
tới `kube-apiserver` cho 2 loại resource: `SecretStore` và `ExternalSecret`.
Khi object ở Bước 1 được ghi xong, `kube-apiserver` đẩy 1 sự kiện
`ADDED {object JSON ở trên}` qua kết nối watch đó. ESO controller nhận được,
lưu vào cache nội bộ (trong RAM của chính pod ESO) — đây là lúc nó "biết"
nguồn secret `fake-store` tồn tại và giá trị hiện tại là gì.

### Bước 3 — `ExternalSecret` kích hoạt vòng lặp reconcile của ESO

`eso/external-secret.yaml` dòng `secretStoreRef: { name: fake-store }` chính
là sợi dây nối 2 object lại — khi bạn apply file này, ESO controller (đang
watch `ExternalSecret` y như Bước 2) thấy object mới, đọc field
`secretStoreRef` để biết phải lấy dữ liệu từ `SecretStore` nào trong cache
của nó, rồi đọc `remoteRef.key: db-password` để biết lấy đúng field nào.

Sau đó nó **tự build** 1 object `Secret` chuẩn của Kubernetes trong process
của nó (không phải bạn viết tay):

```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": { "name": "db-credentials", "namespace": "demo" },
  "data": { "password": "aW5pdGlhbC1wYXNzd29yZC12MQ==" }
}
```

Lưu ý: field `data` trong `Secret` của Kubernetes **luôn** phải mã hoá
Base64 (đây là quy ước bắt buộc của kind `Secret`, không phải mã hoá bảo
mật — chỉ là cách JSON chứa byte tuỳ ý an toàn). Chuỗi trên là Base64 của
`initial-password-v1`, đúng với khai báo trong `eso/external-secret.yaml`:

```yaml
data:
  - secretKey: password # <- key này trong Secret = field "password" ở JSON trên
    remoteRef:
      key: db-password
```

ESO controller tự `POST`/`PUT` object `Secret` này lên `kube-apiserver` để
ghi vào etcd — đây chính là hành động "ESO tự sync secret" mà đề bài nói
tới.

Vòng lặp này (Bước 2 → 3) không chạy 1 lần rồi xong — ESO **lên lịch chạy
lại đúng theo**:

```yaml
spec:
  refreshInterval: 30s
```

sau 30s, ESO tự hỏi lại `SecretStore` xem `value`/`version` có đổi không, có
thì build lại `Secret` JSON ở trên với giá trị mới và ghi đè.

> **Lưu ý quan trọng nếu sau này đổi sang AWS thật**: `kube-apiserver`
> **không bao giờ** tự gọi ra AWS. Mọi cuộc gọi tới AWS Secrets Manager (API
> HTTPS, có ký request bằng AWS Signature V4) đều do chính process ESO thực
> hiện, trong Bước 3, ngay trước khi build `Secret` JSON.

### Bước 4 — `kube-apiserver` chỉ là người gác cổng, không tự đẩy gì cho pod

Tới đây, `Secret` đã nằm trong etcd, nhưng **pod `api` đang chạy không hề
nhận được gì** — `kube-apiserver` không có khái niệm "đẩy" dữ liệu Secret
vào trong container của pod đang sống. Việc này hoàn toàn do **kubelet**
(chương trình chạy trên từng node, quản lý mọi container trên node đó) làm.

### Bước 5 — kubelet biến `Secret` JSON thành file thật trong container

Khi pod `api` được tạo, `app-api/rollout.yaml` khai báo:

```yaml
volumeMounts:
  - name: db-credentials
    mountPath: /etc/secrets/db
    readOnly: true
volumes:
  - name: db-credentials
    secret:
      secretName: db-credentials
```

kubelet trên node chứa pod đó thấy volume kiểu `secret`, tên `secretName:
db-credentials` → nó tự gọi `kube-apiserver` lấy `Secret` JSON ở Bước 3 về,
giải mã Base64 từng field trong `data`, rồi **viết ra file thật** trong 1
vùng nhớ tạm (`tmpfs`, RAM, không phải đĩa) gắn vào container tại đúng
`mountPath: /etc/secrets/db`. Field `password` trong JSON trở thành file
`/etc/secrets/db/password` với nội dung là chuỗi đã giải mã.

kubelet **không làm việc này một lần rồi nghỉ** — nó có 1 thành phần tên
"Secret Manager" giữ cache của Secret đang dùng, với 1 TTL (mặc định khoảng
1 phút, có thể chỉnh qua flag `--sync-frequency` của kubelet, repo này không
chỉnh gì nên dùng mặc định). Hết TTL, kubelet tự hỏi lại `kube-apiserver`
"Secret `db-credentials` hiện tại là gì", nếu khác bản đang có trong tmpfs,
nó **ghi đè lại file** — và đây là cơ chế thay thế file mà **không cần khởi
động lại container**: file trong tmpfs bị thay nội dung ngay trong khi
process trong container vẫn đang chạy, đúng như khi bạn sửa 1 file bằng
editor mà không cần tắt chương trình đang mở file đó.

### Vậy "đọc lại file" ở phía app nghĩa là gì

Container `api` không có code đọc secret trong repo này (đây là demo lab,
mục tiêu là chứng minh hạ tầng đồng bộ đúng, không phải app logic) — nhưng
nguyên tắc bắt buộc để pattern này có tác dụng thật là: code app phải
**đọc lại file `/etc/secrets/db/password` mỗi khi cần dùng** (ví dụ mỗi lần
mở connection tới DB), **không** đọc 1 lần lúc khởi động rồi giữ biến trong
RAM. Nếu code chỉ đọc 1 lần lúc start, dù file dưới đĩa đã đổi, giá trị
trong RAM của app vẫn là giá trị cũ — bug kinh điển khi áp dụng pattern
"secret rotation không restart" mà không sửa code app cho đúng.

### Tổng kết: data đi qua bao nhiêu định dạng khác nhau

```
YAML bạn viết (eso/secret-store.yaml)
   │  kubectl apply: parse YAML -> JSON
   ▼
JSON object trong etcd (qua kube-apiserver)
   │  ESO watch nhận event JSON, đọc field thường (string)
   ▼
ESO build JSON "Secret" mới, field data.password = Base64(string)
   │  ghi qua kube-apiserver -> etcd
   ▼
kubelet GET lại JSON Secret, giải Base64 -> raw bytes
   │  viết xuống tmpfs (RAM filesystem) trong container
   ▼
File thật /etc/secrets/db/password — raw bytes, KHÔNG còn Base64
   │  app tự đọc bằng syscall read() bình thường, như đọc file .txt
   ▼
Chuỗi password app dùng để connect DB
```

Mỗi mũi tên ở trên là 1 lớp hệ thống khác nhau (kubectl, kube-apiserver,
etcd, ESO controller, kubelet, container runtime) — không có lớp nào "biết"
về AWS Secrets Manager ngoài chính ESO controller ở bước build JSON Secret.

## Bug thật đã gặp: `Secret does not exist` dù YAML "đúng"

Provider `fake` lưu mỗi entry bằng 1 key nội bộ = **ghép dính `key` + `version`**
(đọc trong source code của ESO, file `pkg/provider/fake/fake.go`):

```go
func mapKey(key, version string) string {
    return fmt.Sprintf("%v%v", key, version)
}

func (p *Provider) GetSecret(_ context.Context, ref esv1beta1.ExternalSecretDataRemoteRef) ([]byte, error) {
    data, ok := p.config[mapKey(ref.Key, ref.Version)]
    if !ok || data.Version != ref.Version {
        return nil, esv1beta1.NoSecretErr // <- chính là "Secret does not exist"
    }
    ...
}
```

`eso/external-secret.yaml` không khai báo `remoteRef.version` (mặc định `""`), nên
`SecretStore` cũng phải để `version` rỗng cho field tương ứng — nếu `secret-store.yaml`
có thêm `version: v1`, key nội bộ lúc lưu là `"db-passwordv1"`, còn key lúc ESO tra cứu
(version rỗng) là `"db-password"` → 2 chuỗi khác nhau → lookup miss → `NoSecretErr`, dù
nhìn 2 file YAML qua mắt người vẫn thấy "khớp nhau" (cùng `key: db-password`). Đây là lý
do `eso/secret-store.yaml` trong repo này **không** có field `version`.

## Test thử (tóm tắt — xem đầy đủ ở [`docs/runbook-rotate-secret-eso.md`](runbook-rotate-secret-eso.md))

```bash
# Đổi giá trị tại nguồn — CHỈ đổi value, không thêm field version (xem lý do ở trên)
kubectl edit secretstore fake-store -n demo

# Theo dõi K8s Secret cập nhật
watch -n5 "kubectl get secret db-credentials -n demo -o jsonpath='{.data.password}' | base64 -d"

# Xác nhận pod không restart
kubectl get pods -n demo -l app=api
```
