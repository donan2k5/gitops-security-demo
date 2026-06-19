# OPA Gatekeeper — giải thích cơ chế

## 0. 4 luật đang enforce — cấm cái gì, vì sao nguy hiểm

| #   | Risk | Cấm gì                                                         | Vì sao nguy hiểm nếu không cấm                                                                                                                                                                                                                                                                                                                   |
| --- | ---- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | F-01 | Tag `:latest` hoặc không ghi tag (image không pin version)     | `:latest` trỏ tới image **mới nhất tại thời điểm pull**, không cố định. Cùng 1 manifest, deploy hôm nay và hôm sau có thể chạy 2 phiên bản code khác nhau → không reproducible, không rollback được chính xác (vì không biết bản cũ là image nào), dễ bị tấn công supply-chain (ai đó push đè tag `latest` bằng image độc hại mà bạn không biết) |
| 2   | F-02 | Container không khai `resources.limits` (cpu/memory)           | Không có limit = container có thể ăn hết CPU/RAM của node, làm crash hoặc làm chậm **toàn bộ pod khác cùng node** (noisy neighbor) — 1 bug leak memory trong app của bạn có thể kéo sập cả cluster                                                                                                                                               |
| 3   | F-04 | `runAsUser: 0` (hoặc không khai gì, vì image tự mặc định root) | Process trong container chạy bằng root (UID 0). Nếu attacker khai thác được lỗ hổng trong app để thực thi code tùy ý, họ có ngay quyền root trong container → dễ leo thang chiếm namespace kernel, mount, hoặc thoát container (container breakout) hơn nhiều so với chạy bằng user thường                                                       |
| 4   | —    | `hostNetwork: true`                                            | Pod dùng chung network namespace với **node vật lý** — pod thấy được hết traffic/port của node, bỏ qua luôn network policy/isolation giữa các pod. Một pod bị compromise có `hostNetwork` có thể nghe trộm traffic của pod khác trên cùng node, hoặc truy cập service nội bộ chỉ bind `localhost` trên node                                      |

## 1. Vì sao phải cài "chart của gatekeeper" trước?

Gatekeeper **không phải** là 1 file YAML đơn lẻ — nó là một bộ **controller** chạy
trong cluster, gồm:

- 1 Deployment "controller-manager" (đọc Constraint, đánh giá Rego/CEL)
- 1 Deployment "audit" (quét định kỳ resource cũ chưa qua admission)
- 1 `ValidatingWebhookConfiguration` — đây là phần quan trọng nhất: nó báo cho
  API server biết "mỗi khi có request tạo/sửa Pod (hoặc bất kỳ kind nào), hãy
  gọi sang webhook của tôi trước, tôi sẽ trả lời allow/deny"
- Vài CRD mới: `ConstraintTemplate`, và `Constraint` (mỗi Constraint thực ra là
  1 CRD con được tạo _động_ — xem mục 2)

Nếu không cài controller này, viết `ConstraintTemplate`/`Constraint` thuần
chỉ là yaml vô nghĩa — không có ai lắng nghe API server để chặn request. Đó
là lý do bước 1 luôn là cài Helm chart `gatekeeper/gatekeeper`
(`argocd/apps/gatekeeper.yaml`), và phải xong **trước cả** khi áp luật
(sync-wave `-2`, sớm nhất trong toàn hệ thống).

## 2. ConstraintTemplate là gì?

`ConstraintTemplate` = **định nghĩa một LOẠI luật** (giống như định nghĩa 1
class trong lập trình, chưa khởi tạo object cụ thể).

Bên trong nó có 2 phần:

```yaml
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowedTags # (a) tên CRD MỚI mà template này sinh ra
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: | # (b) logic kiểm tra viết bằng Rego (OPA)
        package k8sdisallowedtags
        violation[{"msg": msg}] { ... }
```

- **(a)** Khi bạn `kubectl apply` 1 `ConstraintTemplate` có `kind:
K8sDisallowedTags`, Gatekeeper sẽ **tự động đăng ký 1 CRD mới** tên
  `K8sDisallowedTags` vào cluster (giống việc bạn tự viết `CustomResourceDefinition`,
  nhưng Gatekeeper làm hộ). Đây là lý do `gatekeeper-templates` phải sync
  **trước** `gatekeeper-constraints` — CRD `K8sDisallowedTags` phải tồn tại
  trước khi bạn tạo 1 object thuộc kind đó.
- **(b)** Đoạn Rego là hàm `violation` — nhận `input` là request admission
  (pod sắp được tạo) + `input.parameters` (tham số do Constraint truyền vào),
  trả về `msg` nếu vi phạm. ConstraintTemplate **chưa chạy** luật này cho ai
  cả — nó chỉ là khuôn mẫu.

4 ConstraintTemplate trong `gatekeeper/templates/` lấy **nguyên văn** từ
[gatekeeper-library](https://open-policy-agent.github.io/gatekeeper-library/)
(không tự viết Rego):

| File                      | CRD kind sinh ra            | Check gì                               |
| ------------------------- | --------------------------- | -------------------------------------- |
| `disallowed-tags.yaml`    | `K8sDisallowedTags`         | image có tag nằm trong blacklist không |
| `container-limits.yaml`   | `K8sContainerLimits`        | container có `resources.limits` chưa   |
| `allowed-users.yaml`      | `K8sPSPAllowedUsers`        | `runAsUser`/`runAsNonRoot` hợp lệ chưa |
| `host-network-ports.yaml` | `K8sPSPHostNetworkingPorts` | có set `hostNetwork: true` không       |

## 3. Constraint là gì? Khác ConstraintTemplate ra sao?

`Constraint` = **1 INSTANCE cụ thể** của ConstraintTemplate (giống tạo 1 object
từ class). Nó trả lời 3 câu hỏi mà ConstraintTemplate không tự quyết được:

1. **Áp dụng luật này cho ai?** → `spec.match` (kind nào, namespace nào, loại
   trừ namespace nào)
2. **Tham số cụ thể là gì?** → `spec.parameters` (ví dụ tag nào bị cấm — Rego
   trong template chỉ viết `input.parameters.tags`, còn giá trị thật `["latest"]`
   nằm ở đây)
3. **Vi phạm thì làm gì?** → `spec.enforcementAction: deny` (chặn thật) hay
   `dryrun` (chỉ log, không chặn)

Ví dụ `gatekeeper/constraints/disallow-latest-tag.yaml`:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1 # group cố định cho mọi Constraint
kind: K8sDisallowedTags # PHẢI khớp đúng kind do template sinh ra
metadata:
  name: disallow-latest-tag
spec:
  enforcementAction: deny
  match:
    kinds: [{ apiGroups: [""], kinds: ["Pod"] }]
    excludedNamespaces:
      [kube-system, gatekeeper-system, argocd, monitoring, argo-rollouts]
  parameters:
    tags: ["latest"]
```

→ 1 ConstraintTemplate có thể sinh ra **nhiều Constraint khác nhau** dùng cùng
1 logic Rego nhưng tham số/scope khác nhau (ví dụ: 1 Constraint cấm `latest`
áp cho ns `demo`, 1 Constraint khác cấm tag `dev` áp cho ns `staging`).

## 4. Luồng chạy thực tế khi có người `kubectl apply` 1 Pod

```
kubectl apply -f pod-xau.yaml
        │
        ▼
API server nhận request CREATE Pod
        │
        ▼
API server thấy có ValidatingWebhookConfiguration (do Gatekeeper tạo)
   → gọi HTTPS sang Gatekeeper webhook, gửi kèm pod-xau.yaml làm "input.review.object"
        │
        ▼
Gatekeeper controller lặp qua TẤT CẢ Constraint đang có
   → với mỗi Constraint, check spec.match có khớp Pod này không (namespace, kind)
   → nếu khớp: chạy hàm `violation` (Rego) của ConstraintTemplate tương ứng,
     truyền input.parameters từ Constraint vào
        │
        ▼
Nếu có bất kỳ violation nào VÀ enforcementAction=deny
   → Gatekeeper trả response "denied" kèm msg
        │
        ▼
API server từ chối request → user thấy lỗi ngay tại dòng lệnh,
Pod KHÔNG được tạo (chặn tại admission, không phải sau khi tạo xong mới xoá)
```

Đây là lý do gọi là **admission control** — chặn _trước khi_ object được ghi
vào etcd, khác hoàn toàn với việc dùng CronJob/script quét và xoá resource
vi phạm sau (đó là "audit", Gatekeeper cũng có nhưng chỉ để phát hiện resource
cũ, không phải cơ chế chính ở đây).

## 5. Vì sao phải tách App theo sync-wave?

```
wave -2: gatekeeper (controller + webhook)       ← phải Healthy trước
wave -1: gatekeeper-templates (4 ConstraintTemplate → sinh CRD)
wave  0: gatekeeper-constraints (4 Constraint → bắt đầu enforce thật)
wave  0/1/2: workload thật (api, alert, analysis...)
```

Nếu apply tất cả cùng 1 lúc (1 wave): Constraint có thể được tạo trước khi CRD
do ConstraintTemplate sinh ra tồn tại → lỗi `no matches for kind
"K8sDisallowedTags"`. ArgoCD đảm bảo wave N phải **Synced + Healthy hoàn
toàn** mới chuyển sang wave N+1, nên thứ tự controller → template →
constraint → workload luôn đúng, kể cả khi tất cả nằm trong cùng 1 lần
`git push`.

## 6. Bên dưới Kubernetes thật ra đang làm gì? (mổ từng lớp)

Phần trên giải thích "luồng logic". Phần này lật từng lớp xem **thực thể nào
trong cluster** đang nắm việc gì — để hiểu vì sao mọi quy tắc ở trên (CRD
trước, webhook, sync-wave...) là bắt buộc về kỹ thuật, không phải quy ước
suông.

### 6.1. `ConstraintTemplate` không tạo ra "luật" — nó gọi API server tạo ra một **kind** mới

Khi Gatekeeper controller thấy 1 object `ConstraintTemplate` mới (nó watch
resource này qua informer, giống mọi controller khác trong K8s), nó làm 2
việc tuần tự:

1. Lấy `spec.crd.spec.names.kind` (ví dụ `K8sContainerLimits`) +
   `spec.crd.spec.validation.openAPIV3Schema` (schema tham số), rồi **tự gọi
   API server tạo ra một `CustomResourceDefinition` thật** tên
   `k8scontainerlimits.constraints.gatekeeper.sh`. Sau bước này, với API
   server, `K8sContainerLimits` là một resource type hợp lệ — y hệt như
   `Pod` hay `Deployment`, chỉ khác là do Gatekeeper định nghĩa runtime thay
   vì built-in.
2. Lấy đoạn Rego trong `spec.targets[].rego` (+ `libs` nếu có), đưa vào OPA
   engine nhúng sẵn trong chính Gatekeeper controller (Gatekeeper viết bằng
   Go, dùng thư viện `github.com/open-policy-agent/opa` để **compile** đoạn
   Rego này thành 1 module sẵn sàng eval — không có service OPA riêng nào
   được gọi qua network, OPA chạy ngay trong process của Gatekeeper).

→ Đây là lý do gọi ConstraintTemplate là "định nghĩa một LOẠI luật": nó vừa
sinh ra **kind mới** (phần khung — CRD) vừa nạp **logic check** (phần thân —
Rego) cho kind đó. Hai việc xảy ra cùng lúc khi 1 ConstraintTemplate được
apply, và CRD chỉ tồn tại sau khi Gatekeeper xử lý xong — đó là khoảng trễ
khiến `gatekeeper-templates` phải Healthy trước khi `gatekeeper-constraints`
được sync.

### 6.2. `Constraint` chỉ là 1 object bình thường của kind vừa được sinh ra

Khi bạn `kubectl apply -f require-container-limits.yaml`
(`kind: K8sContainerLimits`), với API server đây **chỉ là tạo 1 custom
resource** như tạo bất kỳ object nào khác — API server không biết gì về
"Gatekeeper" hay "Rego" cả, nó chỉ biết "có CRD tên này, schema hợp lệ thì
cho lưu vào etcd". Gatekeeper controller cũng đang watch mọi instance của
kind này (vì nó tự đăng ký dynamic informer cho từng CRD nó sinh ra ở bước
6.1), nên khi object được lưu xong, Gatekeeper nạp nó vào bộ nhớ trong (cache
gọi là **constraint framework**) để dùng ở bước tiếp theo.

### 6.3. Cơ chế thật sự chặn request: `ValidatingWebhookConfiguration`

Đây là phần dễ bị bỏ qua nhất nhưng lại là **chốt chặn duy nhất**. Khi cài
Helm chart `gatekeeper`, ngoài Deployment, nó tạo:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
  - name: validation.gatekeeper.sh
    clientConfig:
      service: { name: gatekeeper-webhook-service, namespace: gatekeeper-system }
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["*"]
        resources: ["*"]
    failurePolicy: Ignore # (mặc định) — webhook lỗi/timeout thì KHÔNG chặn
```

Object này không nằm trong "thế giới Gatekeeper" — nó là tài nguyên **gốc của
Kubernetes**, API server tự đọc danh sách này mỗi khi xử lý request. Cơ chế:

1. API server nhận `kubectl apply` → parse xong object, biết đây là `CREATE
Pod` (hoặc bất kỳ resource nào khớp `rules` ở trên).
2. Trước khi ghi vào etcd, API server tra danh sách `ValidatingWebhookConfiguration`
   đang có trong cluster, thấy entry của Gatekeeper khớp `operations`/
   `resources` → tạm dừng, gửi 1 HTTP POST (AdmissionReview request, JSON) tới
   địa chỉ `clientConfig.service` (chính là Service trỏ vào Deployment
   `gatekeeper-controller-manager`).
3. Trong cluster, **mọi kind** (Pod, Deployment, ConfigMap...) đều đi qua bước
   này nếu khớp `rules` — không riêng gì Pod. Đây cũng là lý do
   `excludedNamespaces` trong từng Constraint quan trọng: nếu không loại trừ
   `gatekeeper-system`/`argocd`, chính các pod hệ thống cũng bị đưa vào diện
   xét, có thể gây tự-deadlock (Gatekeeper tự chặn pod của chính nó khi nó
   tự nâng cấp).
4. Gatekeeper controller nhận POST này, lấy `input.review` = toàn bộ object
   sắp được tạo, lặp qua các Constraint đã cache ở bước 6.2, với mỗi
   Constraint khớp `spec.match` thì **eval Rego module đã compile sẵn ở 6.1**
   với input đó.
5. Gatekeeper trả lại HTTP response dạng `AdmissionReview.response` với field
   `allowed: true/false` (+ `message` nếu false). API server đọc field này:
   `false` → trả lỗi 403 cho `kubectl`, **không** ghi object vào etcd dù chỉ
   một phần.

→ Hiểu rõ điều này thì thấy `failurePolicy: Ignore` (mặc định của
Gatekeeper) là một quyết định đánh đổi rất thật: nếu Gatekeeper controller
đang crash/restart, API server **không chặn lỗi này lại** mà tự cho qua
(fail-open) — để tránh việc 1 bug trong Gatekeeper làm sập toàn cluster
(không tạo được pod nào nữa). Đổi lại, có 1 khoảng thời gian ngắn (lúc
Gatekeeper restart) các luật **không được enforce**.

### 6.4. "Audit" — cơ chế thứ 2, chạy song song, không liên quan tới webhook

Bên cạnh webhook (chặn lúc tạo mới), Gatekeeper còn có 1 Deployment riêng
(`gatekeeper-audit`) chạy định kỳ (mặc định mỗi 60s): nó liệt kê toàn bộ
resource **đã có sẵn trong cluster** (kể cả tạo từ trước khi Constraint tồn
tại, hoặc lọt qua do `failurePolicy: Ignore`), chạy lại từng Rego module với
input là resource đó, rồi ghi kết quả vào field `status.violations` ngay
trên object `Constraint` (xem bằng `kubectl get k8scontainerlimits
require-container-limits -o yaml`). Audit **không xoá hay sửa** resource vi
phạm — nó chỉ để bạn nhìn thấy "đang có bao nhiêu resource cũ phạm luật" mà
không cần tự viết script quét.

### 6.5. Vì sao 4 Constraint trong repo này dùng `enforcementAction: warn`/`dryrun`

```bash
grep -r enforcementAction gatekeeper/constraints/
```

`deny` = chặn thật ở webhook (bước 6.3, request bị 403 ngay). `warn` = webhook
vẫn cho request đi qua, nhưng `kubectl apply` sẽ thấy 1 dòng `Warning:`
in ra (dùng cơ chế "Warning Headers" chuẩn của K8s API, không phải lỗi).
`dryrun` = im lặng hoàn toàn ở webhook, chỉ audit (6.4) mới thấy violation.
Đây là cách triển khai luật mới an toàn trong môi trường thật: bật `warn`
trước để biết luật có đang chặn nhầm workload nào không (mà chưa thực sự
chặn ai), rồi mới đổi sang `deny` khi đã chắc chắn.

## 7. Tự viết một ConstraintTemplate mới — từ đầu tới cuối

Giả sử muốn thêm luật mới: **mọi Pod phải có label `team`** (để biết pod nào
thuộc team nào khi audit chi phí/incident). Đây là ví dụ Rego đơn giản nhất
có thể, đủ để thấy hết các phần bắt buộc.

### Bước 1 — Xác định 3 thứ trước khi viết YAML

| Câu hỏi | Trả lời cho ví dụ này |
|---|---|
| Kind mới sẽ tên gì? | `K8sRequiredLabels` (PascalCase, Gatekeeper quy ước prefix `K8s`) |
| Input cần đọc từ đâu trong object? | `input.review.object.metadata.labels` |
| Tham số nào người dùng (Constraint) sẽ tự khai, không hard-code trong Rego? | `parameters.labels` — danh sách label key bắt buộc, để 1 template dùng lại được cho nhiều label khác nhau |

Nguyên tắc quan trọng nhất: **Rego trong ConstraintTemplate không bao giờ
hard-code giá trị cụ thể** (như tên label `team`) — giá trị cụ thể luôn nằm ở
`Constraint.spec.parameters`. Template = logic chung; Constraint = tham số +
phạm vi áp dụng. Vi phạm nguyên tắc này (hard-code trong Rego) là sai lầm
phổ biến nhất khi mới viết.

### Bước 2 — Viết `ConstraintTemplate`

```yaml
# gatekeeper/templates/required-labels.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels        # <- CRD mới sẽ mang tên này
      validation:
        openAPIV3Schema:               # <- schema cho Constraint.spec.parameters
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          required := input.parameters.labels[_]
          provided := input.review.object.metadata.labels
          not provided[required]
          msg := sprintf("object thieu label bat buoc: %v", [required])
        }
```

Giải nghĩa từng dòng cho người chưa từng đọc Rego:

- `package k8srequiredlabels` — bắt buộc khớp tên (không dấu hoa, không số
  đầu) với `crd.spec.names.kind` viết thường, đây là namespace nội bộ của
  module Rego, không phải tên hiển thị ra ngoài.
- `violation[{"msg": msg}] { ... }` — đây là **tên hàm cố định** mà Gatekeeper
  luôn tìm và gọi. Bạn không tự đặt tên khác được — Gatekeeper hard-code việc
  query rule tên `violation` sau khi eval. Mỗi lần thân hàm (phần trong `{}`)
  đúng (tất cả dòng bên trong đều true) thì sinh ra 1 violation với nội dung
  `msg`.
- `required := input.parameters.labels[_]` — `input.parameters` chính là
  `Constraint.spec.parameters` (không phải hard-code trong template, đây là
  binding tới Bước 1). Dấu `[_]` nghĩa là "lặp qua từng phần tử của mảng" —
  Rego không có vòng `for` truyền thống, nó dùng cơ chế này: dòng này sẽ
  được Rego tự thử lại với **mỗi** giá trị trong `labels`, và nếu bất kỳ lần
  thử nào toàn bộ thân hàm đúng → có 1 violation cho lần đó.
- `provided := input.review.object.metadata.labels` — `input.review.object`
  chính là **toàn bộ JSON của object sắp được tạo/sửa** (cùng dữ liệu mà
  `kubectl apply` gửi lên), tức là bạn có thể đọc bất kỳ field nào của nó,
  không riêng `metadata.labels`.
- `not provided[required]` — true nếu key `required` (ví dụ `"team"`) không
  tồn tại trong object `provided`. Đây là cách Rego biểu diễn "không có".
- `msg := sprintf(...)` — build message trả về cho người dùng thấy khi bị
  chặn.

### Bước 3 — Viết `Constraint` (instance, gắn tham số + phạm vi)

```yaml
# gatekeeper/constraints/require-team-label.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels             # phải khớp CHÍNH XÁC kind ở Bước 2
metadata:
  name: require-team-label
spec:
  enforcementAction: warn           # bật warn trước, xem 6.5
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
      - argocd
      - monitoring
      - argo-rollouts
  parameters:
    labels: ["team"]                # giá trị cụ thể nằm ở đây, không ở Rego
```

### Bước 4 — Test Rego **trước khi** đưa vào cluster

Đừng debug bằng cách apply thẳng vào cluster rồi đoán lỗi qua message —
Gatekeeper có CLI riêng tên `gator` chạy được OPA engine y hệt controller,
nhưng hoàn toàn local (không cần cluster):

```bash
go install github.com/open-policy-agent/gatekeeper/v3/cmd/gator@latest

# gator cần 1 file "test case": Constraint + ConstraintTemplate + object mẫu
gator test -f gatekeeper/templates/required-labels.yaml \
            -f gatekeeper/constraints/require-team-label.yaml \
            -f path/to/pod-mau-khong-co-label.yaml
```

`gator test` in ra đúng những violation mà Gatekeeper thật sẽ sinh ra khi
object đó đi qua webhook — đây là cách duy nhất để chắc Rego đúng logic
trước khi nó có quyền chặn request thật trong cluster.

### Bước 5 — Đưa vào GitOps đúng thứ tự sync-wave

File template đặt vào `gatekeeper/templates/` (đã được `gatekeeper-templates`
App, wave `-1`, sync toàn bộ thư mục này — không cần sửa gì thêm ở ArgoCD).
File constraint đặt vào `gatekeeper/constraints/` (App `gatekeeper-constraints`,
wave `0`). Vì 2 App này đã trỏ theo path (`gatekeeper/templates`,
`gatekeeper/constraints`), chỉ cần thêm file đúng thư mục + đúng thứ tự
sync-wave có sẵn sẽ tự áp dụng đúng — không cần tạo App mới.
