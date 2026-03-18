---
title: 'OpenClaw Skill: Code Analyzer'
type: skill-spec
version: 1.0.0
status: draft
tags:
  - openclaw
  - skill
  - code-analysis
  - git
---
# OpenClaw Skills: Code Analyzer Skill

## Metadata

| Field | Value |
|-------|-------|
| Skill Name | `code-analyzer` |
| Version | 1.0.0 |
| Author | MRG Architecture Team |
| Dependencies | Git, Go tools |

---

## Description

Skill untuk checkout source code dari Git repository dan perform static analysis untuk trace error path berdasarkan log information.

---

## Capabilities

1. **Clone Repository** - Clone repo dari GitLab (read-only)
2. **Checkout Version** - Checkout specific tag/commit (production version)
3. **Find Function** - Locate function definition by name
4. **Trace Call Stack** - Follow call chain dari error location
5. **Analyze Error Path** - Identify potential issues dalam error handling

---

## Required Permissions

```yaml
permissions:
  - git:clone
  - git:checkout
  - filesystem:read
  - filesystem:write  # untuk work directory
```

⚠️ **NO push/write access ke repository**

---

## Configuration

### Environment Variables

```bash
# Required
GIT_BASE_URL=git@gitlab.bluebird.id
GIT_TOKEN=glpat-xxxxxxxxxxxx

# Optional
WORK_DIR=/tmp/openclaw-code-analysis
DEPLOY_API=https://deploy.bluebird.id/api
```

### config.yaml

```yaml
name: code-analyzer
version: 1.0.0

environment:
  GIT_BASE_URL: ${GIT_BASE_URL}
  GIT_TOKEN: ${GIT_TOKEN}
  WORK_DIR: ${WORK_DIR:-/tmp/openclaw-code-analysis}

repositories:
  mrg:
    order-orchestrator:
      url: git@gitlab.bluebird.id:mrg/order-orchestrator.git
      language: go
      main_package: ./cmd/server
      
    session-manager:
      url: git@gitlab.bluebird.id:mrg/session-manager.git
      language: go
      main_package: ./cmd/server
      
    auth-service:
      url: git@gitlab.bluebird.id:mrg/auth-service.git
      language: go
      main_package: ./cmd/server
      
    notification-center:
      url: git@gitlab.bluebird.id:mrg/notification-center.git
      language: go
      main_package: ./cmd/server
      
  upg:
    payment-service:
      url: git@gitlab.bluebird.id:upg/payment-service.git
      language: go
      main_package: ./cmd/server
      
    wallet-service:
      url: git@gitlab.bluebird.id:upg/wallet-service.git
      language: go
      main_package: ./cmd/server

analysis_settings:
  context_lines: 50
  max_call_depth: 5
  include_tests: false
  
production_tags:
  # How to determine production version
  method: deploy_api  # or "latest_tag"
  api_endpoint: ${DEPLOY_API}/services/{service}/version
```

---

## Usage Examples

### 1. Checkout Production Version

**Prompt:**
```
Checkout order-orchestrator code at current production version.
```

**Skill Action:**
```yaml
action: checkout
service: order-orchestrator
version: production  # resolves to current prod tag
```

**Behind the scenes:**
```bash
# 1. Get production version from deploy API
PROD_VERSION=$(curl -s $DEPLOY_API/services/order-orchestrator/version)
# Returns: v2024.03.15-1

# 2. Clone repository (if not exists)
git clone --depth 1 $GIT_BASE_URL/mrg/order-orchestrator.git \
    $WORK_DIR/order-orchestrator

# 3. Fetch specific tag
cd $WORK_DIR/order-orchestrator
git fetch origin tag $PROD_VERSION --no-tags

# 4. Checkout
git checkout $PROD_VERSION
```

**Output:**
```yaml
checkout_result:
  service: order-orchestrator
  version: v2024.03.15-1
  local_path: /tmp/openclaw-code-analysis/order-orchestrator
  commit: abc123def456
  timestamp: "2024-03-15T10:30:00Z"
```

### 2. Find Function Definition

**Prompt:**
```
Find the ProcessPayment function in payment-service.
```

**Skill Action:**
```yaml
action: find_function
service: payment-service
function: ProcessPayment
```

**Output:**
```yaml
function_location:
  file: internal/handler/payment.go
  line: 220
  package: handler
  
  signature: |
    func (h *PaymentHandler) ProcessPayment(
        ctx context.Context,
        req *PaymentRequest,
    ) (*PaymentResponse, error)
  
  doc_comment: |
    // ProcessPayment handles payment processing for an order.
    // It validates the request, checks balance, and executes the payment.
```

### 3. Trace Error from Stack

**Prompt:**
```
Given this stack trace from logs, find the code path:

payment-service/internal/handler/payment.go:234
payment-service/internal/service/balance.go:89
payment-service/internal/client/wallet.go:156
```

**Skill Action:**
```yaml
action: trace_error
service: payment-service
stack_trace:
  - file: internal/handler/payment.go
    line: 234
  - file: internal/service/balance.go
    line: 89
  - file: internal/client/wallet.go
    line: 156
context_lines: 20
```

**Output:**
```yaml
trace_analysis:
  - location:
      file: internal/handler/payment.go
      line: 234
      function: ProcessPayment
    code_snippet: |
      func (h *PaymentHandler) ProcessPayment(ctx context.Context, req *PaymentRequest) (*PaymentResponse, error) {
          // ... lines 220-233 ...
          
          // LINE 234 - Error origin
          result, err := h.balanceService.CheckAndReserve(ctx, req.UserID, req.Amount)
          if err != nil {
              return nil, fmt.Errorf("check balance failed: %w", err)
          }
          
          // ... lines 237-250 ...
      }
    analysis:
      issue: "Error from CheckAndReserve is wrapped but no retry logic"
      suggestion: "Add retry for transient errors from balance service"
      
  - location:
      file: internal/service/balance.go
      line: 89
      function: CheckAndReserve
    code_snippet: |
      func (s *BalanceService) CheckAndReserve(ctx context.Context, userID string, amount int64) (*ReserveResult, error) {
          // ... lines 75-88 ...
          
          // LINE 89 - Calls wallet client
          balance, err := s.walletClient.GetBalance(ctx, userID)
          if err != nil {
              return nil, err  // Raw error, no context added
          }
          
          // ... lines 92-100 ...
      }
    analysis:
      issue: "Error from wallet client not wrapped with context"
      suggestion: "Wrap error: fmt.Errorf(\"get balance for user %s: %w\", userID, err)"
      
  - location:
      file: internal/client/wallet.go
      line: 156
      function: GetBalance
    code_snippet: |
      func (c *WalletClient) GetBalance(ctx context.Context, userID string) (*Balance, error) {
          // ... lines 140-155 ...
          
          // LINE 156 - HTTP call to wallet-service
          resp, err := c.httpClient.Get(ctx, fmt.Sprintf("/api/v1/users/%s/balance", userID))
          if err != nil {
              return nil, fmt.Errorf("wallet service request failed: %w", err)
          }
          
          // ... lines 159-170 ...
      }
    analysis:
      issue: "HTTP timeout not explicitly handled"
      suggestion: "Add specific handling for context.DeadlineExceeded"

call_chain_summary:
  depth: 3
  entry_point: "PaymentHandler.ProcessPayment"
  root_cause_location: "WalletClient.GetBalance"
  
  potential_issues:
    - type: missing_retry
      location: payment.go:234
      severity: medium
      
    - type: error_context
      location: balance.go:89
      severity: low
      
    - type: timeout_handling
      location: wallet.go:156
      severity: high
```

### 4. Analyze Error Handling

**Prompt:**
```
Analyze error handling in the ProcessPayment function.
Check if all error paths are properly handled.
```

**Skill Action:**
```yaml
action: analyze_error_handling
service: payment-service
function: ProcessPayment
check:
  - error_wrapping
  - nil_checks
  - timeout_handling
  - retry_logic
  - logging
```

**Output:**
```yaml
error_handling_analysis:
  function: ProcessPayment
  file: internal/handler/payment.go
  
  findings:
    - line: 234
      type: missing_retry
      severity: medium
      current_code: |
        result, err := h.balanceService.CheckAndReserve(ctx, req.UserID, req.Amount)
        if err != nil {
            return nil, fmt.Errorf("check balance failed: %w", err)
        }
      suggested_fix: |
        var result *ReserveResult
        err := retry.Do(func() error {
            var innerErr error
            result, innerErr = h.balanceService.CheckAndReserve(ctx, req.UserID, req.Amount)
            return innerErr
        }, retry.Attempts(3), retry.Delay(100*time.Millisecond))
        if err != nil {
            return nil, fmt.Errorf("check balance failed after retries: %w", err)
        }
      explanation: |
        Balance check can fail due to transient network issues.
        Adding retry logic will improve reliability.
        
    - line: 245
      type: missing_nil_check
      severity: high
      current_code: |
        h.logger.Info("Payment processed", "order_id", result.OrderID)
      suggested_fix: |
        if result != nil {
            h.logger.Info("Payment processed", "order_id", result.OrderID)
        }
      explanation: |
        If previous step fails silently, result could be nil.
        
  summary:
    total_error_paths: 5
    properly_handled: 3
    needs_improvement: 2
    critical_issues: 1
```

---

## Code Parsing Implementation

### Go-specific Parsing

```go
// Using go/ast for accurate parsing
package analyzer

import (
    "go/ast"
    "go/parser"
    "go/token"
)

func FindFunction(filePath, funcName string) (*FunctionInfo, error) {
    fset := token.NewFileSet()
    f, err := parser.ParseFile(fset, filePath, nil, parser.ParseComments)
    if err != nil {
        return nil, err
    }
    
    var result *FunctionInfo
    ast.Inspect(f, func(n ast.Node) bool {
        if fn, ok := n.(*ast.FuncDecl); ok {
            if fn.Name.Name == funcName {
                result = &FunctionInfo{
                    Name:      fn.Name.Name,
                    StartLine: fset.Position(fn.Pos()).Line,
                    EndLine:   fset.Position(fn.End()).Line,
                    // ... more details
                }
                return false
            }
        }
        return true
    })
    
    return result, nil
}
```

### Finding Callers/Callees

```go
func TraceCallChain(rootFunc string, maxDepth int) (*CallChain, error) {
    // Use go/callgraph for call graph analysis
    // Or simpler: grep-based search for function calls
}
```

---

## Sandboxing

Code analysis runs in isolated environment:

```yaml
sandbox:
  enabled: true
  runtime: docker
  image: openclaw-code-analyzer:latest
  
  resources:
    memory: 2Gi
    cpu: 1
    timeout: 120s
    
  mounts:
    - type: bind
      source: ${WORK_DIR}
      target: /workspace
      read_only: false  # need to clone here
      
  network:
    # Only allow git clone
    allowed:
      - gitlab.bluebird.id:22
      - gitlab.bluebird.id:443
    denied:
      - "*"
      
  security:
    no_new_privileges: true
    read_only_root: true
    drop_capabilities:
      - ALL
```

---

## Error Handling

```yaml
errors:
  CLONE_FAILED:
    message: "Failed to clone repository"
    causes:
      - "Invalid credentials"
      - "Repository not found"
      - "Network error"
    action: log_and_notify
    
  VERSION_NOT_FOUND:
    message: "Requested version/tag not found"
    action: try_latest
    fallback:
      use: main
      warn: true
      
  PARSE_ERROR:
    message: "Failed to parse source file"
    action: fallback_to_grep
    
  FUNCTION_NOT_FOUND:
    message: "Function not found in codebase"
    action: expand_search
    fallback:
      search_all_packages: true
```

---

## Output Templates

### Code Context Template

```yaml
code_context:
  file: "{{file_path}}"
  version: "{{git_tag}}"
  
  function:
    name: "{{func_name}}"
    start_line: {{start_line}}
    end_line: {{end_line}}
    
  highlighted_lines:
    - line: {{error_line}}
      content: "{{line_content}}"
      annotation: "ERROR: {{error_description}}"
      
  surrounding_code: |
    {{code_with_line_numbers}}
    
  dependencies:
    imports:
      - "{{import_path}}"
    calls:
      - function: "{{called_func}}"
        file: "{{called_file}}"
```

---

## Security Notes

1. **Read-Only Access:** NO write access ke repositories
2. **Sandboxed Execution:** Code analysis runs di isolated container
3. **No Code Execution:** Only static analysis, tidak execute code
4. **Credential Isolation:** Git credentials tidak exposed ke analysis context
5. **Cleanup:** Work directory cleaned after analysis

---

## Related Files

- [[RFC-001-openclaw-incident-handler|Main RFC]]
- [[SKILL-gitlab-issue|GitLab Issue Skill]]
- [[SKILL-elk-log-analyzer|ELK Log Analyzer Skill]]
- [[SKILL-report-generator|Report Generator Skill]]
