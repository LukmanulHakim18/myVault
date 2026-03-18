---
title: 'OpenClaw Skill: GitLab Issue'
type: skill-spec
version: 1.0.0
status: draft
tags:
  - openclaw
  - skill
  - gitlab
---
# OpenClaw Skills: GitLab Issue Skill

## Metadata

| Field | Value |
|-------|-------|
| Skill Name | `gitlab-issue` |
| Version | 1.0.0 |
| Author | MRG Architecture Team |
| Dependencies | GitLab API v4 |

---

## Description

Skill untuk monitor dan interact dengan GitLab issues. Digunakan sebagai trigger utama untuk incident handler workflow.

---

## Capabilities

1. **Watch Issues** - Monitor new issues dengan filter labels
2. **Parse Issue** - Extract structured data dari issue description
3. **Create Issue** - Buat issue baru (bugfix ticket, discussion)
4. **Add Comment** - Post comment ke existing issue
5. **Update Issue** - Update labels, assignee, status

---

## Required Permissions

```yaml
permissions:
  - gitlab:read_issues
  - gitlab:write_issues
  - gitlab:read_repository
  - gitlab:read_user  # untuk resolve @mentions
```

---

## Configuration

### Environment Variables

```bash
# Required
GITLAB_URL=https://gitlab.bluebird.id
GITLAB_TOKEN=glpat-xxxxxxxxxxxx

# Optional
GITLAB_WEBHOOK_SECRET=your-webhook-secret
```

### config.yaml

```yaml
name: gitlab-issue
version: 1.0.0

environment:
  GITLAB_URL: ${GITLAB_URL}
  GITLAB_TOKEN: ${GITLAB_TOKEN}

triggers:
  - type: webhook
    event: issue.created
    filter:
      labels:
        - production-incident
      projects:
        - mrg/*
        - upg/*

projects:
  mrg:
    order-orchestrator:
      id: 123
      path: meta-reservation-gateway/order-orchestrator
    session-manager:
      id: 124
      path: meta-reservation-gateway/session-manager
    auth-service:
      id: 125
      path: meta-reservation-gateway/auth-service
    notification-center:
      id: 126
      path: meta-reservation-gateway/notification-center
      
  upg:
    payment-service:
      id: 200
      path: universal-payment-gateway/payment-service
    wallet-service:
      id: 201
      path: universal-payment-gateway/wallet-service

templates:
  bugfix:
    labels: [bug, incident-related]
    milestone: current-sprint
  discussion:
    labels: [discussion, needs-decision]
```

---

## Usage Examples

### 1. Watch for Production Incidents

**Prompt:**
```
Watch GitLab issues with label "production-incident" in MRG and UPG projects.
When new issue detected, notify me with issue details.
```

**Skill Action:**
```yaml
action: watch
filter:
  labels: ["production-incident"]
  projects: ["mrg/*", "upg/*"]
callback:
  type: workflow
  trigger: incident-handler
```

### 2. Parse Issue Content

**Prompt:**
```
Parse issue #1234 and extract order_id, trace_id, and service name.
```

**Skill Action:**
```yaml
action: parse
issue_id: 1234
extract:
  - order_id
  - trace_id
  - service
  - timestamp
```

**Expected Output:**
```json
{
  "order_id": "ORD-ABC123",
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "service": "order-orchestrator",
  "timestamp": "2026-03-18T10:15:30+07:00",
  "raw_description": "..."
}
```

### 3. Create Bugfix Ticket

**Prompt:**
```
Create bugfix ticket in order-orchestrator repo for payment timeout issue.
Title: "[BUGFIX] Payment timeout not handled correctly"
Assign to current on-call engineer.
```

**Skill Action:**
```yaml
action: create_issue
project: mrg/order-orchestrator
data:
  title: "[BUGFIX] Payment timeout not handled correctly"
  description: |
    ## Related Incident
    - Original Issue: #1234
    
    ## Problem
    Payment timeout errors are not being handled, causing cascade failures.
    
    ## Suggested Fix
    Add retry logic with exponential backoff.
    
  labels:
    - bug
    - incident-related
    - priority-high
  assignee: "@oncall"
```

### 4. Add Analysis Report Comment

**Prompt:**
```
Post incident analysis report as comment on issue #1234.
```

**Skill Action:**
```yaml
action: add_comment
issue_id: 1234
comment: |
  ## 🔍 Incident Analysis Report
  
  **Generated:** 2026-03-18 10:30:00 WIB
  
  ### Summary
  ...
  
  ### Root Cause
  ...
```

---

## Parsing Rules

### Order ID Detection

```yaml
patterns:
  - regex: "order[_-]?id[:\\s]+([A-Z0-9-]+)"
    flags: i
  - regex: "booking[_-]?id[:\\s]+([A-Z0-9-]+)"
    flags: i
  - regex: "\\b(ORD-[A-Z0-9]+)\\b"
  - regex: "\\b(BKG-[A-Z0-9]+)\\b"

examples:
  - input: "Order ID: ORD-ABC123"
    output: "ORD-ABC123"
  - input: "booking_id: BKG-XYZ789"
    output: "BKG-XYZ789"
```

### Trace ID Detection

```yaml
patterns:
  - regex: "trace[_-]?id[:\\s]+([a-f0-9-]{36})"
    flags: i
  - regex: "x-trace-id[:\\s]+([a-f0-9-]{36})"
    flags: i
  - regex: "correlation[_-]?id[:\\s]+([a-f0-9-]{36})"
    flags: i

examples:
  - input: "trace_id: a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    output: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
```

### Service Detection

```yaml
patterns:
  - regex: "service[:\\s]+([a-z][a-z0-9-]+)"
    flags: i
  - regex: "affected[:\\s]+([a-z][a-z0-9-]+)"
    flags: i

keyword_mapping:
  order-orchestrator:
    - order
    - booking
    - orchestrator
    - create order
  session-manager:
    - session
    - fleet
    - driver
    - assign
  payment-service:
    - payment
    - charge
    - refund
    - wallet
  auth-service:
    - auth
    - login
    - token
    - otp
```

---

## Error Handling

```yaml
errors:
  GITLAB_UNAUTHORIZED:
    code: 401
    message: "Invalid or expired GitLab token"
    action: notify_admin
    
  GITLAB_NOT_FOUND:
    code: 404
    message: "Issue or project not found"
    action: log_and_skip
    
  GITLAB_RATE_LIMITED:
    code: 429
    message: "GitLab API rate limit exceeded"
    action: wait_and_retry
    retry_after: 60
    
  PARSE_FAILED:
    message: "Could not extract required fields from issue"
    action: escalate_to_human
```

---

## Security Notes

1. **Token Scope:** Gunakan token dengan minimum required scope
2. **Webhook Validation:** Selalu validate webhook signature
3. **Input Sanitization:** Strip potential injection patterns dari issue content
4. **Audit Log:** Log semua actions untuk audit trail

---

## Related Files

- [[RFC-001-openclaw-incident-handler|Main RFC]]
- [[SKILL-elk-log-analyzer|ELK Log Analyzer Skill]]
- [[SKILL-code-analyzer|Code Analyzer Skill]]
