# Loki-LLM Analyst

Loki-LLM Analyst is an LLM-powered SOC analyst workflow. It utilizes the Grafana Model Context Protocol (MCP) to query Loki logs, processes the raw data using predefined Fabric security patterns, and outputs structured, MITRE ATT&CK-aligned reports for human review.

## Architecture and Rationale

The primary objective of this architecture is not full automation, but cognitive offloading during critical incident response scenarios. 

In traditional workflows, analysts are required to manually construct LogQL queries, sift through raw log streams, and cross-reference findings with threat frameworks. This process is time-consuming and prone to human error, particularly during off-hours alerts. 

This project delegates log extraction and initial pattern matching to an LLM, while strictly maintaining a human-in-the-loop for final containment or escalation decisions.

### System Boundaries

To ensure security and predictability, the architecture enforces a strict separation between the AI agent and the infrastructure:

1. **Layer 1 (Analyst Environment):** The LLM agent operates locally. It interfaces with the Fabric framework for structured analysis and uses the Grafana MCP server to securely request data.
2. **Layer 2 (Cloud Infrastructure):** The production environment containing Grafana, Loki, and Promtail. 

The AI agent does not have direct access to the Loki endpoint. All queries are brokered through the Grafana API via the MCP server using a read-only service account.

```text
[ LLM Agent ] ---> [ Grafana MCP ] ===(HTTPS)===> [ Grafana API ] ---> [ Loki ]
      |
      v
[ Fabric Patterns ]
      |
      v
[ Structured Report ]
```

## Key Features

- **Strict Access Control:** The infrastructure exposes only standard web ports (80/443). The AI operates entirely on the client side.
- **Reproducible Analysis:** By piping LLM outputs through immutable [Fabric](https://github.com/danielmiessler/fabric) patterns, the system mitigates hallucinated reporting formats and ensures consistency.
- **Standardized Reporting:** All findings are strictly categorized into a predictable structure: Summary, IOCs, MITRE ATT&CK mapping, and Recommendations.

## Deployment Guide

### 1. Infrastructure Setup

The cloud environment requires Docker and Docker Compose. Caddy is utilized for automatic SSL termination via `nip.io`.

```bash
export DOMAIN="<YOUR_SERVER_IP>.nip.io"
sudo -E docker-compose up -d
```

### 2. Grafana Configuration

Generate a read-only service account for the MCP server:
1. Navigate to Grafana UI > Administration > Service Accounts.
2. Create an account named `antigravity-analyst` with the `Viewer` role.
3. Generate and securely store the API token.

### 3. Local Environment Setup

Install Go (1.21+) and the Fabric framework:

```bash
go install github.com/danielmiessler/fabric/cmd/fabric@latest
fabric --update
```

Install the Python package manager `uv` to run the Grafana MCP server. Configure your AI IDE with the following MCP settings:

```json
{
    "mcpServers": {
        "grafana": {
            "command": "uvx",
            "args": ["mcp-grafana"],
            "env": {
                "GRAFANA_URL": "https://<YOUR_SERVER_IP>.nip.io",
                "GRAFANA_API_KEY": "<YOUR_GRAFANA_TOKEN>"
            }
        }
    }
}
```
*(Note for Windows users: You may need to provide the absolute path to the `uvx` executable in the command field).*

### 4. Usage

Invoke the workflow through your AI IDE. Example prompt:

> "Investigate Apache logs for the last 15 minutes via Grafana MCP. Pipe the results through Fabric's `analyze_logs` and `create_cyber_summary` patterns, and output the final report."

## References

- [Grafana MCP Server](https://github.com/grafana/mcp-grafana)
- [Fabric Framework](https://github.com/danielmiessler/fabric)
- [uv](https://github.com/astral-sh/uv)
