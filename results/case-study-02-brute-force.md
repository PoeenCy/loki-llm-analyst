# Case Study 02 — Brute Force Attack & Credential Theft

**Scenario:** Automated credential brute-force leading to successful login and post-compromise enumeration  
**Dataset:** `lab/access.log`, 06 Jun 2026  
**MTTD (AI):** 1 min 46 sec  
**MTTD (Manual):** 9 min 45 sec  
**Result:** ✅ Detected

---

## 1. Context

At 04:00:01 UTC+7 on 06 Jun 2026, IP `185.220.101.45` began a rapid-fire brute-force attack against the login endpoint `/api/v1/login`. The attacker used `curl/7.88.1` — a command-line HTTP client, not a browser — indicating an automated script. Over a span of **22 seconds**, 11 consecutive `POST` requests returned `HTTP 401 Unauthorized`. On the 12th attempt (04:00:23), the response changed to `HTTP 200 OK`.

Within 11 seconds of gaining access, the attacker:
1. Queried `/api/v1/users` (HTTP 200) — **user enumeration**
2. Queried `/api/v1/admin/config` (HTTP 200) — **configuration data access**
3. Attempted `/api/v1/admin/secrets` (HTTP 403) — **escalation attempt blocked**

**Full Attack Timeline:**
```
04:00:01  185.220.101.45  POST /api/v1/login  → 401  [attempt  1]
04:00:03  185.220.101.45  POST /api/v1/login  → 401  [attempt  2]
04:00:05  185.220.101.45  POST /api/v1/login  → 401  [attempt  3]
04:00:07  185.220.101.45  POST /api/v1/login  → 401  [attempt  4]
04:00:09  185.220.101.45  POST /api/v1/login  → 401  [attempt  5]
04:00:11  185.220.101.45  POST /api/v1/login  → 401  [attempt  6]
04:00:13  185.220.101.45  POST /api/v1/login  → 401  [attempt  7]
04:00:15  185.220.101.45  POST /api/v1/login  → 401  [attempt  8]
04:00:17  185.220.101.45  POST /api/v1/login  → 401  [attempt  9]
04:00:19  185.220.101.45  POST /api/v1/login  → 401  [attempt 10]
04:00:21  185.220.101.45  POST /api/v1/login  → 401  [attempt 11]
04:00:23  185.220.101.45  POST /api/v1/login  → 200  ← CREDENTIAL COMPROMISED
04:00:28  185.220.101.45  GET  /api/v1/users        → 200  ← USER ENUMERATION
04:00:31  185.220.101.45  GET  /api/v1/admin/config → 200  ← CONFIG ACCESS
04:00:34  185.220.101.45  GET  /api/v1/admin/secrets→ 403  ← BLOCKED
```

---

## 2. Investigation Prompt

```
Investigate the Apache logs for the last 24 hours via Grafana MCP.
I'm looking for any brute-force or credential stuffing activity against
the /api/v1/login endpoint. Include post-authentication behavior if a
login succeeded. Pipe through analyze_logs then analyze_malware,
and finish with create_cyber_summary.
```

---

## 3. AI Workflow Execution Trace

```
Step 1 [00:00-00:22] → grafana MCP: query_loki
  LogQL: {job="apache"} |= "/api/v1/login" | status = "401"
  Result: 11 lines, all from 185.220.101.45 within 22 seconds

Step 2 [00:22-00:31] → grafana MCP: query_loki (pivot)
  LogQL: {job="apache"} |= "/api/v1/login" | status = "200"
  Result: 1 line — 04:00:23, 185.220.101.45 → LOGIN CONFIRMED

Step 3 [00:31-00:46] → grafana MCP: query_loki (post-auth)
  LogQL: {job="apache"} |= "185.220.101.45"
  Result: 14 lines total — all activity from this IP

Step 4 [00:46-01:05] → Fabric: analyze_logs
  Input: 14 lines
  Output: Brute force pattern identified, post-auth access flagged

Step 5 [01:05-01:28] → Fabric: analyze_malware
  Input: analyze_logs output
  Output: IOC extraction — IP, UA, targeted endpoints, timeline

Step 6 [01:28-01:46] → Fabric: create_cyber_summary
  Output: Final structured report (see Section 4)
```

---

## 4. AI-Generated Report (Actual Output)

---

## INCIDENT REPORT: Credential Brute Force & Post-Compromise Enumeration

### SUMMARY
On 06 Jun 2026 at 04:00 UTC+7, IP `185.220.101.45` executed a successful credential brute-force attack against `/api/v1/login` using an automated `curl`-based script. After **11 failed attempts in 22 seconds**, the attacker gained authenticated access on attempt #12. Within 11 seconds of authentication, the attacker performed post-compromise enumeration: accessing the user list and administrative configuration data. An attempt to access `/api/v1/admin/secrets` was blocked (HTTP 403). **This is a confirmed compromise.** The compromised credential must be rotated immediately.

**Overall Severity: 🔴 CRITICAL**

---

### IOCS (Indicators of Compromise)
- **Attacker IP:** `185.220.101.45`
- **Attack Tool (User-Agent):** `curl/7.88.1` — non-browser, confirmed automated
- **Attack Vector:** `/api/v1/login` endpoint — no rate limiting, no lockout policy
- **Compromised Endpoint:** `POST /api/v1/login` → HTTP 200 at 04:00:23
- **Data Accessed Post-Compromise:**
  - `GET /api/v1/users` → HTTP 200 (user list retrieved)
  - `GET /api/v1/admin/config` → HTTP 200 (configuration data retrieved)
- **Blocked Escalation Attempt:** `GET /api/v1/admin/secrets` → HTTP 403

---

### ATTACK TIMELINE
```
04:00:01 – 04:00:21  Brute force phase (11 attempts, 2-second cadence)
04:00:23             Authentication successful — credentials compromised
04:00:28             Post-auth: /api/v1/users accessed (user enumeration)
04:00:31             Post-auth: /api/v1/admin/config accessed (config leak)
04:00:34             Escalation attempt blocked: /api/v1/admin/secrets (403)
```

---

### MITRE ATT&CK
| Tactic | Technique | ID | Evidence |
|---|---|---|---|
| Credential Access | Brute Force: Password Guessing | **T1110.001** | 11 × HTTP 401 from same IP in 22 seconds |
| Initial Access | Valid Accounts | **T1078** | HTTP 200 on login attempt #12 |
| Discovery | Account Discovery | **T1087** | `GET /api/v1/users` → HTTP 200 |
| Discovery | System Information Discovery | **T1082** | `GET /api/v1/admin/config` → HTTP 200 |
| Privilege Escalation | Exploitation for Privilege Escalation | **T1068** | `GET /api/v1/admin/secrets` → HTTP 403 (attempted, blocked) |

---

### RECOMMENDATION
**Immediate Actions (within the hour):**
1. **Rotate the compromised credential.** The account that was brute-forced must be considered fully compromised — rotate the password, invalidate all active sessions and tokens.
2. **Block `185.220.101.45`** at the perimeter firewall. Also block its /24 subnet (`185.220.101.0/24`) — this ASN is associated with known Tor exit nodes and malicious hosting.
3. **Audit what `/api/v1/admin/config` returned.** Determine what configuration data was exposed. If it includes database credentials, API keys, or internal hostnames — treat those as compromised and rotate them.

**Short-term (< 24 hours):**
1. **Implement account lockout policy:** Lock account after 5 failed login attempts within 5 minutes.
2. **Add rate limiting** to `/api/v1/login`: max 5 requests per IP per minute.
3. **Require MFA** for all admin-level accounts.

**Long-term Hardening:**
1. Implement CAPTCHA or Proof-of-Work on the login endpoint for non-authenticated clients.
2. Use a secrets manager (HashiCorp Vault, AWS Secrets Manager) — never expose secrets via an HTTP API endpoint.
3. Set up a SIEM alert rule: `POST /api/v1/login status=401 count>5 within 5m from same IP → ALERT`.

---

## 5. What the Manual Analyst Missed

The human analyst correctly identified the brute-force pattern but:
- Took 4 minutes 10 seconds just to isolate the 401 sequence
- Did **not** pivot to check post-authentication activity — missed the `/api/v1/admin/config` access
- Did not produce a MITRE-mapped report

The AI workflow pivoted automatically to post-auth behavior in Step 3, surfacing the most critical finding (config data access) that the human analyst missed entirely.

---

## 6. Reproduce This Case Study

```bash
# Prompt to use with /soc3analyst workflow:
# "Investigate the Apache logs for the last 24 hours via Grafana MCP.
#  Look for brute-force or credential stuffing against /api/v1/login.
#  If a login succeeded, trace all post-authentication activity from that IP.
#  Run analyze_logs, then analyze_malware, then create_cyber_summary."
```
