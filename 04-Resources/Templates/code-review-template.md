---
title: "Code Review: [Project/PR Name]"
date: 
reviewer: Lukmanul Hakim
reviewee: 
pr_link: 
repository: 
tags: [code-review, go]
---

# Code Review: [Project/PR Name]

## Overview
- **Repository**: 
- **Branch**: feature/xxx → main/develop
- **PR Number**: #XXX
- **PR Link**: 
- **Description**: [Brief description of what this PR does]
- **Lines Changed**: +XXX -XXX
- **Review Date**: 
- **Estimated Review Time**: XX minutes

---

## Summary
[Ringkasan singkat tentang perubahan dan kesan umum]

**Overall Impression**: ✅ Excellent | 👍 Good | ⚠️ Needs Improvement | ❌ Major Issues

---

## Review Checklist

### ✅ Architecture & Design
- [ ] Follows microservice architecture principles
- [ ] Proper separation of concerns (handler → usecase → repository)
- [ ] Appropriate use of design patterns
- [ ] No tight coupling between components
- [ ] Interfaces used appropriately
- [ ] Dependency injection implemented correctly
- [ ] Scalability considerations addressed
- [ ] Performance implications considered

**Notes**: 

---

### ✅ Go Best Practices
- [ ] Proper error handling (errors wrapped with context)
- [ ] No panic in production code (except main.go init)
- [ ] Proper use of context.Context throughout
- [ ] No goroutine leaks (proper cleanup)
- [ ] Channel usage correct (proper closure, buffering)
- [ ] defer placement correct (especially in loops)
- [ ] Interfaces small and focused
- [ ] No naked returns in long functions
- [ ] Proper nil checks
- [ ] Exported types/functions properly documented

**Notes**:

---

### ✅ Code Quality & Maintainability
- [ ] Clear and descriptive naming (variables, functions, types)
- [ ] No code duplication (DRY principle)
- [ ] Single Responsibility Principle followed
- [ ] Functions are small and focused (< 50 lines ideally)
- [ ] Cyclomatic complexity reasonable
- [ ] No magic numbers (use constants)
- [ ] Comments on exported functions (godoc format)
- [ ] Complex logic has explanatory comments
- [ ] TODO/FIXME properly tracked in issue tracker

**Notes**:

---

### ✅ Testing
- [ ] Unit tests present and comprehensive
- [ ] Test coverage > 80% for new code
- [ ] Table-driven tests used where appropriate
- [ ] Dependencies properly mocked
- [ ] Integration tests for critical paths
- [ ] Error scenarios tested
- [ ] Edge cases covered
- [ ] Test names descriptive (TestFunctionName_Scenario_ExpectedResult)
- [ ] No test code duplication (use test helpers)
- [ ] Tests are deterministic (no flaky tests)

**Coverage**: X%

**Notes**:

---

### ✅ Security
- [ ] No hardcoded credentials or secrets
- [ ] Sensitive data properly handled
- [ ] Input validation comprehensive
- [ ] SQL injection prevention (parameterized queries)
- [ ] Authentication checks in place
- [ ] Authorization checks in place
- [ ] No XSS vulnerabilities
- [ ] CSRF protection where needed
- [ ] Proper encryption for sensitive data
- [ ] Secure random number generation

**Notes**:

---

### ✅ Performance
- [ ] No N+1 query problems
- [ ] Database queries optimized
- [ ] Proper indexing strategy
- [ ] Memory leak prevention
- [ ] Connection pool management appropriate
- [ ] Caching strategy appropriate
- [ ] No unnecessary allocations
- [ ] Efficient algorithms used
- [ ] Batch operations where applicable
- [ ] No blocking operations in hot paths

**Notes**:

---

### ✅ Database
- [ ] Migrations properly structured
- [ ] Schema changes backward compatible
- [ ] Indexes added where needed
- [ ] Foreign key constraints appropriate
- [ ] Transactions used correctly
- [ ] No long-running transactions
- [ ] Proper rollback handling

**Notes**:

---

### ✅ API Design (if applicable)
- [ ] RESTful principles followed
- [ ] Proper HTTP methods used
- [ ] Status codes appropriate
- [ ] Request/response format consistent
- [ ] API versioned properly
- [ ] Pagination implemented for lists
- [ ] Rate limiting considered
- [ ] Documentation updated (Swagger/OpenAPI)

**Notes**:

---
### ✅ Configuration & Environment
- [ ] No environment-specific code
- [ ] Configuration externalized
- [ ] Environment variables properly used
- [ ] Secrets managed securely (not in code)
- [ ] Default values sensible

**Notes**:

---

### ✅ Logging & Monitoring
- [ ] Appropriate log levels used
- [ ] Structured logging implemented
- [ ] No sensitive data in logs
- [ ] Metrics/instrumentation added
- [ ] Tracing context propagated

**Notes**:

---

## Issues Found

### 🔴 Critical Issues (Must Fix Before Merge)
1. **Issue**: [Description]
   - **File**: `path/to/file.go:123`
   - **Impact**: [Why this is critical]
   - **Suggestion**: 
   ```go
   // Bad
   // current code
   
   // Good
   // suggested code
   ```

### 🟡 Major Issues (Should Fix Before Merge)
1. **Issue**: [Description]
   - **File**: `path/to/file.go:456`
   - **Impact**: [Why this matters]
   - **Suggestion**: 

### 🟢 Minor Issues (Nice to Have)
1. **Issue**: [Description]
   - **File**: `path/to/file.go:789`
   - **Suggestion**: 

### 💡 Suggestions (Optional Improvements)
1. **Suggestion**: [Description]
   - **Rationale**: [Why this would be better]

---

## Positive Points ⭐
[Highlight good practices, clever solutions, well-written code]

1. ✅ 
2. ✅ 
3. ✅ 

---

## Refactoring Opportunities
[Long-term improvements that could be done]

1. **Opportunity**: 
   - **Benefit**: 
   - **Effort**: Low | Medium | High
   - **Priority**: Low | Medium | High

---

## Follow-up Actions
- [ ] [Action item] - @assignee - Due: [date]
- [ ] [Action item] - @assignee - Due: [date]

---

## Overall Assessment

**Code Quality**: ⭐⭐⭐⭐⭐ (1-5 stars)
**Test Coverage**: ⭐⭐⭐⭐☆
**Documentation**: ⭐⭐⭐⭐⭐
**Performance**: ⭐⭐⭐⭐☆

**Decision**: 
- ✅ **Approved** - Ready to merge
- 👍 **Approved with Comments** - Minor issues, can merge after addressing
- ⏸️ **Changes Requested** - Need to address issues before re-review
- ❌ **Rejected** - Major issues, requires significant rework

**Reasoning**: 

---

## Review Notes & Discussion
[Additional notes, questions for the author, discussion points]

**Question 1**: 
**Answer**: 

---

## Learning Points
[Things learned from this review that could benefit the team]

1. 
2. 

---

## Related Reviews
- [[code-review-2025-01-01]] - Related PR
- [[rfc-xxx]] - Related RFC

---

**Review completed**: [Date]
**Time spent**: XX minutes
**Next review date** (if changes requested): [Date]