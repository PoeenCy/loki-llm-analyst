# Threat Model: loki-llm-analyst System

> This document models the security threats **to this system itself**, not the threats it is designed to detect in others.

---

## Assets to Protect

| Asset | Sensitivity | Owner |
|---|---|---|
| Grafana API Token (read-only) | HIGH | Analyst |
| LLM API Key (OpenAI/Anthropic) | HIGH | Analyst |
| Log data returned by Loki | MEDIUM | Infrastructure team |
| AI-generated reports | MEDIUM | SOC team |
| MCP config file (`mcp-grafana.json`) | HIGH | Analyst |

---

## Threat Actors

| Actor | Motivation | Capability |
|---|---|---|
| External attacker (internet) | Steal Grafana token to access log data | Medium (opportunistic) |
| Malicious LLM response (prompt injection) | Exfiltrate log data, manipulate report | Low (constrained by Fabric patterns) |
| Compromised Fabric pattern | Alter analysis output | Low (patterns are version-controlled) |
| Insider threat (analyst machine) | Steal tokens from config files | High (local access) |

---

## Threat Analysis (STRIDE)

### S — Spoofing

| Threat | Impact | Mitigation |
|---|---|---|
| Attacker spoofs Grafana API endpoint | Agent sends queries to malicious server, leaking API token | Pin Grafana URL in MCP config; use HTTPS with cert validation |
| Attacker spoofs Loki data | Agent analyzes tampered logs, produces wrong report | Read-only token; Grafana audit log; human review of conclusions |

### T — Tampering

| Threat | Impact | Mitigation |
|---|---|---|
| Attacker modifies Fabric pattern files | Analysis output is manipulated (wrong MITRE, hidden IOCs) | Store patterns in git; verify `git status` before analysis |
| Attacker modifies `access.log` before analysis | Evidence tampering in forensic investigation | Log integrity via Promtail → Loki immutable storage; do not use modified logs |

### R — Repudiation

| Threat | Impact | Mitigation |
|---|---|---|
| Analyst denies making a containment decision | No accountability for actions taken | Reports include timestamps; human decisions logged separately |

### I — Information Disclosure

| Threat | Impact | Mitigation |
|---|---|---|
| Grafana API token in `mcp-grafana.json` committed to git | Token exposed publicly | `.gitignore` covers `mcp-grafana.json` and `.env*` |
| Log data sent to LLM API (OpenAI/Anthropic) | PII or sensitive log content leaves the organization | Use local Ollama model for sensitive environments; review Fabric's model routing |
| LLM prompt injection via malicious log content | Log payload crafted to exfiltrate data via LLM | Fabric pattern output is structured text, not executed code; no exfiltration vector |

### D — Denial of Service

| Threat | Impact | Mitigation |
|---|---|---|
| Agent runs runaway Loki query (huge time range) | Loki backend DoS | Grafana enforces 10-second query timeout; use `$__interval` |
| LLM context window overflow (too many log lines) | Truncated analysis, missed findings | Limit Loki query to `limit=500` lines; agent should paginate |

### E — Elevation of Privilege

| Threat | Impact | Mitigation |
|---|---|---|
| Grafana Viewer token used beyond its scope | Access to non-log datasources, admin functions | Token is `Viewer` role — cannot write, cannot access admin APIs |
| Agent executes code from LLM response | Arbitrary code execution on analyst machine | Agent does not execute arbitrary shell commands; Fabric patterns only |

---

## Security Controls Summary

| Control | Implementation |
|---|---|
| Principle of Least Privilege | Grafana service account with `Viewer` role only |
| Secrets Management | All tokens in `.env` (gitignored), not in code |
| Audit Logging | Grafana query audit log; agent responses logged in IDE |
| Immutable Log Storage | Loki's append-only storage; Promtail ships directly to Loki |
| Network Boundary | AI agent cannot reach Loki directly — must go through Grafana API |
| Pattern Integrity | Fabric patterns in git; changes are tracked and reviewable |
| Human Oversight | No automated containment — all actions require human approval |

---

## Residual Risks (Accepted)

These risks are known and accepted as part of the current design:

1. **LLM API data exposure:** When using cloud LLM APIs (OpenAI, Anthropic), log excerpts are sent to the LLM provider. For environments with strict data residency requirements, use a local model (Ollama with Llama 3, Mistral, etc.).

2. **No token rotation:** The Grafana API token is long-lived (set to "no expiry" in the setup guide). For production use, implement 90-day token rotation.

3. **Analyst machine trust:** This model assumes the analyst's machine is trustworthy. A compromised analyst machine could exfiltrate all tokens and log data. Standard endpoint security (EDR, disk encryption) mitigates this.
