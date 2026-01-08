# ADR-001: Map Service Provider Selection

## Status
**DRAFT** | Proposed | Accepted | Deprecated | Superseded

## Context

Bluebird Group's transportation platform (MRG - Meta Reservation Gateway) requires a reliable map service provider for:
- Geocoding and reverse geocoding
- Route optimization and navigation
- Distance calculations and ETA predictions
- Real-time traffic data
- Multi-modal transportation support (taxi, rental, airport transfer)

### Current State
- [Describe current map provider if any]
- [Current pain points]
- [Limitations]

### Requirements
- Must support Indonesia (Jakarta, Surabaya, Bali, major cities)
- Must handle 10,000+ requests per day
- Response time < 500ms (P95)
- 99.9% uptime SLA
- Cost effective for expected volume

## Decision Drivers

### Critical Factors
1. **Geographic Coverage** - Indonesia coverage quality
2. **Performance** - API response time and reliability
3. **Cost** - Total cost of ownership
4. **Integration** - Ease of implementation with Go backend
5. **Scalability** - Ability to handle growth

### Evaluation Process
- Requirements definition (Week 1)
- Vendor research and shortlisting (Week 2)
- Technical evaluation and benchmarking (Week 3-4)
- Proof of Concept with top 2 candidates (Week 5-6)
- Cost-benefit analysis (Week 7)
- Final decision (Week 8)

## Considered Options

### Option 1: [Vendor Name]
**Score**: /100

**Pros:**
- 
- 
- 

**Cons:**
- 
- 
- 

**Estimated Cost**: $X,XXX/month

### Option 2: [Vendor Name]
**Score**: /100

**Pros:**
- 
- 
-
**Cons:**
- 
- 
- 

**Estimated Cost**: $X,XXX/month

### Option 3: [Vendor Name]
**Score**: /100

**Pros:**
- 
- 
- 

**Cons:**
- 
- 
- 

**Estimated Cost**: $X,XXX/month

## Decision

**Selected Option**: [Vendor Name]

### Rationale
[Explain why this vendor was chosen]

Key deciding factors:
1. 
2. 
3. 

### Trade-offs Accepted
- 
- 
- 

## Consequences

### Positive
- 
- 
- 

### Negative
- 
- 
- 

### Risks & Mitigation
| Risk | Impact | Mitigation |
|------|--------|------------|
| | | |
| | | |

## Implementation Plan

### Phase 1: Preparation (Week 1-2)
- [ ] Finalize contract negotiations
- [ ] Setup accounts and API keys
- [ ] Configure development environment
- [ ] Setup monitoring and alerting

### Phase 2: Development (Week 3-4)
- [ ] Implement Go wrapper library
- [ ] Build abstraction layer
- [ ] Write unit tests
- [ ] Integration testing

### Phase 3: Migration (Week 5-6)
- [ ] Parallel run with existing provider
- [ ] Gradual traffic migration (10% → 50% → 100%)
- [ ] Performance monitoring
- [ ] Cost tracking

### Phase 4: Optimization (Week 7-8)
- [ ] Performance tuning
- [ ] Cost optimization
- [ ] Documentation
- [ ] Team training

## Compliance

### Data Privacy
- [ ] GDPR compliance verified
- [ ] Data residency requirements met
- [ ] Privacy policy reviewed

### Security
- [ ] Security audit completed
- [ ] API key management strategy
- [ ] Network security configured

### Legal
- [ ] Contract terms reviewed
- [ ] SLA terms acceptable
- [ ] Termination clause reviewed

## Monitoring & Review

### Success Metrics
- API response time < 500ms (P95)
- Uptime > 99.9%
- Cost within budget ($X,XXX/month)
- Route accuracy > 95%

### Review Schedule
- **30-day review**: Initial assessment
- **90-day review**: Performance and cost analysis
- **Annual review**: Contract renewal consideration

## References

- [Vendor Comparison Matrix](../analysis/vendor-comparison-matrix.md)
- [Benchmark Results](../benchmarks/results/)
- [PoC Findings](../poc/poc-findings.md)
- [Cost-Benefit Analysis](../analysis/cost-benefit-analysis.md)

---

**Decision Date**: [YYYY-MM-DD]  
**Decided By**: Lukmanul Hakim (Architecture Engineer)  
**Stakeholders**: MRG Team, UPG Team, Platform Engineering  
**Last Updated**: 2025-01-03