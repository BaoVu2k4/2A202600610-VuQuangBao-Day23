# Báo Cáo Thực Hành Ngày 23

> Điền đầy đủ từng mục. Giám khảo chấm kỹ nhất mục "Thay đổi quan trọng nhất".

**Sinh viên:** Vũ Quang Bảo
**Ngày nộp:** 2026-06-29
**Link repo GitHub:** https://github.com/BaoVu2k4/2A202600610-VuQuangBao-Day23

---

## 1. Thông tin phần cứng + kết quả kiểm tra môi trường

Kết quả chạy `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.5.3)
Compose v2:    OK  (5.1.4)
RAM available: 7.43 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: D:\AI_THUCCHIEN\NGAY23\2A202600610-VuQuangBao-Day23\00-setup\setup-report.json
```

*(Các cổng hiển thị BOUND là vì stack đang chạy và đang chiếm những cổng đó — đây là trạng thái bình thường, không phải lỗi.)*

---

## 2. Track 02 — Dashboard & Cảnh báo

### 6 panel chính (ảnh chụp màn hình)

Xem file `submission/screenshots/dashboard-overview.png`.

### Panel Burn-rate

Xem file `submission/screenshots/slo-burn-rate.png`.

### Vòng đời cảnh báo: nổ → tắt

| Thời điểm | Sự kiện | Bằng chứng |
|---|---|---|
| T0 | Tắt service app bằng lệnh `docker stop day23-app` | — |
| T0+90s | Cảnh báo `ServiceDown` nổ (severity=critical, receiver=slack-critical) | Ảnh `alertmanager-firing.png` |
| T1 | Khởi động lại app bằng lệnh `docker start day23-app` | — |
| T1+60s | Cảnh báo tự động tắt (Alertmanager chuyển sang trạng thái Resolved) | — |

### Điều bất ngờ khi làm việc với Prometheus / Grafana

Cái bẫy ẩn nhất là **sự không khớp UID của datasource**. Khi file `provisioning/datasources/datasources.yml` không khai báo trường `uid:`, Grafana tự sinh ra một mã ngẫu nhiên như `PBFA97CFB590B2093` mỗi lần khởi động. Trong khi đó, toàn bộ file JSON dashboard đều dùng `uid: "prometheus"` để tham chiếu datasource. Hậu quả: tất cả 6 panel đều hiển thị "No data" mặc dù Prometheus hoạt động hoàn toàn bình thường. Chỉ cần thêm một dòng `uid: prometheus` vào file YAML rồi restart Grafana là tất cả panel sáng lên ngay lập tức — nhưng để tìm ra nguyên nhân gốc rễ thì mất khá nhiều thời gian so sánh từng panel.

---

## 3. Track 03 — Tracing & Logs

### Ảnh chụp trace từ Jaeger

Xem file `submission/screenshots/jaeger-trace.png` — trace hiển thị chuỗi span: `predict` (lỗi ERROR) → `embed-text` → `vector-search` → `generate-tokens`.

- Trace ID: `2feae2ab9c2e45cd9bc3cb2be279946f`
- Tổng số span: 4 — Độ sâu: 2 — Thời gian: 167.64ms

### Dòng log liên kết với trace

```
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 62, "quality": 0.794,
 "duration_seconds": 0.1746, "trace_id": "b521526de9e94511b711a6e981756d98",
 "event": "prediction served", "level": "info",
 "timestamp": "2026-06-29T03:26:18.198764Z"}
```

Trường `trace_id` trong log có cấu trúc JSON cho phép Loki tự động tạo đường dẫn liên kết sang Jaeger thông qua cấu hình `derivedFields` trong datasource. Giám khảo hoặc kỹ sư on-call chỉ cần click vào `trace_id` trong Loki là chuyển thẳng sang màn hình chi tiết trace trong Jaeger — không cần copy-paste thủ công.

### Tính toán tỉ lệ lấy mẫu (tail-sampling)

Trong 60 giây load test, OTel collector nhận được khoảng **11.951 span** (tương đương ~200 span/giây). Với mỗi request sinh ra ~5 span, tốc độ trace là khoảng 40 trace/giây.

Chính sách lọc:
- `keep-errors`: giữ lại toàn bộ trace có ít nhất 1 span lỗi (HTTP 5xx / status ERROR)
- `keep-slow`: giữ lại trace có độ trễ > 2.000ms
- `probabilistic-1pct`: giữ ngẫu nhiên 1% trace còn lại (40 trace/s × 1% = 0,4 trace/s)

Kết quả thực tế: trong tổng số 11.951 span nhận vào, chỉ có **122 span được chuyển tiếp sang Jaeger** — giảm 99%, đúng như thiết kế để kiểm soát chi phí lưu trữ trace.

---

## 4. Track 04 — Phát hiện trôi dữ liệu (Drift Detection)

### Điểm PSI của từng đặc trưng

Nội dung file `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Chọn phương pháp kiểm định phù hợp cho từng đặc trưng

| Đặc trưng | Phương pháp | Lý do |
|---|---|---|
| `prompt_length` | **PSI** | Phân phối liên tục, không bị chặn. PSI cho một con số dễ hiểu (>0.2 = trôi nghiêm trọng). Độ dịch trung bình từ 50→85 ký tự rất lớn, PSI=3.46 phản ánh rõ ràng mức độ nghiêm trọng. |
| `embedding_norm` | **KS (Kolmogorov-Smirnov)** | Phân phối gần chuẩn, phương sai nhỏ, dao động quanh 1.0. KS nhạy với sự thay đổi nhỏ trong hàm phân phối tích lũy mà không cần chia bin, phù hợp cho phân phối hẹp và ổn định. |
| `response_length` | **KL Divergence** | Phân phối xấp xỉ Gaussian, KL nắm bắt tốt sự phân kỳ không đối xứng, đặc biệt hữu ích khi quan tâm đến phân phối tham chiếu đang "nặng đuôi" hơn phân phối hiện tại. |
| `response_quality` | **PSI** | Điểm chất lượng bị chặn trong [0,1], phân phối Beta. Phân phối chuyển từ chất lượng cao Beta(8,2) sang thấp Beta(2,6) — PSI=8.85 phát hiện sự thay đổi cực đoan này ngay lập tức. |

---

## 5. Track 05 — Tích hợp liên ngày

### Metric nào của các ngày trước khó đưa vào nhất? Tại sao?

Metric khó tích hợp nhất là **token throughput của llama.cpp (Ngày 20)** từ script `monitor-day20-llama-cpp.py`. Khác với các service cloud có endpoint `/metrics` chuẩn, llama.cpp chạy như một tiến trình local và cần được vá thêm sidecar exporter để phát Prometheus metrics. Ngoài ra, cấu hình GPU offload của llama.cpp thay đổi tùy phần cứng, khiến pipeline metric dễ bị vỡ nếu không kiểm soát được model và hardware. Dashboard tích hợp hiển thị đúng "No Data (start monitor-day20...)" vì service này không chạy trong môi trường Day 23 độc lập.

---

## 6. Thay đổi quan trọng nhất

**Thêm `uid: prometheus` và `uid: loki` vào file cấu hình Grafana datasource** là thay đổi duy nhất biến stack từ "chạy được nhưng không nhìn thấy gì" thành hoàn toàn có thể quan sát được.

Toàn bộ panel trong dashboard Grafana tham chiếu datasource qua cấu trúc `{"type": "prometheus", "uid": "prometheus"}`. Khi file `provisioning/datasources/datasources.yml` không khai báo trường `uid:`, Grafana tự sinh mã ngẫu nhiên (ví dụ `PBFA97CFB590B2093`) mỗi lần khởi động. Sáu panel dashboard đều âm thầm hiển thị "No data" vì UID trong datasource được cấp phát không khớp với UID `"prometheus"` được hard-code trong JSON dashboard. Chỉ thêm hai dòng — `uid: prometheus` và `uid: loki` — vào file YAML rồi restart Grafana là toàn bộ panel hiển thị dữ liệu thực ngay lập tức.

Điều này liên hệ trực tiếp với nguyên tắc **dashboards-as-code** trong bài giảng: cấp phát dashboard qua JSON/YAML chỉ hoạt động đáng tin cậy khi mọi tham chiếu đều xác định (deterministic). Mã tự sinh phá vỡ hợp đồng giữa định nghĩa datasource và dashboard tiêu thụ nó. Trong môi trường production thực tế, cùng một bug này sẽ âm thầm vô hiệu hóa toàn bộ hệ thống quan sát sau mỗi lần nâng cấp Grafana hoặc restart pod — chính xác là lúc anh cần dashboard nhất. Bài học: "có thể quan sát được" không chỉ là tính chất của pipeline dữ liệu, mà đòi hỏi toàn bộ chuỗi từ metric emitter → scraper → datasource → dashboard panel phải dùng chung một quy ước đặt tên nhất quán.
