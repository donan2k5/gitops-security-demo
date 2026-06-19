# Docs — Lab 2 (ESO + Trivy/Cosign + Gatekeeper + RBAC)

## Giải thích cơ chế (đọc để hiểu sâu, có mổ dưới Kubernetes)

- [`lab2.1-eso.md`](lab2.1-eso.md) — ESO là gì, rotate hoạt động ra sao, CRD nào cần có, secret nằm ở đâu, vì sao < 60s, vì sao không restart pod.
- [`lab2.2-trivy-cosign.md`](lab2.2-trivy-cosign.md) — Trivy scan cái gì, Cosign ký/lưu chữ ký ở đâu trên registry, Policy Controller verify thế nào lúc admission.
- [`gatekeeper.md`](gatekeeper.md) — ConstraintTemplate/Constraint là gì, luồng admission webhook, cách tự viết 1 ConstraintTemplate mới.
- [`rbac.md`](rbac.md) — ServiceAccount xác thực thế nào, Role/ClusterRole/RoleBinding/ClusterRoleBinding khác nhau ra sao, đọc trực tiếp `rbac/roles.yaml` + `rbac/rolebindings.yaml` của repo này.

## Setup

- [`cosign-key-setup.md`](cosign-key-setup.md) — tạo cosign keypair, push private key vào GitHub Secrets, dán public key vào policy.

## Runbook (vận hành / nghiệm thu)

- [`runbook-rotate-secret-eso.md`](runbook-rotate-secret-eso.md) — cách rotate secret và verify pod không restart.
- [`runbook-image-sign-verify.md`](runbook-image-sign-verify.md) — cách enable enforcement, verify 3 kịch bản scan/sign/admission.
- [`runbook-exception-cve-adr.md`](runbook-exception-cve-adr.md) — template ADR xin exception cho CVE chưa có fix, có thời hạn.
