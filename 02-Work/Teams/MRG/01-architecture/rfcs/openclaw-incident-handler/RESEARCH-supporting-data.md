---
title: OpenClaw Incident Handler - Supporting Data & Research
type: research
created: '2026-03-18'
tags:
  - openclaw
  - research
  - mcp
  - skills
  - integration
---
# OpenClaw Incident Handler - Supporting Data & Research

## Overview

Dokumen ini berisi hasil riset tentang MCP servers, skills, dan integrasi yang tersedia untuk mendukung implementasi OpenClaw Incident Handler.

---

## 1. GitLab Integration

### 1.1 Official MCP/Skill Options

#### Composio GitLab MCP Server
GitLab MCP server adalah implementasi dari Model Context Protocol yang menghubungkan AI agent langsung ke akun GitLab. Menyediakan akses terstruktur dan aman ke repositories, projects, dan issues, sehingga agent bisa perform actions seperti creating projects, managing issues, handling branches, dan automating DevOps workflows.

**Available Tools dari Composio GitLab MCP:**
- `Create GitLab Group` - Create a new group in GitLab
- `Create Project` - Create a new project in GitLab
- `Create Project Issue` - Create a new issue in a GitLab project
- `Create Repository Branch` - Create a new branch in a project
- `Get Job Details` - Retrieve details of a single job
- `Get Commit References` - Get all references a commit is pushed to

**Integration URL:**
```
https://connect.composio.dev/mcp
```

**Config Example:**
```json
{
  "plugins": {
    "entries": {
      "composio": {
        "enabled": true,
        "config": {
          "consumerKey": "ck_your_key_here"
        }
      }
    }
  }
}
```

#### Community Skills

**glab-cli skill:**
Interact with GitLab using the glab CLI. Tersedia di awesome-openclaw-skills repository.

---

## 2. Elasticsearch / ELK Integration

### 2.1 Official Elasticsearch Skill

Connect to Elasticsearch clusters for full-text search, log analysis, and data aggregation using natural language.

### 2.2 Custom Elasticsearch Skill (Elastic Labs Tutorial)

Tutorial dari Elastic Labs menunjukkan cara membuat custom read-only skill untuk OpenClaw yang mengakses dan query Elasticsearch data. Solusi terdiri dari tiga integrated layers yang bekerja bersama melalui OpenClaw orchestration.

**Architecture dari tutorial:**
1. **Data Layer** - Elasticsearch via start-local (single command untuk spin up ES + Kibana dengan Docker)
2. **Sample Indices:**
   - `fresh_produce` - 10 products dengan semantic search (ecommerce)
   - `app-logs-synthetic` - 30 log entries across four services (observability)

Skill yang sama bisa bekerja dengan kedua indices tanpa reconfiguration; agent menginspeksi mapping dan mengadaptasi queries-nya secara otomatis.

**Setup dari tutorial:**
```bash
# Start local ES + Kibana
# Uses start-local command

# Elasticsearch at http://localhost:9200
# Kibana at http://localhost:5601
# Credentials in elastic-start-local/.env
```

### 2.3 ELK Stack Monitoring Integration

Docker Compose untuk ELK monitoring OpenClaw tersedia di community guide dengan konfigurasi Elasticsearch, Logstash, dan Kibana.

**Docker Compose Example:**
```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - /var/log/openclaw:/logs:ro
    depends_on:
      - elasticsearch
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
```

---

## 3. Git/Code Analysis Integration

### 3.1 Available Skills

#### git-summary
Skill ini memberikan comprehensive overview dari current Git repository state termasuk status, recent commits, branches, dan contributors.

**Capabilities:**
- Current branch & status
- Recent commits (last 10)
- Local & remote branches
- Remotes with URLs
- Uncommitted changes summary

#### git-workflows
Advanced git operations beyond add/commit/push.

#### GitHub MCP Server
GitHub MCP Server untuk repository management, file operations, PR/issue tracking, branch management, dan GitHub API integration. Enable AI agents untuk clone repos, read code, create/update files, manage issues dan pull requests.

**Installation:**
```bash
# Community-maintained GitHub MCP server
npm install -g @modelcontextprotocol/server-github

# Or build from source
git clone https://github.com/modelcontextprotocol/servers-archived
cd servers-archived/src/github
npm install
npm run build
```

### 3.2 Code Analysis Skills

#### QA Architecture Auditor
QA Architecture Auditor adalah OpenClaw skill yang perform deep static analysis dan generate HTML atau Markdown report yang serve sebagai complete QA strategy. Termasuk forensic codebase analysis yang detect languages, frameworks, architecture pattern (monolith, microservices, serverless), dependencies, modules, cyclomatic complexity, dan lainnya.

**Capabilities:**
- Risk assessment (score 0-100) based on complexity, external calls, auth handling, data persistence, cryptography, file I/O, coupling, public API surface

---

## 4. Slack Integration

### 4.1 Native OpenClaw Slack Channel

Status: production-ready untuk DMs + channels via Slack app integrations. Default mode adalah Socket Mode; HTTP Events API mode juga didukung.

**Socket Mode Config:**
```yaml
channels:
  slack:
    enabled: true
    mode: "socket"
    appToken: "xapp-..."
    botToken: "xoxb-..."
```

**HTTP Mode Config:**
```yaml
channels:
  slack:
    enabled: true
    mode: "http"
    botToken: "xoxb-..."
    signingSecret: "your-signing-secret"
    webhookPath: "/slack/events"
```

### 4.2 Required Slack App Permissions

Permissions yang dibutuhkan untuk Slack integration:

```
app_mentions:read
channels:history
channels:read
chat:write
groups:history
im:history
mpim:history
reactions:read
```

### 4.3 Slack Notification Example

Contoh Slack webhook notification dari custom skill:

```python
import requests

SLACK_WEBHOOK = os.environ.get('SLACK_WEBHOOK')

def notify_slack(message):
    """Send notification to Slack."""
    requests.post(SLACK_WEBHOOK, json={"text": message})
```

---

## 5. MCP (Model Context Protocol) Architecture

### 5.1 How OpenClaw Uses MCP

Setiap skill di ClawHub (marketplace OpenClaw) adalah MCP server. Ketika kamu enable skill, OpenClaw connect ke MCP server tersebut dan membuat tools-nya available untuk AI agent. Setiap skill expose satu atau lebih tools yang bisa dipanggil agent selama conversations.

### 5.2 MCP Configuration

MCP servers bisa run locally (sebagai subprocess di machine) atau remotely (sebagai hosted service). Contoh connecting ke filesystem MCP server:

```yaml
# In ~/.openclaw/config.yaml
mcp:
  servers:
    filesystem:
      command: "npx"
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/Users/you/Documents"]
      description: "Access to Documents folder"
```

### 5.3 Custom MCP Server

Karena MCP adalah open standard, developers bisa build custom MCP servers yang OpenClaw bisa gunakan. Jika butuh tool yang tidak ada di ClawHub—misalnya connecting ke internal API perusahaan—kamu bisa build MCP server sendiri dan plug ke OpenClaw.

---

## 6. Security Considerations

### 6.1 Skill Security

Audit Bitdefender menemukan lebih dari 135.000 instances terbuka di internet karena tidak ada yang mengubah default setting. OpenClaw bind ke 0.0.0.0 by default, artinya instance listen di setiap network interface. Sekitar 17% skills yang listed diidentifikasi sebagai malicious.

**Recommendations:**
- Change bind address ke 127.0.0.1
- Review skill source code sebelum install
- Use VirusTotal reports di ClawHub

### 6.2 ClawSec Security Suite

ClawSec adalah OpenClaw professional security skill suite yang menyediakan: Drift Detection (monitor AI behavioral deviations, detect prompt injection dan jailbreak attempts), Security Auditing (automatically audit installed skills untuk permissions dan security), Integrity Verification (verify skill files belum di-tamper), Security Recommendations, dan Log Monitoring.

```yaml
skills:
  clawsec:
    auditInterval: 3600  # seconds
    driftDetection: true
    integrityCheck: true
    alertChannel: "slack"
```

### 6.3 SKILL.md Security Rules

Security rules yang mandatory untuk skill execution:

- NEVER read, cat, print, display, or log .env file (contains API secrets)
- NEVER pass credentials as command-line arguments
- NEVER include credentials in output, responses, logs, or error messages
- NEVER transmit credentials to external URL/API except local MCP endpoint
- NEVER run variable-expanded commands

---

## 7. Workflow Automation

### 7.1 Lobster Workflow Shell

Lobster adalah OpenClaw-native workflow shell: typed, local-first "macro engine" yang turns skills/tools menjadi composable pipelines dan safe automations—dan lets OpenClaw call workflows tersebut dalam satu step.

### 7.2 Webhook Integration

OpenClaw mendukung webhook creation untuk berbagai sources:

```bash
# GitHub webhook
openclaw webhook create --source github --events "push,pull_request,deployment"

# CI/CD webhook
openclaw webhook create --source cicd --events "build,deploy"
```

---

## 8. Monitoring & Logging

### 8.1 Built-in Logging

OpenClaw writes application logs by default. CLI untuk view dan follow logs:

```bash
# View last 50 lines
openclaw logs --tail 50

# Follow logs in real time
openclaw logs --follow

# Service status
openclaw status

# Run health/diagnostic checks
openclaw doctor

# Security audit of installed skills
openclaw security audit

# Validate configuration
openclaw config validate
```

### 8.2 External Log Shipping

Untuk enterprise deployments, feed logs ke SIEM atau security analytics platform. Options include:

- Fluentd
- Filebeat
- Grafana Loki + Alertmanager
- Elasticsearch
- Datadog

---

## 9. Relevant Skills Summary

| Category | Skill Name | Purpose |
|----------|------------|---------|
| **GitLab** | glab-cli | Interact with GitLab via CLI |
| **GitLab** | Composio GitLab MCP | Full GitLab API access |
| **Git** | git-summary | Repository status overview |
| **Git** | git-workflows | Advanced git operations |
| **GitHub** | github-mcp | GitHub repository management |
| **Code** | qa-audit | Static code analysis |
| **Logs** | Elasticsearch skill | Log search & analysis |
| **Security** | clawsec | Security auditing |
| **Slack** | Native channel | Built-in Slack integration |

---

## 10. Implementation Notes for Bluebird

### 10.1 Adapting for Internal Use

Karena Bluebird menggunakan GitLab (bukan GitHub), fokus pada:
1. **Composio GitLab MCP** atau **glab-cli** untuk issue management
2. **Custom Elasticsearch skill** mirip tutorial Elastic Labs untuk query log MRG/UPG
3. **Native Slack channel** untuk notifications

### 10.2 mcpmrg Reuse Strategy

Existing `mcpmrg` tool bisa di-wrap sebagai OpenClaw skill:

```yaml
# Option A: External command
elk-log-analyzer:
  type: external
  command: mcpmrg query
  args:
    - --trace-id={{trace_id}}
    - --format=json

# Option B: Convert to MCP server
# Build mcpmrg sebagai MCP server dengan Go SDK
```

### 10.3 Recommended Skill Stack

| Component | Recommended Approach |
|-----------|---------------------|
| GitLab Issues | Composio GitLab MCP + custom webhook listener |
| Log Analysis | Custom skill wrapping mcpmrg |
| Code Analysis | git-summary + custom Go static analysis |
| Notifications | Native Slack channel |
| Security | ClawSec for skill auditing |

---

## References

- [Composio GitLab Integration](https://composio.dev/toolkits/gitlab/framework/openclaw)
- [Elastic Labs OpenClaw Tutorial](https://www.elastic.co/search-labs/blog/openclaw-elasticsearch-ai-agents)
- [OpenClaw Slack Docs](https://docs.openclaw.ai/channels/slack)
- [VoltAgent Awesome OpenClaw Skills](https://github.com/VoltAgent/awesome-openclaw-skills)
- [OpenClaw MCP Guide](https://openclawlaunch.com/guides/openclaw-mcp)
