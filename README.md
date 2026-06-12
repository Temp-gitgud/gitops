# W9 Challenge
Repo này triển khai pipeline phát hành an toàn gồm 3 mảng:
**GitOps** (ArgoCD) · **Observability** (Prometheus/Grafana) ·
**Canary tự động** (Argo Rollouts + AnalysisTemplate).

---
## Cấu trúc repo

```
gitops/
├── app/
│ ├── app.py
│ └── Dockerfile
├── k8s/
│ ├── namespace.yaml
│ ├── web.yaml
│ └── \...
├── k8s-api/
│ ├── api.yaml 
│ ├── servicemonitor.yaml 
│ ├── analysis-template.yaml 
│ └── slo-rule.yaml \
├── argocd/
│ ├── root.yaml
│ └── apps/
│ ├── web.yaml
│ ├── kube-prometheus-stack.yaml
│ ├── argo-rollouts.yaml
│ └── api.yaml
├── .github/
│ ├── workflows/
│   ├── validate.yml
```
---
## Mảng 1 --- GitOps

Mọi thay đổi đều đi qua Git. ArgoCD tự sync, không `kubectl apply`
trực tiếp manifest.

**Workflow:**

```
git push → ArgoCD poll (\~3 phút) → diff Git vs cụm → apply nếu
OutOfSync
```

**selfHeal:** ArgoCD phát hiện drift (ai đó edit tay trong cụm) và
tự kéo về Git trong vài giây.

**Rollback:** Dùng `git revert`, không dùng `kubectl rollout
undo`.

```
Rollback đúng cách --- có audit trail, không bị selfHeal ghi đè

git revert HEAD \--no-edit

git push

ArgoCD sync commit revert → cụm về bản cũ trong < 3 phút

```

> Lý do không dùng `kubectl rollout undo`: ArgoCD selfHeal sẽ thấy
cụm ≠ Git rồi apply lại bản mới, làm rollback thất bại.

---

## Mảng 2 --- Observability & SLO

### SLI (Service Level Indicator)

```
success_rate = request không lỗi (non-5xx) / tổng request
```

Query Prometheus:

```
sum(rate(flask_http_request_total{namespace="demo",
http_status!\~\"5..\"}\[5m\]))

sum(rate(flask_http_request_total{namespace=\"demo\"}\[5m\]))
```

### SLO (Service Level Objective)

| Chỉ tiêu | Mục tiêu | Cửa sổ |

| Success rate | ≥ 99.5% | 30 ngày rolling |

| Error budget | 216 phút/tháng | 30 ngày |

### Alert --- Burn rate

Dùng **multi-window burn rate** thay vì threshold đơn giản --- bắt được cả incident gấp (fast) lẫn rò rỉ âm ỉ (slow).

| Loại | Cửa sổ | Ngưỡng burn rate | Ý nghĩa |

| Fast (critical) | 1h | > 14.4× | Error budget cháy hết trong ~2 giờ |

| Slow (warning) | 6h | > 6× | Error budget cháy hết trong ~5 ngày|

**Tại sao chọn ngưỡng 14.4×?**
```
SLO = 99.5% → error budget = 0.5%/tháng
Burn rate 14.4× = lỗi 14.4 × 0.5% = 7.2%
Cửa sổ 1h với burn rate 14.4× → cháy hết 30-ngày budget trong 30/14.4 ≈
2 ngày
```
Ngưỡng 14.4× trên cửa sổ 1h là ngưỡng chuẩn từ Google SRE Book cho "fast burn" đủ nhạy để alert sớm nhưng không quá nhiều false
positive.

**PrometheusRule** (\`k8s-api/slo-rule.yaml\`):

```
- alert: APIHighBurnRate
expr: |
(
sum(rate(flask_http_request_total{namespace=\"demo\",
http_status=\~\"5..\"}\[1h\]))

sum(rate(flask_http_request_total{namespace=\"demo\"}\[1h\]))

) \> (14.4 \* 0.005)

for: 5m
labels:
severity: critical
```

**Test alert:**

```bash

# Inject lỗi 50% → burn rate = 0.5/0.005 = 100× → alert fire trong \< 5
phút

# Sửa k8s-api/api.yaml: ERROR_RATE \"0\" → \"0.5\"

git commit -am \"inject error\" && git push

```

---

## Mảng 3 --- Canary tự động

### Strategy

Traffic tăng dần theo từng bước, Prometheus metric tự chấm sau mỗi bước.
Không cần người canh.

```

10% → 2 phút quan sát → 50% → 2 phút quan sát → 100%

↓ (nếu metric tệ) ↓ (nếu metric tệ)

ABORT → về bản cũ ABORT → về bản cũ

```

### AnalysisTemplate

**Query** (`k8s-api/analysis-template.yaml`):

```promql

sum(rate(flask_http_request_total{namespace=\"demo\",
http_status!\~\"5..\"}\[2m\]))

sum(rate(flask_http_request_total{namespace=\"demo\"}\[2m\]))

```

**Ngưỡng:** `result\[0\] \>= 0.95\` (≥ 95% thành công)

**Tại sao 95% (không phải 99.5%)?**

Cửa sổ 2 phút có ít dữ liệu hơn cửa sổ 30 ngày --- đặt 99.5% sẽ gây
nhiều false abort do biến động thống kê ngắn hạn. Ngưỡng 95% đủ để bắt
bản thực sự lỗi (ERROR_RATE=0.5 → success rate \~50%) mà không abort bản
tốt do noise.

**failureLimit: 3** --- chấm tệ 3 lần liên tiếp (= 90 giây) mới
abort, tránh abort do spike tạm thời.

### Test canary auto-abort

```bash

# Bản lỗi --- ERROR_RATE=0.5

# Sửa k8s-api/api.yaml: ERROR_RATE \"0\" → \"0.5\", VERSION \"v1\" →
\"v2-bad\"

git commit -am \"api v2-bad\" && git push

# Theo dõi

kubectl argo rollouts get rollout api -n demo \--watch

# Mong đợi: AnalysisRun Degraded → Rollout Aborting → về bản cũ trong < 3 phút

```

```bash

# Bản tốt --- ERROR_RATE=0

# Sửa k8s-api/api.yaml: ERROR_RATE \"0\", VERSION \"v3-good\"

git commit -am \"api v3-good\" && git push

# Mong đợi: AnalysisRun Successful → 10% → 50% → 100% tự động

```

---

## Reproduce từ đầu

```bash

# 1. Khởi động cụm

minikube start -p w9 \--driver=docker \--cpus=4 \--memory=6144

kubectl config use-context w9

# 2. Cài ArgoCD

kubectl create ns argocd

kubectl apply \--server-side -n argocd \\

-f
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd rollout status deploy/argocd-server

# 3. Apply root --- đây là lần duy nhất kubectl apply

kubectl apply -f argocd/root.yaml

# 4. Mọi thứ còn lại ArgoCD tự sync từ Git

kubectl -n argocd get applications -w

```

Build image (cần chạy local, không có CI build image):

```bash

docker build -t w9-api:1 app/

minikube image load w9-api:1 -p w9

```

---

 Bằng chứng

![](./media/image1.png)

![](./media/image2.png)

![](./media/image3.png)

![](./media/image4.png)

![](./media/image5.png)

![](./media/image6.png)

![](./media/image7.png)

![](./media/image8.png){width="5.761805555555555in"
height="1.7458333333333333in"}
