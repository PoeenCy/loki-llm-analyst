---
generated_at: "2026-06-07T08:15:00+07:00"
investigation_trigger: "NetOps alert — bandwidth spike detected on morning monitoring dashboard"
analyst: "AI Agent (SOC Tier 3 workflow)"
time_to_report: "2 min 12 sec"
patterns_used:
  - netflow_baseline
  - create_cyber_summary
dataset: "lab/access.log — 06-07 Jun 2026, 08:00-10:00 window"
severity: CRITICAL
gdpr_relevant: true
---

# SOC Report: Data Exfiltration via HTTP — 86MB Sensitive Data Transferred

**Report ID:** RPT-2026-06-07-002  
**Generated:** 07 Jun 2026 08:15 UTC+7  
**Severity:** 🔴 CRITICAL  
**Status:** OPEN — Immediate response required  
**GDPR:** ⚠️ Data breach notification may be required (72-hour clock starts now)

---

## SUMMARY

IP `203.0.113.99` conducted a **systematic, two-session data exfiltration operation** against this web server on 06–07 Jun 2026, using `Wget/1.21.3` to download every file available under `/exports/` and `/downloads/`. An estimated **~86 MB of sensitive business data** was transferred, including customer PII (GDPR-relevant), financial records (PCI-DSS relevant), and confidential business data. The access pattern — sequential alphabetical file download at 08:00 on consecutive mornings, targeting named export paths — is **inconsistent with any legitimate crawler or monitoring tool**. This is a confirmed data exfiltration event. Immediate containment is required, and a GDPR Data Breach Assessment must begin within the hour.

**Overall Severity: 🔴 CRITICAL**  
**Regulatory Impact: GDPR Article 33 — 72-hour notification window open**

---

## IOCS (Indicators of Compromise)

| Indicator | Type | Details |
|---|---|---|
| `203.0.113.99` | Attacker IP | Not a known CDN, search engine, or monitoring service |
| `Wget/1.21.3 (linux-gnu)` | Exfiltration Tool | Linux command-line downloader — non-browser, scripted |
| `/exports/customer-list.csv` | Data Exfiltrated | 5.0 MB — customer PII (GDPR: personal data) |
| `/exports/transaction-log.csv` | Data Exfiltrated | 10.0 MB — financial transaction records |
| `/exports/user-data-2026.csv` | Data Exfiltrated | 8.0 MB — user account data (GDPR: personal data) |
| `/exports/financial-summary.xlsx` | Data Exfiltrated | 3.5 MB — financial statements |
| `/exports/partner-contacts.xlsx` | Data Exfiltrated | 2.0 MB — B2B relationship data |
| `/exports/inventory.csv` | Data Exfiltrated | 4.0 MB — inventory data |
| `/exports/analytics-june.csv` | Data Exfiltrated | 6.0 MB — marketing analytics |
| `/downloads/*.pdf` | Data Exfiltrated | ~7 files, ~15 MB combined |

---

## EXFILTRATION TIMELINE

```
SESSION 1 — 06 Jun 2026
──────────────────────────────────────────────────────
08:00:00  203.0.113.99  GET /                            200  14 KB
08:00:05  203.0.113.99  GET /index.html                  200  14 KB   [fingerprinting]
08:00:10  203.0.113.99  GET /docs/api.html               200  33 KB   [API discovery]
08:00:15  203.0.113.99  GET /exports/customer-list.csv   200   5 MB   ← DATA THEFT BEGINS
08:01:30  203.0.113.99  GET /exports/transaction-log.csv 200  10 MB
08:04:00  203.0.113.99  GET /exports/user-data-2026.csv  200   8 MB
08:06:00  203.0.113.99  GET /exports/inventory.csv       200   4 MB
08:07:00  203.0.113.99  GET /exports/analytics-june.csv  200   6 MB
08:09:00  203.0.113.99  GET /exports/financial-summary.xlsx 200  3.5 MB
08:10:00  203.0.113.99  GET /exports/partner-contacts.xlsx  200  2.0 MB
... (additional downloads: PDFs, images)
SESSION 1 TOTAL: ~48 MB in 9 minutes

SESSION 2 — 07 Jun 2026
──────────────────────────────────────────────────────
08:00:00  203.0.113.99  GET /                            200  14 KB
08:00:15  203.0.113.99  GET /exports/customer-list.csv   200   5 MB   ← RETURNS FOR MORE
08:01:30  203.0.113.99  GET /exports/transaction-log.csv 200  10 MB
08:04:00  203.0.113.99  GET /exports/user-data-2026.csv  200   8 MB
08:06:00  203.0.113.99  GET /exports/inventory.csv       200   4 MB
08:07:00  203.0.113.99  GET /exports/analytics-june.csv  200   6 MB
... (additional)
SESSION 2 TOTAL: ~38 MB in 10 minutes

GRAND TOTAL: ~86 MB transferred in 2 sessions over 2 days
```

**Behavioral signature of deliberate exfiltration (vs. crawler):**
- ✅ Wget, not a browser or legitimate bot
- ✅ Targets `/exports/` — not web pages, but data files
- ✅ Sequential alphabetical access = scripted enumeration, not random
- ✅ Returns next morning at same time = scheduled, automated operation
- ✅ Downloads PDFs and XLSXs = document collection, not website indexing

---

## MITRE ATT&CK

| Tactic | Technique | ID | Evidence |
|---|---|---|---|
| Collection | Data from Network Shared Drive | **T1039** | Sequential HTTP GET of all `/exports/` files |
| Exfiltration | Exfiltration Over Web Service | **T1567** | HTTP GET used as exfiltration channel |
| Exfiltration | Scheduled Transfer | **T1029** | 08:00 sessions on consecutive mornings |
| Reconnaissance | Gather Victim Org Information | **T1591** | Downloaded financial summary, partner contacts, customer list |

---

## DATA BREACH ASSESSMENT

| Data Category | Files | Estimated Records | Regulatory Framework |
|---|---|---|---|
| Customer PII | `customer-list.csv` | Unknown | **GDPR Art. 33 & 34** — breach notification required |
| User account data | `user-data-2026.csv` | Unknown | **GDPR Art. 33 & 34** — breach notification required |
| Financial transactions | `transaction-log.csv` | Unknown | **PCI-DSS** — incident response required |
| Financial statements | `financial-summary.xlsx` | N/A | Confidential business data |
| Partner contacts | `partner-contacts.xlsx` | Unknown | GDPR if EU contacts included |

> ⚠️ Under **GDPR Article 33**, if personal data has been accessed by an unauthorized party, the supervisory authority must be notified within **72 hours** of becoming aware of the breach. The clock started when this alert was generated. **Contact your DPO (Data Protection Officer) immediately.**

---

## RECOMMENDATION

**Immediate Actions (next 30 minutes):**

1. **Block `203.0.113.99`** at perimeter firewall — immediately.

2. **Take `/exports/` offline** — rename or remove directory from web root. These files must not be publicly HTTP-accessible under any circumstances.
   ```bash
   # On the web server:
   mv /var/www/html/exports /var/www/html/exports_BLOCKED_$(date +%Y%m%d)
   ```

3. **Begin GDPR breach assessment:**
   - Determine exact record count in `customer-list.csv` and `user-data-2026.csv`
   - Identify if EU data subjects are included
   - Contact DPO within 1 hour

4. **Rotate credentials:** If any configuration or credentials were exposed in the downloaded files (check `financial-summary.xlsx` and API configs), rotate immediately.

**Short-term (< 24 hours):**

1. **Move data files out of web root** entirely. Serve via authenticated application layer only.
2. **Implement authentication** on `/api/v1/*` and all data download paths.
3. **Notify affected parties** per GDPR Article 34 if high risk to individuals is determined.

**Long-term Hardening:**

1. **Never serve business data files directly from a web server.** Use pre-signed URLs with short expiry (15 minutes) from object storage (S3, GCS, Azure Blob).
2. **Implement DLP monitoring:** Alert when >5 MB is transferred to a single IP within 5 minutes.
3. **Audit all web-accessible directories** — use `find /var/www/html -name "*.csv" -o -name "*.xlsx"` to identify any remaining business data files accessible via HTTP.

---

## ATTRIBUTION NOTE

IP `203.0.113.99` does not match any known CDN, search engine, or monitoring service IP range. The use of `Wget/1.21.3 (linux-gnu)` is consistent with a Linux-based script or server executing the download. Prior reconnaissance of `/docs/api.html` before downloading exports suggests the attacker knew the API structure — implying either prior access or the reconnaissance from `45.33.32.156` (Case Study 01) was used to map the server. These two IPs may be part of the same threat actor campaign.

**Recommended further investigation:** Check if `203.0.113.99` and `45.33.32.156` share the same ASN, hosting provider, or appeared in the same timeframe on other servers in your infrastructure.
