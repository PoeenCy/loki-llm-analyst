---
generated_at: "2026-06-07T06:00:00+07:00"
investigation_trigger: "Proactive threat hunt — morning review of prior 48 hours"
analyst: "AI Agent (SOC Tier 3 workflow, Threat Hunt mode)"
time_to_report: "1 min 38 sec"
patterns_used:
  - analyze_logs
  - recon_pattern
  - create_cyber_summary
dataset: "lab/access.log — 06-07 Jun 2026"
severity: HIGH
---

# SOC Report: Active Reconnaissance Campaign — Nikto + Gobuster

**Report ID:** RPT-2026-06-07-001  
**Generated:** 07 Jun 2026 06:00 UTC+7  
**Severity:** 🔴 HIGH  
**Status:** Open — Awaiting WAF block + API remediation  
**Analyst Action Required:** Block attacker IP, restrict `/api/v1/users`

---

## SUMMARY

A two-phase, coordinated reconnaissance campaign was detected from IP `45.33.32.156`. Phase 1 occurred on 06 Jun 2026 at 00:01 UTC+7 using **Nikto 2.1.6** — a known open-source vulnerability scanner — generating 25 requests in 25 seconds against sensitive paths. Phase 2 occurred on 07 Jun 2026 at 01:00 UTC+7 using **Gobuster 3.6**, a directory brute-force tool, generating 20 additional requests. **Two critical findings emerged from this campaign:** (1) `/api/v1/users` returned HTTP 200 to an unauthenticated request — the attacker now has confirmed knowledge of the API and potentially a user list, and (2) `/api/health` returned HTTP 200 — infrastructure fingerprinted. This is a pre-attack reconnaissance phase. An exploitation attempt is likely imminent.

**Overall Severity: 🔴 HIGH**

---

## IOCS (Indicators of Compromise)

| Indicator | Type | Details |
|---|---|---|
| `45.33.32.156` | Attacker IP | ASN: Linode LLC (AS63949), US — cloud hosting commonly used by threat actors |
| `Nikto/2.1.6` | Scanner Tool | Open-source web vulnerability scanner |
| `gobuster/3.6` | Scanner Tool | Directory/file brute-force tool |
| `/api/v1/users` | Exposed Endpoint | Returned **HTTP 200** to unauthenticated GET — critical misconfiguration |
| `/api/v1/` | Confirmed Endpoint | Returned HTTP 200 — API exists and is partially unauthenticated |
| `/api/health` | Confirmed Endpoint | Returned HTTP 200 — infrastructure fingerprinted |
| `/.htaccess` | Sensitive File Probe | Returned HTTP 403 — file confirmed to exist |

**Paths probed but not found (HTTP 404):**
`.env`, `.git/HEAD`, `wp-login.php`, `wp-admin/`, `phpmyadmin/`, `shell.php`, `c99.php`, `r57.php`, `backup.zip`, `db_backup.sql`, `setup.php`, `install.php`, `actuator/health`, `actuator/env`, `api/swagger-ui`

---

## ATTACK TIMELINE

```
06/Jun/2026 00:01:01  Phase 1 begins — Nikto scanner
  → 25 requests in 25 seconds (1 req/sec — rate-limited to evade detection)
  → 2 hits: /api/v1/ (200), /api/v1/users (200)
  → 1 sensitive find: /.htaccess (403 — file exists)
  → 23 probes returned 404 (paths not found)

07/Jun/2026 01:00:01  Phase 2 begins — Gobuster directory brute-force
  → 20 requests in 20 seconds
  → 1 hit: /api/health (200)
  → 1 sensitive find: /.htaccess (403 — confirms prior finding)
  → 18 probes returned 404

CURRENT STATUS: Reconnaissance complete. Attacker has:
  ✓ Confirmed API exists: /api/v1/
  ✓ Retrieved user list (or confirmed endpoint): /api/v1/users
  ✓ Confirmed infrastructure running: /api/health
  ✓ Identified .htaccess exists (possible future exploit target)
  ✗ No admin panel, no web shell, no config files found
```

---

## MITRE ATT&CK

| Tactic | Technique | ID | Evidence |
|---|---|---|---|
| Reconnaissance | Active Scanning: Vulnerability Scanning | **T1595.002** | Nikto/2.1.6 — 25 probes in 25 seconds |
| Reconnaissance | Active Scanning: Wordlist Scanning | **T1595.003** | Gobuster/3.6 — 20 directory probes |
| Discovery | Network Service Discovery | **T1046** | `/api/v1/` and `/api/health` confirmed live |
| Collection | Data from Information Repositories | **T1213** | `/api/v1/users` → HTTP 200 (potential user data retrieved) |

**Predicted Next Phase (if not blocked):**
| Tactic | Technique | ID | Prediction Basis |
|---|---|---|---|
| Credential Access | Brute Force: Password Spraying | **T1110.003** | Confirmed `/api/v1/users` list enables targeted credential attack |
| Initial Access | Exploit Public-Facing Application | **T1190** | API endpoint confirmed, attacker will probe for injection vulnerabilities |

---

## RECOMMENDATION

**Immediate Actions (< 1 hour):**

1. **Block `45.33.32.156`** at perimeter firewall. Add entire subnet `45.33.32.0/24` if feasible. This IP has zero legitimate business purpose.

2. **Add authentication to `/api/v1/users` immediately.** This endpoint returning HTTP 200 to an unauthenticated request is a critical security misconfiguration. Even if it returns an empty array, the attacker now knows the endpoint structure. If it returned actual user data — assume that data is compromised.

3. **Review `/api/health` response body.** Ensure it does not leak version numbers, hostnames, or dependency information. It should return only `{"status": "ok"}`.

**Short-term (< 48 hours):**

1. **Implement WAF rules** to block Nikto and Gobuster User-Agent strings:
   - Block: `Nikto/`, `gobuster/`, `dirbuster/`, `wfuzz/`, `sqlmap/`

2. **Rate limiting:** Configure nginx/Apache to return HTTP 429 for IPs generating >20 unique 404 responses within 60 seconds.

3. **Deploy honeypot paths:** Add monitoring on `/admin-console`, `/backup.zip`, `/wp-admin/` — any access to these paths should trigger an immediate alert (these paths are guaranteed to be non-legitimate traffic on this server).

4. **Audit entire `/api/` namespace:** Enumerate all routes and verify each requires appropriate authentication.

**Long-term:**

1. Deploy a WAF with OWASP CRS v3.3+ — it would have blocked both Nikto and Gobuster automatically.
2. Implement API gateway with mandatory JWT authentication on all routes.
3. Remove `.htaccess` from web-accessible paths or ensure it's not readable via HTTP.

---

## CONFIDENCE ASSESSMENT

| Finding | Confidence | Basis |
|---|---|---|
| Nikto scanner activity | 🔴 VERY HIGH | Exact UA string match, probe pattern, timing |
| Gobuster scanner activity | 🔴 VERY HIGH | Exact UA string match, wordlist pattern |
| `/api/v1/users` data exposure | 🟡 MEDIUM-HIGH | HTTP 200 confirmed, actual response body unknown |
| Attacker will return | 🟡 MEDIUM | Standard recon → exploit pattern, but not guaranteed |

---

## HUNT QUERY (Ongoing Monitoring)

Run this LogQL query in Grafana to monitor for follow-up activity from this actor:

```logql
{job="apache"} |= "45.33.32.156"
```

Also monitor for credential attacks following this reconnaissance:

```logql
{job="apache"} |= "/api/v1/login" | status = "401"
| count_over_time [5m] > 5
```
