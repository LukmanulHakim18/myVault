---
title: "Go Programming - Map of Content"
tags: [moc, go, programming]
aliases: [golang-moc, go-moc]
---

# 🐹 Go Programming - Map of Content

> Central hub untuk semua knowledge Go programming Anda

---

## 📚 Core Concepts

### Fundamentals
- [[go-basics]] - Basic syntax, types, variables
- [[go-functions]] - Functions, methods, receivers
- [[go-structs]] - Structs and composition
- [[go-interfaces]] - Interfaces and polymorphism
- [[go-error-handling]] - Error handling patterns
- [[go-context]] - Context usage and patterns

### Advanced Topics
- [[go-concurrency]] - Goroutines and channels
- [[go-reflection]] - Reflection and type assertions
- [[go-generics]] - Generic programming (Go 1.18+)
- [[go-memory-management]] - Memory allocation and GC
- [[go-assembly]] - Understanding Go assembly

---

## 🎨 Design Patterns in Go

### Creational Patterns
- [[singleton-pattern-go]] - Singleton implementation
- [[factory-pattern-go]] - Factory and Abstract Factory
- [[builder-pattern-go]] - Builder pattern
- [[prototype-pattern-go]] - Prototype cloning
- [[object-pool-pattern-go]] - Object pool pattern

### Structural Patterns
- [[adapter-pattern-go]] - Adapter pattern
- [[decorator-pattern-go]] - Decorator pattern
- [[facade-pattern-go]] - Facade pattern
- [[proxy-pattern-go]] - Proxy pattern
- [[composite-pattern-go]] - Composite pattern

### Behavioral Patterns
- [[strategy-pattern-go]] - Strategy pattern
- [[observer-pattern-go]] - Observer pattern
- [[command-pattern-go]] - Command pattern
- [[chain-of-responsibility-go]] - Chain of Responsibility
- [[template-method-go]] - Template Method

### Go-Specific Patterns
- [[functional-options-go]] - Functional options pattern
- [[worker-pool-go]] - Worker pool pattern
- [[circuit-breaker-go]] - Circuit breaker
- [[retry-pattern-go]] - Retry with backoff

---

## ✅ Best Practices

### Code Quality
- [[effective-go-notes]] - Notes from Effective Go
- [[go-proverbs]] - Go proverbs explained
- [[go-code-review-comments]] - Common review comments
- [[common-mistakes-go]] - Common mistakes to avoid
- [[go-idioms]] - Idiomatic Go patterns
- [[go-naming-conventions]] - Naming best practices

### Performance
- [[go-performance-tips]] - Performance optimization
- [[profiling-go]] - Profiling techniques
- [[memory-optimization-go]] - Memory optimization
- [[cpu-optimization-go]] - CPU optimization
- [[benchmarking-go]] - Writing benchmarks

### Security
- [[go-security-best-practices]] - Security guidelines
- [[input-validation-go]] - Input validation
- [[sql-injection-prevention-go]] - SQL injection prevention
- [[crypto-in-go]] - Cryptography in Go

---

## 🧪 Testing

### Testing Fundamentals
- [[unit-testing-go]] - Unit testing basics
- [[table-driven-tests]] - Table-driven test pattern
- [[test-coverage-go]] - Code coverage analysis
- [[testing-best-practices-go]] - Testing best practices

### Advanced Testing
- [[mocking-in-go]] - Mocking strategies
  - [[gomock-guide]] - Using gomock
  - [[testify-guide]] - Using testify/mock
- [[integration-testing-go]] - Integration tests
- [[e2e-testing-go]] - End-to-end tests
- [[fuzz-testing-go]] - Fuzz testing (Go 1.18+)
- [[benchmarking-go]] - Writing benchmarks

---

## 🏗️ Project Structure

### Organization
- [[go-project-layout]] - Standard project layout
- [[package-design-go]] - Package design principles
- [[dependency-management-go]] - Managing dependencies
- [[go-modules]] - Go modules guide

### Architecture
- [[clean-architecture-go]] - Clean architecture
- [[hexagonal-architecture-go]] - Ports & adapters
- [[ddd-in-go]] - Domain-Driven Design
- [[microservices-go]] - Microservices patterns

---

## 🌐 Web Development

### HTTP & APIs
- [[http-server-go]] - HTTP server patterns
- [[rest-api-go]] - RESTful API design
- [[graphql-go]] - GraphQL implementation
- [[grpc-go]] - gRPC services
- [[websockets-go]] - WebSocket implementation

### Frameworks
- [[gin-framework-guide]] - Gin web framework
- [[echo-framework-guide]] - Echo framework
- [[fiber-framework-guide]] - Fiber framework
- [[chi-router-guide]] - Chi router

### Middleware
- [[middleware-patterns-go]] - Middleware patterns
- [[authentication-middleware-go]] - Auth middleware
- [[logging-middleware-go]] - Logging middleware
- [[cors-middleware-go]] - CORS handling

---

## 💾 Database

### SQL Databases
- [[database-sql-go]] - database/sql package
- [[sqlx-guide]] - sqlx library
- [[gorm-guide]] - GORM ORM
- [[database-migrations-go]] - Migration strategies
- [[connection-pooling-go]] - Connection pooling
- [[transaction-handling-go]] - Transaction patterns

### NoSQL
- [[mongodb-go]] - MongoDB driver
- [[redis-go]] - Redis client usage
- [[elasticsearch-go]] - Elasticsearch client

### Query Patterns
- [[orm-vs-raw-sql]] - ORM vs raw SQL
- [[query-builder-patterns]] - Query builders
- [[n-plus-one-prevention]] - N+1 query prevention

---

## ⚡ Concurrency

### Goroutines & Channels
- [[goroutines-basics]] - Goroutine fundamentals
- [[channels-patterns]] - Channel patterns
- [[select-statement]] - Select statement usage
- [[buffered-vs-unbuffered]] - Channel types
- [[channel-direction]] - Channel directions

### Synchronization
- [[sync-package]] - sync package primitives
- [[mutex-patterns]] - Mutex usage
- [[rwmutex-patterns]] - RWMutex patterns
- [[waitgroup-patterns]] - WaitGroup usage
- [[once-patterns]] - sync.Once patterns

### Advanced Concurrency
- [[worker-pools]] - Worker pool implementation
- [[pipeline-pattern]] - Pipeline pattern
- [[fan-out-fan-in]] - Fan-out/fan-in
- [[context-cancellation]] - Context cancellation
- [[goroutine-leaks]] - Preventing leaks

---

## 📦 Popular Packages

### Standard Library
- `net/http` - [[http-package-guide]]
- `database/sql` - [[database-sql-guide]]
- `context` - [[context-package-guide]]
- `encoding/json` - [[json-encoding-guide]]
- `sync` - [[sync-package-guide]]
- `testing` - [[testing-package-guide]]
- `time` - [[time-package-guide]]
- `io` - [[io-package-guide]]

### Third-Party Libraries
- `gin-gonic/gin` - [[gin-framework-guide]]
- `go-redis/redis` - [[redis-client-guide]]
- `stretchr/testify` - [[testify-usage-guide]]
- `uber-go/zap` - [[zap-logging-guide]]
- `spf13/viper` - [[viper-config-guide]]
- `spf13/cobra` - [[cobra-cli-guide]]
- `gorilla/mux` - [[gorilla-mux-guide]]
- `golang-jwt/jwt` - [[jwt-guide]]

---

## 🛠️ Tools & Utilities

### Development Tools
- [[go-toolchain]] - Go toolchain overview
- [[golangci-lint]] - Linting with golangci-lint
- [[gofmt-goimports]] - Code formatting
- [[go-vet]] - Static analysis with vet
- [[delve-debugger]] - Debugging with Delve

### Profiling & Debugging
- [[pprof-guide]] - pprof profiling
- [[trace-tool]] - Execution tracer
- [[memory-profiling]] - Memory profiling
- [[cpu-profiling]] - CPU profiling
- [[debugging-techniques]] - Debugging strategies

---

## 🚀 Microservices

### Patterns
- [[service-discovery-go]] - Service discovery
- [[circuit-breaker-go]] - Circuit breaker pattern
- [[retry-backoff-go]] - Retry with backoff
- [[saga-pattern-go]] - Saga pattern
- [[event-sourcing-go]] - Event sourcing
- [[cqrs-go]] - CQRS pattern

### Communication
- [[rest-microservices]] - REST communication
- [[grpc-microservices]] - gRPC services
- [[message-queues-go]] - Message queue integration
- [[kafka-go]] - Kafka integration
- [[rabbitmq-go]] - RabbitMQ usage

### Observability
- [[logging-microservices]] - Structured logging
- [[metrics-prometheus]] - Prometheus metrics
- [[distributed-tracing]] - OpenTelemetry tracing
- [[health-checks-go]] - Health check endpoints

---

## 📊 Real-World Examples

### From My Projects
- [[id-generator-implementation]] - ID generator system
- [[notification-system-go]] - Notification architecture
- [[payment-processing-flow]] - Payment processing
- [[booking-system-design]] - Booking system
- [[email-strategy-pattern]] - Email strategy implementation

### Code Patterns
- [[error-handling-examples]] - Error handling patterns
- [[middleware-examples]] - Middleware implementations
- [[testing-examples]] - Test examples
- [[concurrency-examples]] - Concurrency patterns

---

## 📝 Code Snippets

### Common Patterns
- [[error-wrapping-snippet]] - Error wrapping
- [[graceful-shutdown-snippet]] - Graceful shutdown
- [[rate-limiting-snippet]] - Rate limiting
- [[pagination-snippet]] - Pagination
- [[jwt-auth-snippet]] - JWT authentication
- [[file-upload-snippet]] - File upload handling

### Utilities
- [[retry-mechanism]] - Retry utility
- [[timeout-context]] - Timeout handling
- [[custom-errors]] - Custom error types
- [[logger-setup]] - Logger initialization

---

## 🎓 Learning Resources

### Official Resources
- [Go Official Documentation](https://golang.org/doc/)
- [Effective Go](https://golang.org/doc/effective_go)
- [Go Blog](https://blog.golang.org/)
- [Go Playground](https://play.golang.org/)

### Books (My Notes)
- [[the-go-programming-language]] - Donovan & Kernighan
- [[concurrency-in-go]] - Katherine Cox-Buday
- [[cloud-native-go]] - Building Go microservices
- [[network-programming-go]] - Network programming

### Courses
- [[go-fundamentals-course]] - Course notes
- [[advanced-go-patterns]] - Advanced patterns
- [[microservices-go-course]] - Microservices course

### Blogs & Articles
- [[go-articles-collection]] - Curated articles
- [[go-weekly-newsletter]] - Newsletter notes

---

## 🔗 Related MOCs

- [[architecture-moc]] - Architecture patterns
- [[microservices-moc]] - Microservices
- [[testing-moc]] - Testing strategies
- [[devops-moc]] - DevOps practices

---

## 📈 Progress Tracker

### Mastery Levels
- ✅ **Beginner**: Basics, syntax, simple programs
- ✅ **Intermediate**: Concurrency, testing, packages
- 🔄 **Advanced**: Design patterns, optimization, architecture
- ⏳ **Expert**: Contributing to Go, deep internals

### Current Focus
- [ ] Master context patterns
- [ ] Deep dive into generics
- [ ] Advanced concurrency patterns
- [ ] Performance optimization

---

## 💡 Quick Tips

### Daily Reminders
1. **Error handling**: Always wrap errors with context
2. **Testing**: Aim for >80% coverage
3. **Interfaces**: Keep them small and focused
4. **Concurrency**: Don't leak goroutines
5. **Context**: Propagate context through call chains

### Performance Tips
- Avoid premature optimization
- Profile before optimizing
- Use sync.Pool for frequently allocated objects
- Minimize allocations in hot paths
- Use buffered channels appropriately

---

## 🏷️ Tags for Quick Search

#go #golang #programming #backend #microservices #concurrency #testing #architecture #patterns #best-practices

---

*Last updated: {{date}}*

[[dashboard]] | [[02-Work/Architecture/]] | [[03-Learning/]]