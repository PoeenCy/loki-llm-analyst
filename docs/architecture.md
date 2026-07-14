# Architecture & Design Decisions

This document explains **why** this system is built the way it is. Each decision involves a tradeoff, and understanding those tradeoffs is important for anyone contributing to or deploying this project.

---

## System Overview

```
┌────────────────────────────────────────────────────────────┐
│  LAYER 1 — Analyst (Local Machine)                         │
│                                                            │
│  ┌──────────────┐   MCP calls   ┌──────────────────────┐  │
│  │  LLM Agent   │ ────────────► │  Grafana MCP Server  │  │
│  │  (Antigravity│               │  (uvx mcp-grafana)   │  │
│  │   IDE)       │               └──────────┬───────────┘  │
│  └──────┬───────┘                          │ HTTPS + Token │
│         │ pipe output                       ▼               │
│         │               ┌───────────────────────────────┐  │
│         └─────────────► │  Fabric Framework             │  │
│                         │  15 Security Patterns         │  │
│                         └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────── ┘
                           │ HTTPS (443 only)
                           ▼
┌────────────────────────────────────────────────────────────┐
│  LAYER 2 — Cloud Infrastructure                            │
│                                                            │
│  Caddy (TLS) → Grafana (:3000 internal) → Loki (:3100)    │
│                                    ▲                       │
│                               Promtail (log shipper)       │
└────────────────────────────────────────────────────────────┘
```

---

## Architecture Decision Records (ADRs)

### ADR-001: LLM Agent on Client Side, Never Server Side

**Decision:** The LLM agent runs entirely on the analyst's local machine. It has no direct access to the production server, Loki endpoint, or any internal network resources.

**Rationale:**
- **Security:** A compromised LLM agent cannot exfiltrate data directly from production systems. The blast radius of an agent failure is limited to the analyst's machine.
- **Privacy:** Raw log data is fetched and processed locally. Logs are not sent to a third-party LLM API without passing through the analyst's machine first.
- **Auditability:** All agent actions are visible in the IDE — no "shadow" automation running on infrastructure.

**Tradeoff:** The analyst must install and maintain Go, uv, and Fabric locally. This is more setup than a SaaS SOC tool, but the security boundary is worth it.

---

### ADR-002: Grafana MCP as the Only Data Access Layer

**Decision:** The LLM agent is explicitly forbidden from querying Loki directly. All log access goes through the Grafana MCP server using a read-only service account.

**Rationale:**
- **Access control:** Grafana's RBAC allows creating a `Viewer`-role service account with exactly the permissions needed — nothing more.
- **Auditability:** Every query the agent makes is logged by Grafana (query audit log), not just in the agent's output.
- **Rate limiting:** Grafana's API enforces query timeouts (10 seconds in the lab config), preventing runaway queries that could DoS the Loki backend.
- **Abstraction:** If the log backend changes from Loki to Elasticsearch or another system, only the Grafana datasource configuration changes — the agent's workflow is unchanged.

**Tradeoff:** Two-layer indirection adds latency (~200–500ms per query). For incident triage this is acceptable.

**Why not direct Loki API?**
Loki's HTTP API has no native RBAC. A compromised API token would give full read access to all log streams with no audit trail per-query.

---

### ADR-003: Fabric as the Analysis Layer (Not Custom LLM Prompts)

**Decision:** All analysis goes through named Fabric patterns (`analyze_logs`, `recon_pattern`, etc.) rather than inline prompts in the agent's system prompt.

**Rationale:**
- **Reproducibility:** A named pattern is a version-controlled artifact. If `recon_pattern/system.md` changes, we know exactly what changed and when.
- **Consistency:** The same pattern applied to the same input produces structurally identical output — same section headers, same format. This makes reports machine-parseable.
- **Hallucination mitigation:** Fabric patterns define strict output schemas (`SUMMARY | IOCS | MITRE | RECOMMENDATION`). An unconstrained LLM prompt would produce free-form text that varies per run.
- **Composability:** Patterns can be chained: `analyze_logs | recon_pattern | create_cyber_summary`. Each stage refines the output.

**Tradeoff:** Fabric must be installed locally. It's an external dependency (Go binary). If Fabric changes its pattern format, all custom patterns may need updating.

**Why not just use a big system prompt?**
A system prompt that tries to do everything (initial analysis + IOC extraction + MITRE mapping + report generation) produces worse results than chained, focused patterns. Each Fabric pattern is optimized for one task.

---

### ADR-004: Human-in-the-Loop for All Containment Decisions

**Decision:** The AI agent produces reports and recommendations but **never** executes containment actions (no firewall rule creation, no IP blocking, no account lockout). All responses are advisory.

**Rationale:**
- **Accountability:** Security actions have consequences. Blocking an IP that turns out to be a legitimate partner's monitoring system can cause service disruption and business impact. A human must own that decision.
- **Context:** The LLM has no knowledge of business context — "is this IP a known partner?", "is this service under a maintenance window?". The analyst has that context.
- **Regulatory:** In regulated environments (GDPR, HIPAA, PCI-DSS), security decisions require an identifiable human accountable for each action taken.
- **Trust calibration:** LLM confidence scores are not reliable enough to automate containment. An 87% confidence score means 13% chance of error — unacceptable for firewall changes.

**Tradeoff:** Response time for containment is limited by human availability. At 2 AM, this may mean a 15–30 minute gap between detection and containment.

**Planned mitigation:** The RECOMMENDATION section of each report includes pre-formatted firewall commands that the analyst can copy-paste with minimal friction — reducing the human execution time even if the decision remains human.

---

### ADR-005: Loki over Elasticsearch/Splunk

**Decision:** This project uses Grafana Loki as the log storage backend rather than Elasticsearch (ELK stack) or Splunk.

**Rationale:**

| Criterion | Loki | Elasticsearch | Splunk |
|---|---|---|---|
| Cost (self-hosted) | Free | Free (BSL license post-7.10) | Expensive |
| Operational complexity | Low | Medium-High | High |
| Grafana integration | Native | Via plugin | Via plugin |
| Log indexing model | Labels only (efficient) | Full-text index (expensive) | Full-text index |
| Grafana MCP support | ✅ Official | ⚠️ Via workaround | ❌ No MCP |

**Tradeoff:** Loki's label-only indexing makes some ad-hoc queries slower than Elasticsearch's full-text index. For structured log analysis (where fields are known), this is acceptable.

---

## What This Architecture Does NOT Do

Being explicit about limitations is important for honest assessment:

1. **No real-time alerting.** This is a reactive/proactive investigation tool, not a streaming alert engine. For real-time alerts, integrate with Grafana Alerting or a dedicated SIEM.

2. **No memory across investigations.** Each investigation is stateless — the agent does not remember what it found last week. A MISP or OpenCTI integration would be needed for long-term threat correlation.

3. **No automatic remediation.** By design (see ADR-004).

4. **No multi-log-source correlation.** Current workflow is optimized for Apache logs. Correlating Apache + DNS + NetFlow + AD logs requires a dedicated correlation engine, not this workflow.

5. **No offline operation.** Fabric requires an LLM API (OpenAI, Anthropic, local Ollama). The Grafana MCP server requires internet access to the Grafana instance.

---

## Security Threat Model

For the threat model of this system itself (not the systems it protects), see [`threat-model.md`](threat-model.md).
