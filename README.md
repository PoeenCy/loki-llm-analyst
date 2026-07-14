# Loki-LLM Analyst

**An AI-assisted SOC Tier 3 investigation workflow.** Query Loki logs through Grafana MCP, analyze with 15 Fabric security patterns, get MITRE ATT&CK-aligned reports — in under 2 minutes.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Patterns](https://img.shields.io/badge/Fabric_Patterns-15_total_(6_custom)-brightgreen)](fabric-patterns/)
[![Stack](https://img.shields.io/badge/Stack-Grafana_%7C_Loki_%7C_Promtail_%7C_Caddy-orange)](lab/)

---

## The Problem

At 2 AM, an alert fires. You have hundreds of raw log lines to read, MITRE ATT&CK to cross-reference, and a ticket to write — before you can decide if this is real.

The traditional SOC workflow looks like this:

```
Alert fires
   → Open Grafana
   → Write LogQL query
   → Read raw log stream
   → Manually cross-reference MITRE ATT&CK
   → Write incident report
   → Escalate or close
```

Each step costs time and cognitive load. At 2 AM, cognitive load costs you more than during business hours.

**This project delegates steps 2–5 to an AI agent**, so you can focus on step 6: the decision that actually requires human judgment.

---

## Key Results

Tested against a 250-event Apache log dataset containing 6 attack scenarios (full methodology in [`results/benchmark.md`](results/benchmark.md)):

| Metric | Manual Analyst | AI-Assisted | Change |
|---|---|---|---|
| **Mean Time to Detect (MTTD)** | 6 min 41 sec | 1 min 18 sec | **−80%** |
| **Attack Patterns Detected** | 3.5 / 6 (58%) | 4.5 / 6 (75%) | **+29%** |
| **Structured Report Output** | ❌ Free-form notes | ✅ SUMMARY+IOC+MITRE+REC | — |
| **MITRE ATT&CK Mapping** | ❌ Manual lookup | ✅ Automatic | — |
| **Correct False Positive Classification** | ✅ 2 / 2 | ✅ 2 / 2 | — |

> These numbers come from a small dataset (6 scenarios). See the [full benchmark](results/benchmark.md) for methodology, limitations, and what wasn't detected.

**Key finding:** On data exfiltration (Case Study 03), the human analyst correctly identified the high-bandwidth source but misclassified the intent as "misconfigured crawler." The AI workflow correctly identified it as deliberate data theft — because it analyzed the *sequence of paths accessed*, not just the volume.

---

## How It Works

```
[ LLM Agent ]
     │
     ├─► [Grafana MCP] ══(HTTPS + read-only token)══► [Grafana API] ──► [Loki]
     │          │
     │     (raw log data returned)
     │
     └─► [Fabric Patterns] ──► [SUMMARY | IOCS | MITRE | RECOMMENDATION]
```

**Two hard constraints:**

1. The AI agent **cannot** query Loki directly. All log access is brokered through the Grafana MCP server using a read-only service account token. This provides auditable, rate-limited, RBAC-enforced access.

2. The AI agent **cannot** execute containment actions. Reports are advisory. A human analyst makes every escalation and remediation decision. See [ADR-004](docs/architecture.md#adr-004-human-in-the-loop-for-all-containment-decisions).

For the full rationale behind every architectural decision, see [`docs/architecture.md`](docs/architecture.md).

---

## Example Output

This is what an actual AI-generated report looks like, produced from real log data in ~2 minutes:

> **From [`example-reports/data-exfiltration-alert.md`](example-reports/data-exfiltration-alert.md):**

```
SEVERITY: 🔴 CRITICAL
GENERATED IN: 2 min 12 sec

SUMMARY
IP 203.0.113.99 conducted a two-session data exfiltration operation on
06–07 Jun 2026 using Wget/1.21.3. ~86 MB of sensitive business data was
transferred across 27 files including customer PII (GDPR-relevant) and
financial records. Access pattern is inconsistent with legitimate crawler
behavior. GDPR Article 33 — 72-hour notification window is open.

IOCS
• Attacker IP: 203.0.113.99 (not a known CDN or search engine)
• Tool: Wget/1.21.3 (linux-gnu) — scripted, non-browser
• Data accessed: /exports/customer-list.csv (5 MB), /exports/user-data-2026.csv
  (8 MB), /exports/transaction-log.csv (10 MB), + 24 additional files

MITRE ATT&CK
• T1039 — Data from Network Shared Drive
• T1567 — Exfiltration Over Web Service
• T1029 — Scheduled Transfer (08:00 on consecutive mornings)

RECOMMENDATION (Immediate)
1. Block 203.0.113.99 at perimeter firewall
2. Take /exports/ offline — remove from web root
3. Begin GDPR breach assessment — contact DPO within 1 hour
```

See [`example-reports/`](example-reports/) for three full reports at different severity levels.

---

## Case Studies

| Case | Attack | MTTD (AI) | Detection |
|---|---|---|---|
| [01 — Recon](results/case-study-01-recon.md) | Nikto + Gobuster scanner | 1 min 38 sec | ✅ |
| [02 — Brute Force](results/case-study-02-brute-force.md) | Credential brute force → successful login → post-auth enumeration | 1 min 46 sec | ✅ |
| [03 — Exfiltration](results/case-study-03-exfiltration.md) | Systematic HTTP data exfiltration (~86 MB, 2 sessions) | 2 min 12 sec | ✅ |

---

## Quick Start

### What you need

| Component | Where | Purpose |
|---|---|---|
| A server with public IP | GCP, AWS, DigitalOcean, etc. | Runs Grafana + Loki |
| Docker + Docker Compose | Server | Container orchestration |
| Go 1.21+ | Local machine | Builds Fabric |
| `uv` | Local machine | Runs Grafana MCP server |
| An AI IDE (Antigravity, Cursor, etc.) | Local machine | Runs the AI agent |

### Step 1 — Deploy the log stack (5 minutes)

```bash
git clone https://github.com/your-org/loki-llm-analyst.git  # replace with your fork URL
cd loki-llm-analyst/lab

# Configure your environment
cp .env.example .env
# Edit .env: set DOMAIN to your server's public IP.nip.io
#             set GF_SECURITY_ADMIN_PASSWORD to something strong

# Start services
export $(grep -v '#' .env | xargs)
sudo -E docker-compose up -d

# Verify (all should show "healthy" within ~60 seconds)
docker-compose ps
```

### Step 2 — Create a read-only Grafana service account

```
Grafana UI → Administration → Service Accounts
→ Add service account: Name=soc-analyst-bot, Role=Viewer
→ Add token → Copy immediately (shown once)
```

Store the token securely. You'll need it in Step 4.

### Step 3 — Install analysis tools (local machine)

```bash
# Fabric (Go required)
go install github.com/danielmiessler/fabric/cmd/fabric@latest
fabric --update   # sync official patterns

# Install 6 custom security patterns from this repo
# Linux/macOS:
cp -r fabric-patterns/* ~/.config/fabric/patterns/

# Windows (PowerShell):
Copy-Item -Recurse .\fabric-patterns\* "$HOME\.config\fabric\patterns\" -Force

# Verify — should return 6 lines:
fabric --list | grep -E "k8s_pod_anomaly|cicd_supply_chain|dns_exfil_detect|netflow_baseline|recon_pattern|lateral_movement"
```

### Step 4 — Connect Grafana MCP to your AI IDE

Add to your IDE's MCP configuration:

```json
{
    "mcpServers": {
        "grafana": {
            "command": "uvx",
            "args": ["mcp-grafana"],
            "env": {
                "GRAFANA_URL": "https://YOUR_SERVER_IP.nip.io",
                "GRAFANA_API_KEY": "YOUR_GRAFANA_SERVICE_ACCOUNT_TOKEN"
            }
        }
    }
}
```

> **Windows:** Replace `"uvx"` with the absolute path: `"C:/Users/YOUR_USERNAME/.local/bin/uvx.exe"`

### Step 5 — Run your first investigation

Type `/soc3analyst` in your AI IDE, then:

```
Investigate Apache logs for the last 15 minutes via Grafana MCP.
Pipe through analyze_logs then create_cyber_summary.
```

---

## Security Patterns (15 total)

### Built-in Fabric Patterns (9)
`analyze_logs` · `analyze_malware` · `analyze_incident` · `create_threat_scenarios` · `create_stride_threat_model` · `analyze_email_headers` · `create_cyber_summary` · `analyze_threat_report` · `write_semgrep_rule`

### Custom Patterns in This Repo (6) — [`fabric-patterns/`](fabric-patterns/)

| Pattern | Detects |
|---|---|
| `k8s_pod_anomaly` | Privilege escalation, suspicious container behavior, namespace escape |
| `cicd_supply_chain` | Dependency confusion, build tampering, artifact integrity failures |
| `dns_exfil_detect` | DNS tunneling, DGA domains, beaconing — with entropy scoring |
| `netflow_baseline` | Traffic volume anomalies, exfiltration sessions, east-west movement |
| `recon_pattern` | Port/directory scanning, scanner User-Agents, credential enumeration |
| `lateral_movement` | Pass-the-Hash, PsExec, WMI exec, LOLBins, movement chain visualization |

---

## SOC Workflows

Four investigation modes, invoked via `/soc3analyst`:

| Mode | When to Use | Patterns Chained |
|---|---|---|
| **Incident Triage** | Active alert, something is on fire right now | `analyze_logs` → `analyze_malware` → `create_cyber_summary` |
| **Threat Hunt** | Proactive 24h baseline review | `netflow_baseline` → `recon_pattern` → `lateral_movement` |
| **K8s / DevOps** | Container, pod, or pipeline security | `k8s_pod_anomaly` → `cicd_supply_chain` → `write_semgrep_rule` |
| **DNS Deep Dive** | DNS anomalies or exfiltration suspicion | `dns_exfil_detect` → `netflow_baseline` → `analyze_threat_report` |

---

## Limitations

Honest assessment — this is not a complete SOC solution:

| Limitation | Impact | Workaround |
|---|---|---|
| **Small dataset benchmark** | The −80% MTTD figure is directional, not statistically significant | Run your own benchmark with production data |
| **LLM token limits** | Log volumes >5,000 lines may be truncated → missed findings | Limit queries with `limit=500`; paginate manually |
| **Second-order attacks** | Multi-step attacks requiring cross-IP timeline correlation are not reliably detected | Complement with OpenCTI/MISP for long-term correlation |
| **No real-time alerting** | This is an investigation tool, not a streaming alert engine | Use Grafana Alerting for real-time alerts; use this for investigation |
| **No false positive rate measured** | We measured detection, not false positive rate | Measure FP rate in your environment before trusting automation |
| **Cloud LLM data exposure** | Log excerpts are sent to OpenAI/Anthropic | Use local Ollama model for sensitive data |

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `exec: "uvx": executable file not found` | PATH not loaded in IDE | Use absolute path to `uvx.exe` in MCP config |
| `Grafana MCP auth error 401` | Wrong or expired API token | Regenerate token in Grafana Service Accounts |
| `context deadline exceeded` in Loki | Query time range too wide | Use `$__interval` instead of fixed ranges like `[24h]` |
| `Caddy: 502 Bad Gateway` | Grafana not yet healthy | Run `docker-compose ps` and wait 60 seconds |
| Fabric pattern not found | Pattern not installed locally | Re-run: `cp -r fabric-patterns/* ~/.config/fabric/patterns/` |

---

## Repository Structure

```
loki-llm-analyst/
│
├── results/                      ← Benchmark data & case studies
│   ├── benchmark.md              ← MTTD comparison: manual vs. AI
│   ├── case-study-01-recon.md    ← Nikto + Gobuster scanner detection
│   ├── case-study-02-brute-force.md  ← Credential brute force
│   └── case-study-03-exfiltration.md ← HTTP data exfiltration
│
├── example-reports/              ← Actual AI-generated report output
│   ├── incident-apache-500.md    ← MEDIUM: Server misconfiguration
│   ├── threat-hunt-recon.md      ← HIGH: Active recon campaign
│   └── data-exfiltration-alert.md ← CRITICAL: 86MB data theft
│
├── fabric-patterns/              ← 6 custom security patterns
│   ├── k8s_pod_anomaly/
│   ├── cicd_supply_chain/
│   ├── dns_exfil_detect/
│   ├── netflow_baseline/
│   ├── recon_pattern/
│   └── lateral_movement/
│
├── docs/                         ← Technical documentation
│   ├── architecture.md           ← Design decisions (5 ADRs)
│   ├── threat-model.md           ← STRIDE threat model of this system
│   └── system-design.md          ← Full system design & execution plan
│
├── lab/                          ← Complete lab environment
│   ├── docker-compose.yml        ← Grafana + Loki + Promtail + Caddy
│   ├── .env.example              ← Configuration template
│   ├── access.log                ← Sample Apache logs with attack patterns
│   ├── LAB_INSTRUCTION.md        ← Guided investigation exercises
│   └── LAB_WORKSHEET.md          ← Student worksheet
│
├── .agents/workflows/
│   └── soc3analyst.md            ← AI agent workflow definition
│
├── blog_soc_workflow.md          ← Full technical blog post (Vietnamese)
├── CONTRIBUTING.md
└── LICENSE
```

---

## References

- [Grafana MCP Server](https://github.com/grafana/mcp-grafana)
- [Fabric Framework by Daniel Miessler](https://github.com/danielmiessler/fabric)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [uv — Python Package Manager](https://github.com/astral-sh/uv)
- [Loki — Grafana's Log Aggregation System](https://grafana.com/oss/loki/)
- [GDPR Article 33 — Breach Notification](https://gdpr.eu/article-33-notification-of-a-personal-data-breach-to-the-supervisory-authority/)

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Most welcome: new Fabric patterns, additional case studies from real incidents, or corrections to the benchmark methodology.

## License

[MIT](LICENSE)
