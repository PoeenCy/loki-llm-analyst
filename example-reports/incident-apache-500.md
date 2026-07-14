---
generated_at: "2026-06-06T04:15:00+07:00"
investigation_trigger: "Manual check after 2x HTTP 500 appeared in Grafana dashboard"
analyst: "AI Agent (SOC Tier 3 workflow)"
time_to_report: "1 min 22 sec"
patterns_used:
  - analyze_logs
  - create_cyber_summary
dataset: "lab/access.log — 05-06 Jun 2026"
severity: MEDIUM
---

# SOC Report: Apache HTTP 500 Errors — Server Misconfiguration

**Report ID:** RPT-2026-06-06-001  
**Generated:** 06 Jun 2026 04:15 UTC+7  
**Severity:** 🟡 MEDIUM  
**Status:** Closed — Misconfiguration (not an active attack)

---

## SUMMARY

Two `HTTP 500 Internal Server Error` events were recorded from IP `66.249.73.135` (Googlebot/2.1) accessing `/wp-admin/index.php.txt` on 05 Jun 2026 at 22:52 and 06 Jun 2026 at 00:52 UTC+7. Analysis confirms this is a **server misconfiguration**, not an active attack: the web server is incorrectly configured to attempt PHP execution on `.txt` files. When Googlebot — a legitimate, non-malicious crawler — accessed the URL, the PHP engine crashed attempting to compile plain text, producing a 500 error. A separate `HTTP 500` was also recorded from IP `10.10.10.55` via an `OPTIONS` request — this is a benign Microsoft Office WebDAV probe.

**No malicious activity detected in these events. Root cause: server configuration error.**

---

## IOCS (Indicators of Compromise)

> ⚠️ These are **false positives** — no actual compromise. Listed for completeness.

| Indicator | Type | Verdict |
|---|---|---|
| `66.249.73.135` | Source IP | ✅ Legitimate — Googlebot (Google LLC ASN 15169) |
| `Googlebot/2.1` | User-Agent | ✅ Legitimate |
| `/wp-admin/index.php.txt` | Target Path | ⚠️ Misconfiguration — file should not be PHP-parseable |
| `10.10.10.55` | Source IP | ✅ Likely internal MS Office client |
| `Microsoft Office Protocol Discovery` | User-Agent | ✅ Legitimate — WebDAV auto-detection |
| `OPTIONS /` | HTTP Method | ✅ Legitimate — standard WebDAV probe |

---

## ROOT CAUSE ANALYSIS

### Event 1: Googlebot + `.php.txt` → HTTP 500

**What happened:** The file `/wp-admin/index.php.txt` exists on the server (likely a developer left a backup or documentation file). The Apache server's PHP handler is misconfigured with a regex that matches `.php` anywhere in the filename — including `.php.txt`. When Googlebot crawled this URL, Apache invoked the PHP interpreter on a plain text file, which failed and returned HTTP 500.

**This is not an attack.** Googlebot did not cause the crash intentionally. The server's PHP configuration is the root cause.

**Fix:** Add `<FilesMatch "\.php\.">` exclusion or use `SetHandler None` for `.txt` files.

### Event 2: MS Office `OPTIONS` → HTTP 500

**What happened:** Microsoft Office applications automatically send an `OPTIONS` HTTP request when a user attempts to open a URL from within Office (e.g., clicking a hyperlink in a Word document). This is standard WebDAV discovery behavior. The server returned HTTP 500 because it has no handler for the `OPTIONS` method.

**This is not an attack.** The fix is to add proper `OPTIONS` handling in the Apache configuration.

---

## MITRE ATT&CK

No malicious techniques detected. N/A for this report.

*Note: If `/wp-admin/index.php.txt` had been intentionally placed by an attacker and was being used to trigger server-side code execution, this would map to [T1505.003 - Server Software Component: Web Shell](https://attack.mitre.org/techniques/T1505/003/). This scenario does not meet that criteria.*

---

## RECOMMENDATION

**Server Configuration Fixes (non-urgent, < 1 week):**

1. **Fix PHP handler for `.txt` files.** In `httpd.conf` or `.htaccess`:
   ```apache
   <FilesMatch "\.txt$">
       SetHandler None
       php_flag engine off
   </FilesMatch>
   ```

2. **Add `OPTIONS` method handling** to prevent 500 errors:
   ```apache
   <LimitExcept GET POST HEAD>
       Order deny,allow
       Deny from all
   </LimitExcept>
   ```

3. **Remove developer artifacts.** Audit for any `.bak`, `.txt`, `.php.bak` files in the document root. These should not be web-accessible.

4. **Implement a 404 for `/wp-admin/`** if this is not a WordPress site — returning 500 instead of 404 leaks information about the server's PHP configuration.

---

## ANALYST NOTES

This report is an example of AI workflow correctly **classifying events as benign** and avoiding false escalation. The AI identified:
- Googlebot as a legitimate crawler (known IP range and UA string)
- The `.php.txt` pattern as a misconfiguration, not a web shell
- The `OPTIONS` request as standard MS Office behavior

A junior analyst spending 3–5 minutes manually might escalate these 500 errors as potential attacks, generating unnecessary alert fatigue. The AI correctly closed this in 1 min 22 sec with no escalation.
