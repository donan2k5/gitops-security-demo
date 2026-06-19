# ServiceAccount & RBAC — tóm gọn + deepdive

Tóm gọn lại phần cần thiết từ bài viết rất hay của tác giả trên Viblo —
["Kubernetes Series Bài 13 - ServiceAccount and Role Based Access Control"](https://viblo.asia/p/kubernetes-series-bai-13-serviceaccount-and-role-based-access-control-security-kubernetes-api-server-07LKXQD4ZV4)
— giữ nguyên cách giải thích của tác giả, chỉ rút lại phần liên quan trực
tiếp tới `rbac/roles.yaml` + `rbac/rolebindings.yaml` trong repo này, và
deepdive thêm ở những đoạn quan trọng nhất.

## 1. API server phân biệt 2 loại "ai đang gọi tôi": người và Pod

> Như chúng ta đã nói ở bài 10, API server có thể được config với một hay
> nhiều authentication plugins, khi một request đi tới API server nó sẽ đi
> qua hết các authen plugins này. Những plugin này sẽ phân tách những thông
> tin cần thiết như username, user id, group mà client thực hiện request
> thuộc về.

Có 2 loại client được API server phân biệt rõ ràng:

- **Humans (users)** — dùng `kubectl` hoặc 1 HTTP request kèm token để xác
  thực.
- **Pod (ứng dụng chạy bên trong container)** — dùng **ServiceAccount** để
  xác thực.

Trong repo này, `rbac/rolebindings.yaml` đang gán quyền cho 3 **User**
(`alice`, `bob`, `carol`) — tức là 3 con người, không phải Pod/app. Cơ chế
ServiceAccount (mục 2 dưới đây) là nền tảng để hiểu RBAC, nhưng bản thân
file RBAC của repo lab này nói về phân quyền cho người vận hành, không phải
cho app.

## 2. ServiceAccount — Pod xác thực với API server bằng gì

> ServiceAccount sẽ được tự động mount vào bên trong container của Pod ở
> folder `/var/run/secrets/kubernetes.io/serviceaccount`. Gồm 3 file là
> `ca.crt`, `namespace`, `token`.
>
> File token này là file sẽ chứa thông tin về Pod client, khi ta dùng nó để
> thực hiện request tới server, API server sẽ tách thông tin từ trong token
> này ra. Và ServiceAccount username của chúng ta sẽ có dạng như sau
> `system:serviceaccount:<namespace>:<service account name>`.

```bash
$ kubectl get sa
NAME     SECRETS  AGE
default  1        10d
```

ServiceAccount là **namespace resource** — chỉ có scope trong 1 namespace,
không dùng chéo namespace được. Mỗi namespace luôn có sẵn 1 ServiceAccount
tên `default`, tự gán cho Pod nào không khai báo `serviceAccountName` riêng.

### Deepdive: vì sao mặc định 1 SA "có quyền làm mọi thứ" nếu không bật RBAC

Đây là đoạn quan trọng nhất của cả bài, đáng đọc nguyên văn:

> Thì để trả lời vấn đề này thì mặc định một thằng SA **nếu ta không bật
> Role Based Access Control authorization plugin** thì nó sẽ có quyền thực
> hiện mọi hành động lên API server, nghĩa là một thằng ứng dụng trong
> container có thể sử dụng SA để authentication tới API server và thực
> hiện list Pod, xóa Pod, tạo Pod mới một cách bình thường, vì nó có đủ
> quyền. Thì để ngăn chặn việc đó, ta cần enable Role Based Access Control
> authorization plugin.

Tách rõ 2 khái niệm hay bị nhập nhằng:

- **Authentication** = "tôi biết bạn là ai" (nhờ token mount trong
  `/var/run/secrets/.../token`, API server tách ra được username dạng
  `system:serviceaccount:<ns>:<sa-name>`).
- **Authorization** = "tôi biết bạn là ai, nhưng bạn có được làm việc này
  không?" — đây là việc của RBAC. Authentication xảy ra **trước**, luôn
  thành công nếu token hợp lệ; Authorization xảy ra **sau**, và nếu không
  bật RBAC, kết quả luôn là "cho qua hết" — đây là lý do mọi cluster sản
  xuất phải bật RBAC, không phải tính năng tuỳ chọn.

## 3. RBAC — 4 loại resource, đúng những gì repo này dùng

> - **Roles**: định nghĩa verb nào có thể được thực hiện lên trên namespace
>   resource
> - **ClusterRoles**: định nghĩa verb nào có thể được thực hiện lên trên
>   cluster resource
> - **RoleBindings**: gán Roles tới một SA (hoặc User)
> - **ClusterRoleBindings**: gán ClusterRoles tới SA (hoặc User)
>
> Điểm khác nhau của Roles và ClusterRoles là Roles là namespace resource,
> nghĩa là nó sẽ thuộc về một namespace nào đó. Còn ClusterRoles thì sẽ
> không thuộc namespace nào cả.

Action ↔ verb (dùng đúng khi viết `rules.verbs`):

| Action (HTTP) | Verb |
|---|---|
| HEAD, GET | `get` / `list` / `watch` |
| POST | `create` |
| PUT | `update` |
| PATCH | `patch` |
| DELETE | `delete` |

### Đúng 3 role mà repo này định nghĩa, đọc trực tiếp `rbac/roles.yaml`

```yaml
# Role: developer - CRUD workload, chỉ trong ns demo
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: demo
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "services", "deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

`developer` là **`Role`** (không phải `ClusterRole`) — có `namespace: demo`
ngay trong `metadata`, đúng như tác giả mô tả "Roles là namespace resource,
thuộc về một namespace nào đó". `apiGroups: ["", "apps"]` nghĩa là áp dụng
cho cả core API group (`""` — chứa `pods`, `services`) và `apps` (chứa
`deployments`); nếu thiếu `"apps"` trong list này, `developer` vẫn login
được nhưng mọi request tới `deployments` sẽ bị 403 dù verb có đủ.

```yaml
# ClusterRole: sre - xem + thao tác pod toàn cụm
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sre
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

`sre` là **`ClusterRole`** — không có `namespace` trong `metadata` (không
thể có, ClusterRole luôn ở cluster scope). Để ý `resources: ["pods/log",
"pods/exec"]` — đây không phải 2 resource riêng, mà là 2 **subresource**
của `pods` (đúng API thật của Kubernetes: muốn chạy `kubectl logs`/`kubectl
exec` vào pod của namespace khác, verb `get`/`create` thường không đủ — bắt
buộc phải xin riêng quyền trên subresource này).

```yaml
# ClusterRole: viewer - chỉ đọc, toàn cụm
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: viewer
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
```

`viewer` dùng `apiGroups: ["*"]` + `resources: ["*"]` — áp dụng cho **mọi**
resource, mọi group, nhưng verb chỉ có `get/list/watch` (không có
`create/update/patch/delete`) → đọc được hết, sửa được gì cũng không.

### Gán quyền: `RoleBinding` / `ClusterRoleBinding`, đọc `rbac/rolebindings.yaml`

> Để cho phép thằng default SA ở namespace foo có thể list được service
> trong namespace foo, thì ta cần phải tạo một Role và dùng RoleBinding để
> gán quyền cho default SA này.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: demo # <- RoleBinding cũng là namespace resource, phải khai báo
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role # this must be Role or ClusterRole
  name: developer # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

`subjects` = "gán **cho ai**" (ở đây là `User: alice` — tác giả gọi
`ServiceAccount` cũng được, `RoleBinding` chấp nhận cả 2 `kind`). `roleRef`
= "gán **cái Role nào**" — `RoleBinding` này nằm trong `namespace: demo`
nên chỉ có hiệu lực bind `Role` cũng đang nằm trong `demo` (không thể dùng
1 `RoleBinding` ở namespace `demo` để gán 1 `Role` đang nằm ở namespace
khác — đây là giới hạn buộc phải dùng `ClusterRole` + `ClusterRoleBinding`
nếu muốn 1 quyền áp dụng nhiều namespace, đúng như `bob`/`carol` dưới đây).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob-sre # <- không có "namespace:" vì ClusterRoleBinding ở cluster scope
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: sre
  apiGroup: rbac.authorization.k8s.io
```

`bob` được bind tới `ClusterRole: sre` qua `ClusterRoleBinding` (không có
`namespace`) → `bob` có quyền `sre` ở **toàn bộ cluster**, mọi namespace,
không chỉ `demo`. `carol-viewer` áp dụng đúng cơ chế tương tự với
`ClusterRole: viewer`.

### Bảng tổng hợp ai được làm gì trong repo này

| User | Bind tới | Loại | Phạm vi | Được làm gì |
|---|---|---|---|---|
| `alice` | `developer` (`Role`) | RoleBinding | chỉ ns `demo` | CRUD pods/services/deployments trong `demo` |
| `bob` | `sre` (`ClusterRole`) | ClusterRoleBinding | toàn cluster | CRUD pods + `logs`/`exec` ở mọi namespace |
| `carol` | `viewer` (`ClusterRole`) | ClusterRoleBinding | toàn cluster | chỉ đọc (`get/list/watch`) mọi resource, mọi namespace |

## 4. ClusterRole mặc định có sẵn — để biết khi nào KHÔNG cần tự viết Role

> - **view**: liệt kê toàn bộ resource trong 1 namespace (`get/list/watch`).
> - **edit**: kế thừa `view`, cộng thêm `create/update/patch/delete` cho
>   mọi resource trong namespace, **trừ** `Secret`, `Role`, `RoleBinding`.
> - **admin**: toàn quyền trên namespace, **kể cả** sửa `Secret`, `Role`,
>   `RoleBinding`. Khác `edit` đúng ở điểm này.
> - **cluster-admin**: toàn quyền trên toàn cluster, cross-namespace, truy
>   cập được cả cluster resource (Node, PersistentVolume...).

```bash
$ kubectl get clusterroles
admin
cluster-admin
edit
view
...
```

Đây là lý do `rbac/roles.yaml` không tự viết lại 1 ClusterRole tên `editor`
hay `admin-namespace` — nếu nhu cầu chỉ là "đọc hết, không sửa gì" thì gán
thẳng `ClusterRoleBinding` trỏ tới `view` có sẵn còn nhanh hơn viết Role
mới; repo này tự viết `developer`/`sre`/`viewer` vì cần khoanh **đúng tập
resource cụ thể** (`pods/services/deployments` cho developer,
`pods/log,pods/exec` cho sre) mà 4 role có sẵn không khớp 1-1.

## 5. Nguyên tắc khi thêm role mới: Principle of Least Privilege

> Khi làm thực tế, ta nên áp dụng Principle of Least Privilege, chỉ cho
> phép một thằng thực hiện được những thứ nó cần, không cung cấp những
> quyền không cần thiết cho nó.

Áp vào checklist cụ thể khi sửa `rbac/roles.yaml`:

- Đừng dùng `resources: ["*"]` hoặc `apiGroups: ["*"]` trừ khi thật sự cần
  (chỉ `viewer` trong repo này dùng, vì nó chỉ đọc — rủi ro thấp).
- Đừng gán `ClusterRoleBinding` nếu quyền chỉ cần dùng trong 1 namespace —
  dùng `Role` + `RoleBinding` như `developer`/`alice`, giữ blast radius
  trong đúng `demo`.
- Không gộp `verbs: ["delete"]` vào role chỉ cần đọc — nếu sau này phát
  hiện cần thêm quyền, sửa thêm vẫn dễ hơn lúc đầu cấp dư rồi phải đi thu
  hồi.
