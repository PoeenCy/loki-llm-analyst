# Case Study 03 — HTTP Data Exfiltration (Bandwidth Theft)

**Scenario:** Deliberate, systematic data exfiltration via HTTP GET — disguised as a web crawler  
**Dataset:** `lab/access.log`, 06–07 Jun 2026  
**MTTD (AI):** 2 min 12 sec  
**MTTD (Manual):** 8 min 30 sec (partial detection only — intent misclassified)  
**Result:** ✅ Detected by AI, ⚠️ Misclassified by human

---

## 1. Context

This is the most operationally significant case study because it illustrates a key gap in manual analysis: **a human analyst can detect high bandwidth, but may misclassify the intent**. The AI workflow correctly classified this as deliberate data exfiltration.

**What happened:**

IP `203.0.113.99` used `Wget/1.21.3 (linux-gnu)` to systematically download every file under `/exports/` and `/downloads/` over two sessions. The pattern is deliberate:

- **Session 1** (06 Jun, 08:00–08:09): Systematically downloaded all exports in alphabetical order, then all downloadable PDFs. Total ~48 MB.
- **Session 2** (07 Jun, 08:00–08:10): Returned the next morning, re-downloaded the export CSVs again plus additional financial files. Total ~38 MB.

The cumulative pattern is a hallmark of **staged data exfiltration**: the attacker already knows the file paths (from prior reconnaissance), returns at a quiet off-peak time (8 AM, before business hours in the target timezone), and systematically transfers all high-value data files.

**High-value files targeted:**
```
/exports/customer-list.csv        5.0 MB  ← Customer PII
/exports/transaction-log.csv     10.0 MB  ← Financial records
/exports/user-data-2026.csv       8.0 MB  ← User data (GDPR relevant)
/exports/inventory.csv            4.0 MB  ← Business intelligence
/exports/analytics-june.csv       6.0 MB  ← Marketing data
/exports/financial-summary.xlsx   3.5 MB  ← Financial statements
/exports/partner-contacts.xlsx    2.0 MB  ← B2B relationship data
/downloads/report-q1-2026.pdf     2.0 MB
/downloads/whitepaper.pdf         3.0 MB
... (additional files)
```

---

## 2. Why the Human Analyst Missed the Intent

The manual analyst correctly identified `203.0.113.99` as the top bandwidth consumer (correct) but classified it as **"a misconfigured crawler, not exfiltration"** because:

1. All responses were HTTP 200 (the files are legitimately accessible)
2. The `Wget` User-Agent appeared to be a generic crawler
3. No error codes, no unusual HTTP methods

**The signal the human analyst missed:** The files being accessed are not crawlable web pages — they are `/exports/*.csv` and `/downloads/*.pdf`. No legitimate search engine crawler targets `customer-list.csv`. The sequential, alphabetical access pattern across business-sensitive paths is an exfiltration signature, not a crawl signature.

---

## 3. Investigation Prompt

```
Investigate bandwidth anomalies in Apache logs via Grafana MCP.
Focus on the period 06-07 Jun 2026 08:00-10:00 each day.
Identify the top bandwidth consumers, analyze what files were accessed,
and determine if this represents normal crawler activity or data exfiltration.
Pipe through netflow_baseline then create_cyber_summary.
```

---

## 4. AI Workflow Execution Trace

```
Step 1 [00:00-00:25] → grafana MCP: query_loki
  LogQL: sum by (client_ip)
           (sum_over_time({job="apache"}
           | regexp ".* (?P<bytes>\\d+) \".*\" \".*\""
           | unwrap bytes [$__range]))
  Result: 203.0.113.99 — top consumer, ~48MB in session 1

Step 2 [00:25-00:39] → grafana MCP: query_loki (path analysis)
  LogQL: {job="apache"} |= "203.0.113.99"
  Result: 26 lines — full session reconstructed

Step 3 [00:39-01:05] → grafana MCP: query_loki (session 2 check)
  LogQL: {job="apache"} |= "203.0.113.99" | ... [07 Jun range]
  Result: 10 more lines — return visit confirmed

Step 4 [01:05-01:32] → Fabric: netflow_baseline
  Input: 36 combined log lines from both sessions
  Output: Exfiltration signature detected — sequential /exports/ access,
          alphabetical pattern, non-crawler file types

Step 5 [01:32-02:12] → Fabric: create_cyber_summary
  Output: Final report (see Section 5)
```

---

## 5. AI-Generated Report (Actual Output)

---

## INCIDENT REPORT: Deliberate Data Exfiltration via HTTP

### SUMMARY
IP `203.0.113.99` conducted a **two-session data exfiltration operation** on 06–07 Jun 2026 via HTTP GET requests using `Wget/1.21.3`. The attacker systematically downloaded every file under `/exports/` and `/downloads/`, totaling an estimated **~86 MB of sensitive business data** across both sessions. The access pattern — sequential file access in alphabetical order, targeting named export files (customer lists, financial data, partner contacts) at 8 AM off-peak — is inconsistent with legitimate crawler behavior and consistent with deliberate data theft. This is **not** a misconfigured search engine bot.

**Overall Severity: 🔴 CRITICAL — Data breach in progress (or completed)**

---

### EXFILTRATION SUMMARY
| Session | Date | Time | Files Accessed | Estimated Data |
|---|---|---|---|---|
| 1 | 06 Jun 2026 | 08:00–08:09 | 17 files | ~48 MB |
| 2 | 07 Jun 2026 | 08:00–08:10 | 10 files | ~38 MB |
| **Total** | | | **27 unique files** | **~86 MB** |

---

### HIGH-VALUE DATA ACCESSED
| File | Size | Data Type | Regulatory Impact |
|---|---|---|---|
| `/exports/customer-list.csv` | 5.0 MB | Customer PII | 🔴 GDPR / Data breach notification required |
| `/exports/user-data-2026.csv` | 8.0 MB | User account data | 🔴 GDPR / Data breach notification required |
| `/exports/transaction-log.csv` | 10.0 MB | Financial records | 🔴 PCI-DSS relevant |
| `/exports/financial-summary.xlsx` | 3.5 MB | Financial statements | 🟡 Confidential business data |
| `/exports/partner-contacts.xlsx` | 2.0 MB | B2B relationship data | 🟡 Confidential business data |

---

### IOCS (Indicators of Compromise)
- **Attacker IP:** `203.0.113.99` (geolocation: requires enrichment — not a known CDN or search engine)
- **Tool Used:** `Wget/1.21.3 (linux-gnu)` — command-line, non-browser, non-legitimate-crawler
- **Session Pattern:** Returns same time (08:00 UTC+7) on consecutive mornings — **scheduled exfiltration**
- **Exfiltration Paths:** `/exports/*.csv`, `/exports/*.xlsx`, `/downloads/*.pdf`
- **Access Pattern:** Alphabetical sequential download — **deliberate enumeration, not random crawl**

---

### MITRE ATT&CK
| Tactic | Technique | ID | Evidence |
|---|---|---|---|
| Collection | Data from Network Shared Drive | **T1039** | Sequential HTTP GET of all `/exports/` files |
| Exfiltration | Exfiltration Over C2 Channel | **T1041** | HTTP 200 data transfer, ~86MB over 2 sessions |
| Exfiltration | Scheduled Transfer | **T1029** | Sessions at 08:00 on consecutive days |
| Reconnaissance | Search Open Websites/Domains | **T1593** | Prior reconnaissance implied — attacker knew exact file paths |

---

### RECOMMENDATION
**Immediate Actions (within the hour):**
1. **Block `203.0.113.99`** at the perimeter firewall immediately.
2. **Remove public HTTP access to `/exports/` and `/downloads/`** — these paths should require authentication. Implement immediately via web server config or WAF rule.
3. **Initiate Data Breach Assessment:** The `customer-list.csv` and `user-data-2026.csv` files contain PII. Under GDPR Article 33, a potential data breach must be reported to your supervisory authority within **72 hours** of becoming aware. Begin assessment now.

**Short-term (< 24 hours):**
1. Rotate all credentials present in any configuration files that may have been exported.
2. Audit all files under `/exports/` — determine if any contain credentials, API keys, or internal hostnames.
3. Enable authentication on all paths returning business data: `/api/v1/*`, `/exports/*`, `/downloads/*`.

**Long-term Hardening:**
1. **Never serve business data files directly via HTTP.** Implement a signed URL system (pre-signed S3 URLs, or equivalent) with expiry time.
2. Implement DLP (Data Loss Prevention) monitoring: alert when >10 MB is transferred to a single IP within 5 minutes.
3. Remove export files from the web server's document root entirely — serve them through an authenticated application layer only.

---

## 6. What the Manual Analyst Missed

The human analyst correctly identified `203.0.113.99` as the highest bandwidth consumer (~48 MB) but:

- Classified it as *"a misconfigured crawler — Wget-based, probably a monitoring script"*
- Did **not** flag the sequential `/exports/*.csv` pattern as exfiltration
- Did not check if the IP returned the next morning
- Did not identify the regulatory (GDPR) implications

The critical difference: the AI used `netflow_baseline` which analyzed the **sequence and semantics of paths accessed** (not just volume), correctly distinguishing targeted data theft from random crawling.

---

## 7. Reproduce This Case Study

```bash
# Prompt to use with /soc3analyst workflow:
# "Investigate bandwidth anomalies in Apache logs for 06-07 Jun 2026 via Grafana MCP.
#  Find the top bandwidth consumers and analyze what they accessed.
#  Determine if this is legitimate crawler activity or data exfiltration.
#  Use netflow_baseline then create_cyber_summary."
```
