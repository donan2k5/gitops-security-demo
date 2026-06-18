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
