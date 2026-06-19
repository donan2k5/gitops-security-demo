# Runbook: Rotate DB password qua ESO (Lab 2.1)

## Bối cảnh

`app-api` mount `db-credentials` dưới dạng **volume** (`app-api/rollout.yaml`,
`volumeMounts` → `/etc/secrets/db`), không phải env var. Kubelet tự đồng bộ lại nội dung
volume Secret theo chu kỳ poll riêng của nó (~60-90s, `--sync-frequency`) mà **không** cần
restart pod, nên khi Secret đổi nội dung, app sẽ thấy giá trị mới ngay ở lần đọc file tiếp
theo, không cần restart. Env var thì ngược lại — chỉ được set một lần lúc container start,
muốn đổi giá trị phải restart pod mới nhận. Đây là lý do bài lab yêu cầu mount volume,
không dùng env.

`eso/external-secret.yaml` đặt `refreshInterval: 30s`. ESO poll nguồn (ở đây là giá trị
trong bộ nhớ của provider `fake`, đóng vai trò thay cho AWS Secrets Manager) mỗi 30s và
ghi lại vào Secret `db-credentials` nếu giá trị đổi. 30s + thời gian đồng bộ của kubelet
vẫn nằm thoải mái dưới ngưỡng 60s yêu cầu. Nếu đặt ngắn hơn (ví dụ 5s) sẽ spam backend
secret thật (AWS Secrets Manager tính tiền theo API call và có rate limit) mà không có lợi
gì thêm; nếu đặt dài hơn (ví dụ 5 phút) thì sẽ trễ SLA 60s.

## Rotate và verify

```bash
# 1. Đổi giá trị ở nguồn (ở đây là fake provider; trên AWS thật sẽ là lệnh
#    `aws secretsmanager put-secret-value --secret-id db-password --secret-string ...`)
kubectl edit secretstore fake-store -n demo
# tăng spec.provider.fake.data[0].value và .version

# 2. Theo dõi K8s Secret cập nhật (kỳ vọng đổi trong khoảng ~30-60s)
watch -n5 "kubectl get secret db-credentials -n demo -o jsonpath='{.data.password}' | base64 -d"

# 3. Xác nhận pod KHÔNG restart (cột AGE/RESTARTS không đổi trước/sau khi rotate)
kubectl get pods -n demo -l app=api
```

Kỳ vọng: giá trị Secret đổi trong vòng dưới 60s; cột `AGE` và `RESTARTS` của các pod api
không thay đổi trước và sau khi rotate.

## Troubleshooting

- Lỗi `no matches for kind "SecretStore"` trên cluster mới → Application `eso` (operator,
  sync-wave `-2`) chưa cài xong CRD trước khi `eso-config` (sync-wave `-1`) cố apply
  `SecretStore`/`ExternalSecret`. Re-sync `eso` trước, rồi mới `eso-config`.
- Secret không cập nhật → kiểm tra `kubectl describe externalsecret db-credentials -n demo`
  xem có lỗi sync không, và xác nhận `refreshInterval` chưa bị tăng quá 60s.
