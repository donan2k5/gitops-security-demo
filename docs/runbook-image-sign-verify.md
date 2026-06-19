# Runbook: Scan, ký và verify image qua admission (Lab 2.2)

## Pipeline (`.github/workflows/build-push.yml`)

1. Build image local (`load: true`, chưa push).
2. **Trivy** scan image local đó; `exit-code: 1` nếu có bất kỳ phát hiện `HIGH`/`CRITICAL`
   nào sẽ fail job trước khi image kịp lên registry.
3. Chỉ khi scan pass mới build+push thật, lấy digest kết quả.
4. **Cosign** ký `ghcr.io/<owner>/w10-api@<digest>` dùng `COSIGN_PRIVATE_KEY` /
   `COSIGN_PASSWORD` (GitHub Actions secrets — xem [`docs/cosign-key-setup.md`](cosign-key-setup.md)
   để biết cách tạo; private key không bao giờ được commit).

Ký theo digest (không phải theo tag) nghĩa là bất kể tag nào admission webhook resolve ra
(`:<semver>` trong `app-api/rollout.yaml`, hoặc `:latest`) đều được kiểm tra với cùng một
chữ ký — nên đây không phải kiểu "chỉ ký `:latest` rồi coi như xong"; mọi tag trỏ tới digest
đó đều được tính, kể cả tag mà Argo Rollouts thực sự đang deploy.

## Enforcement: policy-controller + ClusterImagePolicy

- `argocd/apps/policy-controller.yaml` (sync-wave `-2`) cài admission webhook + CRD của
  Sigstore vào namespace `cosign-system`.
- `argocd/apps/policies.yaml` (sync-wave `-1`) sync `policies/cluster-image-policy.yaml`,
  yêu cầu chữ ký hợp lệ từ `signing/cosign.pub` cho bất kỳ image khớp
  `ghcr.io/donan2k5/w10-api:**`.
- **Policy chỉ áp dụng cho namespace có label `policy.sigstore.dev/include=true`.**
  Label này **chủ ý không** được đưa vào GitOps manifest của repo này.

### Vì sao label không được commit (cái bẫy)

Nếu `demo` bị gắn label trước khi có image nào thực sự được ký và verify thành công,
lần rollout `app-api` kế tiếp sẽ bị admission reject ngay — kể cả lúc bootstrap cluster
mới, lúc đó `root.yaml` sẽ không bao giờ xanh được. Chỉ bật enforcement bằng tay, sau khi
đã xác nhận một image đã ký deploy thành công:

```bash
# 1. Xác nhận image do CI build + ký đang chạy thành công
kubectl get rollout api -n demo

# 2. Chỉ sau đó mới bật enforcement cho namespace này
kubectl label namespace demo policy.sigstore.dev/include=true
```

## Verify 3 kịch bản nghiệm thu

```bash
# A. Push một thay đổi vào src/api dùng base image/dependency có CVE đã biết
#    -> kỳ vọng: job build-push.yml fail ở step Trivy, không có gì được push/ký.

# B. Thử deploy image chưa ký vào namespace đã gắn label
kubectl run unsigned-test --image=ghcr.io/donan2k5/w10-api:<bản build thủ công không có tag CI> -n demo
#    -> kỳ vọng: admission webhook reject với lỗi signature-verification.

# C. Deploy image do CI tạo ra (đã ký)
kubectl get pods -n demo -l app=api
#    -> kỳ vọng: pod được schedule bình thường, không có lỗi admission.
```

## Exception cho CVE

Nếu Trivy chặn vì CVE `HIGH`/`CRITICAL` mà vendor chưa kịp fix, đừng tắt Trivy toàn cục —
viết một exception có thời hạn. Xem [`docs/runbook-exception-cve-adr.md`](runbook-exception-cve-adr.md)
để biết template và quy trình.
