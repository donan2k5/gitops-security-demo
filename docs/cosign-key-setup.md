# Setup cosign key (làm 1 lần, ở máy local — KHÔNG commit private key)

```bash
cosign generate-key-pair
# sinh ra 2 file: cosign.key (private) và cosign.pub (public) trong thư mục hiện tại
```

1. Thay nội dung `signing/cosign.pub` trong repo này bằng `cosign.pub` vừa sinh ra, commit lại.
2. Dán cùng public key (PEM) đó vào `authorities[0].key.data` trong `policies/cluster-image-policy.yaml`.
3. Đẩy private key + password lên GitHub Actions secrets (repo Settings → Secrets and variables → Actions):
   ```bash
   gh secret set COSIGN_PRIVATE_KEY < cosign.key
   gh secret set COSIGN_PASSWORD --body "<password bạn đặt lúc generate-key-pair>"
   ```
4. Xoá `cosign.key` ở local sau khi đã đưa vào GitHub Secrets (hoặc lưu trong password manager) — không bao giờ commit file này vào git.

`build-push.yml` dùng 2 secret này để ký mỗi image vừa push, theo digest.

Giải thích chi tiết "ký là ký cái gì, lưu ở đâu, admission verify bằng cách
nào dưới Kubernetes" — xem [`docs/lab2.2-trivy-cosign.md`](lab2.2-trivy-cosign.md).
