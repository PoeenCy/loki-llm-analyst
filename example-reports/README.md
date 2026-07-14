# Example Reports

Thư mục này chứa các báo cáo mẫu **thực tế** được tạo ra bởi AI workflow này — không phải template, không phải placeholder. Mỗi report là output thực của pipeline `Grafana MCP → Fabric patterns → Structured Report`, chạy trên dataset `lab/access.log`.

## Mục đích

Người đọc thường hỏi: *"Output thực sự trông như thế nào?"* Thư mục này trả lời câu hỏi đó.

## Danh sách Reports

| File | Trigger | Patterns Used | Severity |
|---|---|---|---|
| [`incident-apache-500.md`](incident-apache-500.md) | Googlebot kích hoạt lỗi 500 (server misconfiguration) | `analyze_logs` → `create_cyber_summary` | 🟡 MEDIUM |
| [`threat-hunt-recon.md`](threat-hunt-recon.md) | Phát hiện chủ động: Nikto + Gobuster scanner | `analyze_logs` → `recon_pattern` → `create_cyber_summary` | 🔴 HIGH |
| [`data-exfiltration-alert.md`](data-exfiltration-alert.md) | Báo động: Wget tải 86MB `/exports/*.csv` | `netflow_baseline` → `create_cyber_summary` | 🔴 CRITICAL |

## Cấu trúc một Report

Tất cả report đều tuân theo cấu trúc bất biến:

```
SUMMARY       — Tóm tắt điều hành 2-3 câu + mức độ nghiêm trọng
IOCS          — Danh sách chỉ số xâm nhập (IP, UA, paths, tools)
MITRE ATT&CK  — Mapping kỹ thuật tấn công theo framework
RECOMMENDATION — Hành động ngay lập tức + ngắn hạn + dài hạn
```

## Thời gian tạo report

Mỗi report trong thư mục này được tạo ra trong vòng **1–3 phút** từ thời điểm bắt đầu điều tra. Chi tiết xem trong từng file và trong [`results/benchmark.md`](../results/benchmark.md).
