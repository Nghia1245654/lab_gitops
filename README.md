# Hướng Dẫn Vận Hành GitOps API Service: Canary Auto-Abort & Observability

Tài liệu này giải thích chi tiết về thiết kế đo lường (Observability), cấu hình cảnh báo, cơ chế tự động hoá Canary (Auto-Abort & Rollback) và quy trình vận hành khôi phục nhanh qua Git (GitOps Rollback).

---

## 1. Đáp Ứng Các Tiêu Chi Đánh Giá (Thế Nào Là Đạt)

Hệ thống đã triển khai đầy đủ và đáp ứng trọn vẹn cả 4 tiêu chí chấm điểm của Lab:

| Tiêu Chí Đánh Giá | Cách Thức Triển Khai & Cấu Hinh | Minh Chứng Đạt (Ảnh/Thao Tác) |
|---|---|---|
| **1. Thay đổi qua Git & ArgoCD Synced** | Toàn bộ mã nguồn Deployment, Rollout, Service, AnalysisTemplate, PrometheusRule được khai báo dạng mã nguồn (IaC) trong repo Git. ArgoCD tự động theo dõi repo Git và đồng bộ hóa tự động không lệch cấu hình (no drift). | [Xem ảnh ArgoCD UI Synced](#32-minh-chứng-2-trạng-thái-trực-quan-trên-argocd-ui) |
| **2. `git revert` rollback < 5 phút** | Khi phát hiện phiên bản lỗi trên Production, chỉ cần revert commit trên Git và push. ArgoCD tự động nhận biết thay đổi và hoàn trả (rollback) về phiên bản tốt trước đó trong vòng chưa đầy 5 phút. | [Xem Hướng dẫn vận hành Git Revert](#23-quy-trình-vận-hành-git-revert-rollback-dưới-5-phút) |
| **3. 1 SLO + 1 alert fire về email cá nhân** | Định nghĩa Rule cảnh báo `ApiHighErrorRate` trong `prometheus-rules.yaml`. Khi inject lỗi, tỷ lệ lỗi vượt quá 5% (SLO) trong 2 phút sẽ kích hoạt Alert. Alertmanager kết nối SMTP Gmail cá nhân gửi email tự động. | [Xem ảnh Prometheus Alert Firing](#33-minh-chứng-3-đồ-thị-cảnh-báo-trên-prometheus-ui) và [Gmail nhận alert](#34-minh-chứng-4-email-cảnh-báo-gửi-về-gmail-cá-nhân) |
| **4. Canary bản lỗi tự abort về bản cũ** | Cấu hình `AnalysisTemplate` đo lường `Success Rate` liên tục trong quá trình Canary. Khi phát hiện tỷ lệ lỗi tăng cao (> 5%), Argo Rollout tự động Abort đợt triển khai, scale-down bản lỗi về 0 và chuyển toàn bộ traffic về bản cũ ổn định. | [Xem ảnh Rollout Auto-Abort trên Terminal](#31-minh-chứng-1-canary-auto-abort--rollback-tự-động-trên-terminal) |

---

## 2. Giải Thích Đo Lường & Cảnh Báo (Observability)

### 2.1. Query Prometheus Đo Lường Success Rate (Trong AnalysisTemplate)
Để tự động hoá việc đánh giá chất lượng phiên bản mới, ta sử dụng tài nguyên `AnalysisTemplate` truy vấn Prometheus định kỳ mỗi `10s` trong vòng 1 phút (`count: 6`):

```yaml
query: |
  1 - (
    (sum(rate(flask_http_request_total{status="500", namespace="demo", job="api"}[1m])) or vector(0))
    /
    (sum(rate(flask_http_request_total{namespace="demo", job="api"}[1m])) or vector(1))
  )
```

**Giải thích công thức:**
* `flask_http_request_total`: Metric đếm số lượng HTTP request gửi tới ứng dụng Python Flask.
* `status="500"`: Lọc ra các request bị lỗi hệ thống.
* `rate(...[1m])`: Tính tốc độ số lượng request lỗi/thành công phát sinh trên mỗi giây trong khoảng thời gian 1 phút gần nhất.
* `or vector(...)`: Xử lý giá trị rỗng (tránh lỗi chia cho 0 hoặc trả về kết quả rỗng khi không có request).
* Công thức tính: `Success Rate = 1 - (Số request lỗi 500 / Tổng số request)`.
* **Ngưỡng an toàn (Threshold)**: `>= 0.95` (Tỷ lệ thành công phải đạt từ 95% trở lên). Nếu tỷ lệ thành công sụt giảm dưới 95%, hệ thống sẽ đánh dấu lần kiểm tra đó là **thất bại (Failed)**.
* **Ngưỡng huỷ bỏ (Failure Limit)**: `failureLimit: 1` (Chỉ cho phép tối đa 1 lần lỗi. Từ lần lỗi thứ 2 trở đi, tiến trình Canary sẽ bị dừng ngay lập tức).

### 2.2. Luật Cảnh Báo Prometheus Rule (Trong prometheus-rules.yaml)
Cấu hình cảnh báo hệ thống thông qua `PrometheusRule`:

```yaml
expr: |
  (sum(rate(flask_http_request_total{status="500", namespace="demo", job="api"}[2m])) or vector(0))
  /
  (sum(rate(flask_http_request_total{namespace="demo", job="api"}[2m])) or vector(1)) > 0.05
```

**Giải thích:**
* Nếu tỷ lệ lỗi HTTP 500 của dịch vụ API vượt quá **5%** (`> 0.05`) liên tục trong vòng **2 phút**, cảnh báo `ApiHighErrorRate` ở mức độ `critical` sẽ kích hoạt trạng thái **Firing**.
* Alertmanager sẽ bắt tín hiệu này, đóng gói và thực hiện gửi email thông báo về địa chỉ Gmail cá nhân của bạn thông qua máy chủ SMTP của Google.

### 2.3. Quy Trình Vận Hành `git revert` Rollback (Dưới 5 phút)
Khi phát hiện lỗi nghiêm trọng trên hệ thống và bạn muốn khôi phục thủ công một cách an toàn thông qua GitOps:
1. Xác định commit lỗi đã push lên nhánh chính bằng lệnh:
   ```bash
   git log --oneline
   ```
2. Thực hiện đảo ngược commit đó (ví dụ commit lỗi là `a1b2c3d`):
   ```bash
   git revert a1b2c3d
   ```
3. Push commit revert lên Git Server:
   ```bash
   git push origin main
   ```
4. ArgoCD với tính năng `automated` sync policy sẽ tự động phát hiện sự thay đổi cấu hình trên Git và đồng bộ hóa về Kubernetes cluster, hạ cấp ứng dụng trở lại phiên bản ổn định trước đó trong vòng chưa đầy **5 phút** (thường trong vòng 1-3 phút theo chu kỳ quét, hoặc lập tức nếu bấm Sync thủ công).

---

## 3. Các Minh Chứng Kết Quả (Báo Cáo Lab)

### 3.1. Minh Chứng 1: Canary Auto-Abort & Rollback tự động trên Terminal
*Khi phiên bản lỗi v5 được triển khai, hệ thống tự động phát hiện tỷ lệ lỗi tăng cao, Abort đợt deploy và Rollback về v4:*
![alt text](image/image%20copy%205.png)

---

### 3.2. Minh Chứng 2: Trạng Thái Trực Quan Trên ArgoCD UI
*Trực quan hóa tài nguyên trên giao diện ArgoCD: AnalysisRun bị thất bại (trái tim vỡ màu đỏ) và Rollout tự động hướng traffic về các Pods của ReplicaSet cũ:*
![alt text](image/image%20copy.png)

---

### 3.3. Minh Chứng 3: Đồ Thị Cảnh Báo Trên Prometheus UI
*Cảnh báo ApiHighErrorRate chuyển sang màu đỏ rực ở trạng thái **FIRING** trên Prometheus Dashboard:*
![alt text](image/image%20copy%203.png)
![alt text](image/image%20copy%204.png)

---

### 3.4. Minh Chứng 4: Email Cảnh Báo Gửi Về Gmail Cá Nhân
*Hộp thư Gmail cá nhân nhận được email cảnh báo đỏ từ Alertmanager thông báo `ApiHighErrorRate`:*
![alt text](image/image%20copy%202.png)