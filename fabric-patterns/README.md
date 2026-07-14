# Custom Fabric Security Patterns

Thư mục này chứa **6 custom Fabric patterns** được thiết kế dành riêng cho workflow **SOC Tier 3 Analyst** kết hợp với Grafana MCP + Loki. Đây là phần bổ sung cho 9 built-in patterns của Fabric framework.

## Cài đặt

### Linux / macOS
```bash
# Copy tất cả patterns vào thư mục Fabric
cp -r ./* ~/.config/fabric/patterns/
```

### Windows (PowerShell)
```powershell
# Copy tất cả patterns vào thư mục Fabric
Copy-Item -Recurse .\* "$HOME\.config\fabric\patterns\" -Force
```

### Verify cài đặt
```bash
fabric --list | grep -E "k8s_pod_anomaly|cicd_supply_chain|dns_exfil_detect|netflow_baseline|recon_pattern|lateral_movement"
# Expected: 6 dòng output
```

---

## Danh sách Patterns

| Pattern | Mục đích | Use case |
|---|---|---|
| [`k8s_pod_anomaly`](./k8s_pod_anomaly/system.md) | Phát hiện bất thường trong Kubernetes pods | K8s security audit, runtime threat detection |
| [`cicd_supply_chain`](./cicd_supply_chain/system.md) | Phân tích bảo mật CI/CD pipeline | Phát hiện supply chain attack, build tampering |
| [`dns_exfil_detect`](./dns_exfil_detect/system.md) | Phát hiện DNS tunneling & C2 qua DNS | Exfiltration detection, DGA detection |
| [`netflow_baseline`](./netflow_baseline/system.md) | Phân tích baseline lưu lượng mạng | Anomaly detection, lateral movement via network |
| [`recon_pattern`](./recon_pattern/system.md) | Phát hiện trinh sát của kẻ tấn công | Early warning, scanner detection, path probing |
| [`lateral_movement`](./lateral_movement/system.md) | Theo dõi di chuyển ngang trong mạng | Post-compromise detection, credential abuse |

---

## Cách sử dụng

### Cơ bản (pipe từ output khác)
```bash
# Sau khi lấy log từ Grafana MCP, pipe qua pattern
echo "[raw log data]" | fabric -p recon_pattern

# Kết hợp nhiều patterns
echo "[log data]" | fabric -p analyze_logs | fabric -p recon_pattern
```

### Trong workflow SOC Tier 3
Các patterns này được gọi tự động bởi AI Agent khi sử dụng lệnh `/soc3analyst`. Xem [`.agents/workflows/soc3analyst.md`](../.agents/workflows/soc3analyst.md) để biết thêm chi tiết.

---

## Cấu trúc một Pattern

Mỗi pattern là một thư mục chứa file `system.md` — đây là system prompt gửi cho LLM:

```
fabric-patterns/
└── pattern_name/
    └── system.md    ← System prompt của pattern
```

Để tạo pattern mới, xem [CONTRIBUTING.md](../CONTRIBUTING.md).
