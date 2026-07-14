# Case Study 01 — Web Scanner Detection

**Scenario:** Active reconnaissance by a vulnerability scanner  
**Dataset:** `lab/access.log`, 06–07 Jun 2026  
**MTTD (AI):** 1 min 38 sec  
**MTTD (Manual):** 9 min 45 sec  
**Result:** ✅ Detected

---

## 1. Context

On 06 Jun 2026 starting at 00:01 UTC+7, the web server received 25 rapid, sequential requests from IP `45.33.32.156` — all targeting sensitive paths (`.env`, `.git/HEAD`, admin panels, web shells) with the `Nikto/2.1.6` User-Agent. The following morning (07 Jun, 01:00), the same IP returned using `gobuster/3.6` and probed 20 additional paths.

These two sessions together form a complete reconnaissance phase: automated vulnerability scanning followed by directory brute-forcing.

**Timeline:**
```
06/Jun/2026 00:01:01  45.33.32.156  GET /.env          → 404  Nikto/2.1.6
06/Jun/2026 00:01:02  45.33.32.156  GET /.git/HEAD      → 404  Nikto/2.1.6
06/Jun/2026 00:01:03  45.33.32.156  GET /wp-login.php   → 404  Nikto/2.1.6
...
06/Jun/2026 00:01:18  45.33.32.156  GET /api/v1/        → 200  Nikto/2.1.6  ← HIT
06/Jun/2026 00:01:19  45.33.32.156  GET /api/v1/users   → 200  Nikto/2.1.6  ← HIT
...
07/Jun/2026 01:00:01  45.33.32.156  GET /.env.local     → 404  gobuster/3.6
07/Jun/2026 01:00:07  45.33.32.156  GET /.htaccess      → 403  gobuster/3.6  ← Sensitive
07/Jun/2026 01:00:16  45.33.32.156  GET /api/health     → 200  gobuster/3.6  ← HIT
```

---

## 2. Investigation Prompt

The exact prompt used to trigger the AI workflow:

```
Investigate the Apache logs in Grafana for the last 48 hours via the grafana MCP.
Focus on any IP addresses showing scanner or reconnaissance behavior — look for
high 404 rates, known scanner User-Agents, and probing of sensitive paths.
Pipe the raw findings through Fabric's recon_pattern, then create_cyber_summary.
```

---

## 3. AI Workflow Execution Trace

```
Step 1 [00:00-00:18] → grafana MCP: query_loki
  LogQL: {job="apache"} | json | status = "404"
         | summarize count by client_ip, useragent
  Result: 45.33.32.156 — 45 hits, UA: "Nikto/2.1.6" & "gobuster/3.6"

Step 2 [00:18-00:30] → grafana MCP: query_loki (targeted)
  LogQL: {job="apache"} |= "45.33.32.156"
  Result: 47 log lines across 2 sessions returned

Step 3 [00:30-00:50] → Fabric: analyze_logs
  Input: 47 raw log lines
  Output: Anomaly summary — high 404 rate, scanner UA fingerprinted

Step 4 [00:50-01:08] → Fabric: recon_pattern
  Input: analyze_logs output
  Output: Full reconnaissance report (see Section 4)

Step 5 [01:08-01:38] → Fabric: create_cyber_summary
  Input: recon_pattern output
  Output: Final structured report delivered to analyst
```

---

## 4. AI-Generated Report (Actual Output)

> *The following is the verbatim output from the `recon_pattern` → `create_cyber_summary` pipeline run against the sample dataset.*

---

## RECONNAISSANCE DETECTION REPORT

### SUMMARY
A two-phase automated reconnaissance campaign was detected originating from `45.33.32.156`. Phase 1 (06 Jun 2026, 00:01–00:01 UTC+7) employed **Nikto 2.1.6** — a known open-source web vulnerability scanner — probing 25 paths in 25 seconds. Two successful hits (HTTP 200) on `/api/v1/` and `/api/v1/users` indicate the attacker now knows the API surface exists and is unauthenticated-readable. Phase 2 (07 Jun 2026, 01:00–01:00 UTC+7) used **Gobuster 3.6** for directory brute-forcing, yielding one notable finding: `/api/health` returns 200 (infrastructure fingerprinting). Overall threat confidence: **HIGH**.

---

### RECON ACTORS
| Source IP | Country/ASN | Techniques Observed | Score | Confidence | First Seen | Last Seen |
|---|---|---|---|---|---|---|
| 45.33.32.156 | US / Linode (AS63949) | Nikto scan, Gobuster bruteforce, sensitive path probing | 9/10 | HIGH | 06/06 00:01 | 07/06 01:00 |

---

### TARGETED ASSETS
| Asset / Path | Hit Count | Status | Risk |
|---|---|---|---|
| `/api/v1/users` | 1 | **200 OK** | 🔴 HIGH — unauthenticated user list exposure |
| `/api/v1/` | 1 | **200 OK** | 🟡 MEDIUM — API endpoint confirmed |
| `/api/health` | 1 | **200 OK** | 🟡 MEDIUM — infrastructure fingerprint |
| `/.htaccess` | 1 | **403 Forbidden** | 🟡 MEDIUM — file exists, path confirmed |
| `.env`, `.git/HEAD`, `wp-login.php`, `phpmyadmin/`, `shell.php` | 1 each | 404 | 🟢 LOW — not found |

---

### IOCS (Indicators of Compromise)
- **Scanner IPs:** `45.33.32.156` (ASN: Linode LLC, US)
- **Scanner User-Agents:** `Nikto/2.1.6`, `gobuster/3.6`
- **Confirmed Exposed Paths:** `/api/v1/users` (HTTP 200, unauthenticated), `/api/health`
- **Sensitive Paths Confirmed to Exist:** `/.htaccess` (403)
- **Probed Paths (not found):** `.env`, `.git/HEAD`, `wp-login.php`, `phpmyadmin/`, `c99.php`, `shell.php`, `config.php`, `backup.zip`

---

### MITRE ATT&CK
| Tactic | Technique | ID | Evidence |
|---|---|---|---|
| Reconnaissance | Active Scanning: Vulnerability Scanning | **T1595.002** | Nikto/2.1.6 User-Agent, 25 sequential path probes in 1 second |
| Reconnaissance | Active Scanning: Wordlist Scanning | **T1595.003** | Gobuster/3.6, 20 directory probes |
| Discovery | Network Service Discovery | **T1046** | API endpoint enumeration — `/api/v1/` confirmed live |
| Collection | Data from Information Repositories | **T1213** | `/api/v1/users` returned HTTP 200 (potential data exposure) |

---

### RECOMMENDATION
**Immediate Actions (< 1 hour):**
1. **Block `45.33.32.156`** at WAF/firewall level. This IP has no legitimate business purpose on this server.
2. **Restrict `/api/v1/users`** — add authentication requirement immediately. An unauthenticated endpoint returning user data is a critical misconfiguration.
3. **Review `/api/health`** — ensure it does not leak version or infrastructure details in its response body.

**Short-term (< 24 hours):**
1. Add WAF rule to detect and block Nikto User-Agent string (`Nikto/`).
2. Implement rate limiting: any IP generating >20 unique 404s within 60 seconds should be auto-blocked.
3. Remove or restrict `/.htaccess` direct HTTP access — it should not be web-readable.

**Long-term Hardening:**
1. Deploy a Web Application Firewall (WAF) with OWASP CRS ruleset.
2. Add honeypot paths (e.g., `/admin-console`, `/backup.zip`) that trigger immediate alerts when accessed.
3. Implement API authentication on all `/api/v1/*` routes — no public endpoints should return sensitive data without a valid JWT/API key.

---

## 5. What the Manual Analyst Missed

The human analyst identified Nikto scanner activity (Session 1) but did not:
- Connect Session 2 (Gobuster, next morning) to the same source IP
- Flag `/api/v1/users` HTTP 200 as a critical misconfiguration
- Generate a MITRE-mapped report (noted as "too time-consuming")

Time to detection (manual): **9 min 45 sec** — and report was free-form, not structured.

---

## 6. Reproduce This Case Study

```bash
# Prompt to use with /soc3analyst workflow:
# "Investigate the last 48 hours of Apache logs via Grafana MCP.
#  Look specifically for scanner activity and reconnaissance patterns.
#  Pipe findings through recon_pattern and create_cyber_summary."

# Expected Loki query the agent should generate:
# {job="apache"} | json | status = "404" [last 48h]
# Then targeted: {job="apache"} |= "45.33.32.156" [last 48h]
```
